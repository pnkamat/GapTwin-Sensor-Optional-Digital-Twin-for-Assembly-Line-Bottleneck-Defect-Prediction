# GapTwin — Digital Twin for Assembly Line Bottleneck & Defect Prediction

## The problem

On a vehicle assembly line, no station works in isolation. A slowdown or a defect at one station rarely stays local — it can ripple through the stations downstream. By the time the problem becomes obvious in output or quality numbers, the plant may already be dealing with rework, delayed vehicles, or several units carrying the same issue.

A big part of the problem is visibility. Real assembly lines are a mix of modern and legacy equipment: some stations are well instrumented, while others rely on manual checks or have no direct sensor data at all. A digital twin that assumes perfect sensor coverage does not match that reality.

**GapTwin is designed around the line a plant actually has — including the stations that are easy to measure and the ones that are not.**

## What it does

- Continuously watches **station health** (cycle time vs. expected/takt time) for every station on the line.
- For stations **without direct sensors**, infers their health from the timing behavior of neighboring sensored stations — a slow, unsensored station shows up as a buildup/starvation pattern around it, even with zero hardware installed there.
- Every alert carries a **confidence score**, so plant teams know whether they're looking at direct sensor data or an inference.
- Tracks a lightweight **risk profile per vehicle**, so units that passed through a degrading station can be flagged for spot-checking before final inspection — not just the station.
- Surfaces all of this on a single dashboard with two views: a **Floor Supervisor** view (real-time station status, what to check now) and a **Plant Manager** view (rollout roadmap, estimated impact) — because different stakeholders need different things from the same underlying model.

## What it deliberately does NOT do

We are keeping the scope deliberately clear:

- **It does not simulate the physical assembly process.** No robotics, no CAD, no physics engine. It's a monitoring + statistical inference + reasoning system that ingests line data (timing, in this prototype) and reasons about it.
- **It does not diagnose *why* a station is degrading** — only *that* it's deviating from normal, with what confidence. Root-cause investigation (tool wear vs. bad part vs. operator variation) is left to the plant team; the twin's job is to point them at the right place early, not replace their judgment.
- **It does not act on the line.** It observes and recommends. Every decision stays with the plant team — this is a deliberate design choice, not a limitation, reflecting the challenge's own framing of AI + human ingenuity working together.


## How it works — architecture

```
simulate_v2.py  →  assembly_line_data.csv  →  detect.py  →  station_summary.csv
                         (+ true_anomaly_station.json,           + vehicle_risk.csv
                            kept separate, validation-only)              ↓
                                                                  dashboard (HTML/JS)
```

### 1. Data — `simulate_v2.py`

Because real plant data is proprietary and unavailable for this challenge, we use a synthetic data generator to stand in for the factory's MES/sensor system. The Round 2 brief explicitly allows this approach.

- **40 stations**, split into three zones: Body construction (1–15), Paint (16–22), Final Assembly (23–40) — within the brief's reference range of 30–50 stations.
- **300 simulated vehicles** move through the line in sequence.
- **~70% of stations are "sensored"** (report `cycle_time_s` directly, plus `torque_reading` at a subset of quality-relevant stations); **~30% are "unsensored"** (these fields are left blank in the data — simulating a real coverage gap, not just a data quality issue).
- **The anomaly location, severity, and onset timing are randomized on every run** (a random unsensored station is chosen, drift starts somewhere between unit 100–180, ramps over 70–130 units, peaks between +20–45% of baseline cycle time). This keeps the demo honest: the detector cannot simply be tuned to one known answer. It has to find the station that is actually drifting, wherever the simulator places it.
- **Ground truth** (which station, when, how severe) is written to a **separate file**, `true_anomaly_station.json`. The detection engine never reads this file — it exists purely so we can check, after the fact, whether detection was correct. This mirrors a proper blind validation setup.
- **Defects** have a small, independent baseline rate (~3%) applied across every inspection checkpoint, representing ordinary quality variation unrelated to our injected anomaly, plus an elevated, correlated rate downstream of the anomaly station once it starts drifting. This avoids the dataset looking artificially clean/circular — defects exist for reasons other than the one thing we're demonstrating, same as a real line.

Run it with a fixed seed for a reproducible demo (`python3 simulate_v2.py --seed 42`), or without a seed for a fresh random scenario each time.

### 2. Detection engine — `detect.py`

**For sensored stations:** classic Statistical Process Control (SPC) — the same principle factories have used for quality control for decades, applied in code. A baseline (mean + standard deviation) is established from each station's first 30 units. Every subsequent unit is compared against that baseline using a rolling 20-unit window and a z-score. A station is only flagged if it stays more than 2 standard deviations above baseline for at least 5 consecutive units — this avoids a single noisy reading triggering a false alarm.

