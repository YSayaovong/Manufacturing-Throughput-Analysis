# Multi-Machine Uptime & Bottleneck Simulator: Monte Carlo Production Line Analysis → Throughput Risk → Bottleneck Detection → Downtime Quantification

## 1. Project Background

As a reliability and industrial systems analyst building production-grade simulation tooling from scratch, I designed this engine to answer a question that spreadsheet-based capacity planning cannot reliably answer:

*When machines fail unpredictably across a multi-station production line, which station is most likely to become the bottleneck — and what does that cost in throughput?*

Traditional uptime analysis models each station in isolation: calculate availability, multiply by cycle time, report a theoretical throughput number. The problem is that production lines are serial systems. A single underperforming station starves every downstream station regardless of their individual availability. The interaction effects between stations, failure distributions, and parallel machine configurations cannot be captured by per-station averages — they require simulation across thousands of scenarios.

This tool applies **Monte Carlo simulation** to model those interactions explicitly. Each run samples failure and repair events from exponential distributions calibrated to real MTBF and MTTR parameters, computes station-level throughput under that failure scenario, identifies the constraining station, and records the result. After thousands of runs, the simulator surfaces the probability distribution of outcomes — not a single deterministic number — revealing which stations carry the most throughput risk and by how much.

The simulator is structured around four analytical outputs:

- **Throughput risk quantification** — the full distribution of line output across simulated scenarios, not just the mean
- **Bottleneck probability by station** — which station most frequently constrains the line and with what confidence
- **Downtime loss analysis** — average machine-hours lost per station, isolating where maintenance investment has the greatest return
- **Sensitivity to parallel machine configuration** — how adding machines at the bottleneck station shifts the throughput distribution

---

## 2. Simulation Architecture

The simulator models a serial production line where total line throughput is constrained by the lowest-output station in any given scenario — the manufacturing equivalent of the theory of constraints applied probabilistically.

```
Station Configuration (cycle time, MTBF, MTTR, machines)
        │
        ▼
┌─────────────────────┐
│  Failure Sampling   │  ← exponential distribution on MTBF per machine
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Repair Sampling    │  ← exponential distribution on MTTR per machine
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Station Output     │  ← uptime × machines ÷ cycle time per station
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Line Throughput    │  ← min(station outputs) — serial constraint
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Bottleneck Record  │  ← argmin(station outputs) recorded per run
└─────────────────────┘
        │  [× N Monte Carlo runs]
        ▼
┌─────────────────────┐
│  Aggregate Results  │  → throughput distribution, bottleneck probability,
└─────────────────────┘     downtime by station
```

| Stage | Input | Output | Distribution |
|---|---|---|---|
| Failure sampling | MTBF per machine | Uptime per machine | Exponential |
| Repair sampling | MTTR per machine | Downtime per machine | Exponential |
| Station output | Uptime, machines, cycle time | Units produced per station | Derived |
| Line throughput | All station outputs | Constrained line output | min() |
| Bottleneck detection | All station outputs | Constraining station ID | argmin() |

The use of exponential distributions for both failure and repair intervals is standard in reliability engineering: exponential inter-arrival times correspond to a memoryless (Poisson) failure process, which is the baseline assumption for mechanical failure modeling in the absence of wear-curve data. MTBF and MTTR parameters can be calibrated directly from historical maintenance logs.

---

## 3. Station Model Parameters

Each station in the production line is defined by four parameters:

| Parameter | Symbol | Description | Unit |
|---|---|---|---|
| Mean cycle time | CT | Time required to process one unit per machine | seconds / unit |
| Mean Time Between Failures | MTBF | Expected operating time between failure events | hours |
| Mean Time To Repair | MTTR | Expected time to restore a failed machine | hours |
| Parallel machines | M | Number of machines operating in parallel at the station | count |

**Derived metrics computed per station per run:**

| Metric | Formula | Interpretation |
|---|---|---|
| Machine availability | `MTBF / (MTBF + MTTR)` | Fraction of time a single machine is operational |
| Station uptime (sampled) | `sum(exponential(MTBF))` across repair cycles | Actual operating hours in the simulation window |
| Station output | `(uptime × M) / cycle_time` | Units produced at this station in the scenario |
| Downtime loss | `total_time − uptime` | Machine-hours lost to failure and repair |

Because failure and repair events are sampled independently for each machine in each run, the station-level output in any given scenario reflects the specific overlap of individual machine failure windows — an interaction that per-station availability averages cannot capture.

---

## 4. Outputs

### Line Throughput Distribution

![Throughput Distribution](assets/line_throughput_distribution.png)