**For unsensored stations (the core mechanism):** the engine finds the nearest sensored station before and after the unsensored one, and measures the total elapsed time for a unit to cross that whole block. The same rolling z-score logic is applied to that block-level timing. If the block is taking longer than its own history says it should, something inside it is degrading — the engine never reads the station's true hidden value (`true_cycle_time_s`, present in the data only for our own later validation); it only ever sees what a real sensorless station would actually expose: the surrounding timing pattern.

**Confidence scoring** reflects how the flag was derived, not just whether one exists:

| Station type | Flagged | Confidence |
|---|---|---|
| Sensored | Yes | 0.90 |
| Sensored | No | 0.95 |
| Unsensored (inferred) | Yes | 0.55 |
| Unsensored (inferred) | No | 0.70 |
| Unsensored, no usable neighbor | — | 0.30 |

These are currently rule-based values rather than statistically calibrated confidence estimates. We state that explicitly rather than presenting them as more precise than the prototype supports. A natural next step (noted under Limitations) is calibrating these against measured precision/recall across many simulated runs.

**Predicted lead time** extrapolates the recent rate of change forward to estimate how many more units until a critical threshold (1.5× expected time) is crossed — simple linear projection, not a trained model. One limitation is that a very small early drift can produce an unrealistically large lead-time estimate; see the limitations below.

**Vehicle risk rollup:** each vehicle accumulates a risk score based on the confidence-weighted set of flagged stations it passed through, counted only from the point those stations were actually confirmed to be drifting (not before). The result is a ranked list of vehicles worth spot-checking instead of treating the entire batch as equally risky.

### 3. Dashboard — `dashboard/`

A single-file HTML/CSS/JS control-room-style interface with a dark industrial look, Chart.js for trend visualization, and no backend required. It shows:
- A line schematic — all 40 stations as connected nodes, colored by sensored/flagged status, with a pulsing indicator on any currently flagged station
- An explicit text alert naming the flagged station/block (not just implied through color)
- A trend chart comparing the flagged block's timing against a healthy sensored control station, for visual contrast
- Confidence and method (direct sensor vs. inference) shown for every alert
- Ranked recommended interventions
- A **Floor Supervisor / Plant Manager** toggle — same underlying data, two different views suited to two different roles, directly addressing the Round 2 brief's point that different stakeholders need different views of the same twin
- A vehicle risk table, showing which specific units are worth spot-checking and why


## Validation — does the inference actually work?

We tested the pipeline in a blind setup: `simulate_v2.py` picks a random unsensored anomaly station and hides it in `true_anomaly_station.json`; `detect.py` is run against only the visible data. In our locked demo run (seed 42), the true anomaly was at **station 8** (unsensored) — the detection engine correctly flagged the block containing it (stations 6–9) purely from neighbor timing inference, without ever being told where to look.

We also checked whether the resulting vehicle risk scores meant anything: units flagged as at-risk showed a substantially higher real (simulated) defect rate than unflagged units, both for defects specifically caused by the anomaly and for defects overall — confirming the risk score is a genuine signal, not noise.


## How to run it

```bash
# 1. Generate a dataset (reproducible demo run)
python3 engine/simulate_v2.py --seed 42

# 2. Run the detection engine against it
python3 engine/detect.py

# 3. Open the dashboard
open dashboard/index.html   # or just double-click it — no server needed
```

To test that detection generalizes to a different, hidden anomaly location, omit `--seed` in step 1 for a fresh random scenario, then compare `station_summary.csv`'s flagged stations against `true_anomaly_station.json` afterward.


## Known limitations

- **Predicted lead time can produce unrealistic values** when the detected drift rate is still very small — a sanity cap or an "insufficient rate to estimate yet" fallback is a planned near-term fix rather than a resolved feature.
- **Confidence scores are reasoned placeholders**, not calibrated against measured false-positive/false-negative rates across many runs. Calibrating this properly (e.g., running detection against 50+ randomized simulations and measuring actual precision per confidence tier) is the natural next step to make the confidence numbers statistically defensible rather than illustrative.
- **The engine identifies *that* a station is deviating, not *why*.** Root-cause diagnosis (tool wear, part quality, operator variation) is intentionally left to the plant team; the recommended interventions are historically-informed suggestions, not a diagnosed cause.
- **Data is simulated**, not from a real plant — explicitly permitted and expected for this round, but worth restating plainly. The simulator's assumptions (station count, sensor ratio, defect rates, drift shape) are documented above so they can be scrutinized and adjusted.
- **Dashboard currently runs against one dataset at a time**, loaded from a fixed CSV. Allowing a user to upload their own dataset and exclude non-applicable stations is a planned extension to demonstrate generalization across different plant layouts.


## Why this matters / where it goes next

This prototype demonstrates the core mechanism at a scope buildable in a hackathon timeframe. The same approach — infer from what you can observe, be explicit about confidence, keep humans in control of the decision — is designed to extend beyond a single simulated line: to other lines within the same plant, to plants with different equipment vintages and sensor maturity, and to other domains that share the same structure (a sequence of interdependent process steps with uneven instrumentation). See the accompanying Business Proposal document for target users, business case, phased rollout plan, and risks.