The throughput distribution shows the range of production outcomes across all simulated scenarios. The width of this distribution quantifies throughput risk: a narrow distribution indicates a stable, predictable line; a wide distribution indicates high scenario-to-scenario variability driven by failure clustering. The left tail of the distribution represents worst-case outcomes and is the relevant input for production scheduling buffers and safety stock calculations.

### Bottleneck Probability by Station

![Bottleneck Probability](assets/bottleneck_probability.png)

Bottleneck probability is the fraction of Monte Carlo runs in which each station produced the minimum output — i.e., was the constraining station for the line. A station with 60% bottleneck probability is the production constraint in more than half of all simulated failure scenarios. This metric directly prioritizes where maintenance investment, redundant capacity, or cycle time improvement will have the greatest impact on line throughput.

### Average Downtime by Station

![Station Downtime](assets/station_downtime.png)

Average downtime quantifies machine-hours lost per station across all runs. High downtime at a non-bottleneck station may have low impact on throughput if parallel machines absorb the loss. High downtime at the primary bottleneck station compounds throughput loss directly. Reading this chart alongside bottleneck probability identifies where downtime reduction produces throughput gain versus where it produces only cost avoidance.

---

## 5. Engineering Use Cases

| Use Case | How the Simulator Addresses It |
|---|---|
| Throughput risk forecasting | Full throughput distribution quantifies P10/P50/P90 production outcomes for scheduling |
| Preventive maintenance planning | Bottleneck probability and downtime by station prioritize maintenance ROI by station |
| Bottleneck identification & line balancing | Bottleneck probability distribution replaces deterministic bottleneck guesses |
| Downtime loss analysis | Machine-hours lost per station quantifies cost of current failure rates |
| Capacity planning | Re-running with modified `M` (parallel machines) quantifies throughput gain from capital investment |
| MTBF/MTTR sensitivity testing | Adjusting parameters isolates which reliability improvements most shift the throughput distribution |

---

## 6. Running the Simulator

### Install dependencies

```bash
pip install numpy matplotlib
```

### Run the simulation

```bash
python simulator.py
```

Output charts are saved automatically to the `assets/` directory. The simulator is safe to re-run; output files are overwritten on each execution.

### Modify station configuration

Station parameters are defined directly in `simulator.py` as a list of station dictionaries. To model a different production line, update the `stations` list with the appropriate cycle time, MTBF, MTTR, and machine count values for each station:

```python
stations = [
    {"name": "Station A", "cycle_time": 30, "mtbf": 200, "mttr": 4, "machines": 2},
    {"name": "Station B", "cycle_time": 45, "mtbf": 150, "mttr": 6, "machines": 1},
    {"name": "Station C", "cycle_time": 25, "mtbf": 300, "mttr": 3, "machines": 3},
]
```

---

## 7. Potential Enhancements

| Enhancement | Value |
|---|---|
| Weibull failure distributions | Model wear-based failure patterns where failure probability increases with machine age — more accurate for mechanical components with known wear curves |
| Buffer / WIP inventory between stations | Decouple serial stations with intermediate queues; quantify how buffer size reduces throughput sensitivity to upstream failures |
| Shift schedule modeling | Apply operating window constraints (8-hour shifts, planned downtime) to align simulation with real production calendars |
| Cost layer | Attach revenue-per-unit and downtime-cost-per-hour to convert throughput distributions directly into financial risk ranges |
| Parameter sweep automation | Loop over MTBF, MTTR, and machine count ranges to generate sensitivity surfaces for capital investment decisions |
| Interactive dashboard | Replace static Matplotlib exports with a Streamlit or Dash interface for real-time parameter adjustment and scenario comparison |
| CSV parameter input | Load station configurations from a structured CSV to enable non-engineering stakeholders to define scenarios without modifying code |

---

## 8. Summary

This simulator applies Monte Carlo methods to quantify production throughput risk, bottleneck probability, and downtime losses across a configurable multi-station manufacturing line — outputs that deterministic capacity planning models cannot produce. The serial constraint model reflects the theory of constraints as a probabilistic system: the bottleneck is not a fixed station but a distribution of outcomes across all stations, shaped by their individual reliability parameters and parallel machine configurations. The three output charts together provide a complete reliability engineering brief: where the line constrains, how often, and what it costs in machine-hours. MTBF and MTTR parameters are directly calibrated from maintenance records, making the simulator applicable to real production environments without modification to the underlying methodology.

---

## Tools Used

- Python 3 — simulation engine, output generation
- NumPy — exponential distribution sampling, vectorized station output computation
- Matplotlib — throughput distribution, bottleneck probability, and downtime charts
- Git / GitHub — version control
