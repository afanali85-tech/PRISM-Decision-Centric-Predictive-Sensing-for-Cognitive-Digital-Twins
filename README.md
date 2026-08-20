# CDT System-Level Simulation

A system-level simulation for evaluating **Digital Twin (DT) sensing and knowledge-update strategies** in a dense urban 6G environment.

The simulation models a mobile network as a grid of sensing/knowledge cells in which mobile nodes move using a random-waypoint mobility model. Environmental changes are represented by random **unknown events** that alter the true state of individual cells. These changes can make the Digital Twin's stored knowledge stale until the affected cells are sensed again.

The notebook compares four baseline Digital Twin update strategies against a proposed **Cognitive Digital Twin (CDT)** approach based on decision-centric predictive perception.

## Overview

The simulated environment consists of a (15 \times 15) grid covering a (1500 \times 1500) m area. A total of 100 mobile nodes move through the environment.

Each grid cell maintains:

* A **true state**, representing the actual environment.
* An **estimated state**, representing the Digital Twin's current knowledge.
* A **staleness value**, representing how long it has been since the cell was sensed.

At every simulation step, cells may experience random state changes. Different sensing policies determine which cells should be updated in the Digital Twin.

The objective is to study the trade-off between:

* Knowledge accuracy
* Sensing overhead
* Decision success
* Decision latency
* Mobility
* Environmental dynamics

## Compared Policies

Five sensing and knowledge-update policies are implemented.

| Policy Key | Strategy                                | Description                                                                                                                       |
| ---------- | --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `periodic` | Periodic Full-Sync DT                   | Periodically synchronizes the complete Digital Twin with the physical environment.                                                |
| `predict`  | Fixed-Radius Predictive DT              | Predicts cells along node headings and senses them subject to a sensing budget.                                                   |
| `voi`      | Goal-Oriented / Value-of-Information DT | Prioritizes cells according to their staleness.                                                                                   |
| `reactive` | Closed-Loop Reactive DT                 | Senses cells when node transitions occur, without anticipating future decisions.                                                  |
| `cdt`      | Proposed Cognitive DT / PRISM           | Uses decision-centric predictive perception to anticipate relevant cells and uses remaining sensing capacity for uncertain cells. |

The proposed CDT policy attempts to achieve better decision performance by sensing **where knowledge is expected to matter for upcoming decisions**, rather than continuously maintaining a globally synchronized Digital Twin.

## Proposed CDT Strategy

The `cdt` policy combines two sensing mechanisms.

First, it predicts cells that nodes are about to enter during the next movement step. These cells correspond to imminent decision locations and are given sensing priority.

The remaining sensing budget is then assigned to the cells with the greatest staleness.

Conceptually:

```text
1. Predict next-step node locations.
2. Identify nodes that are about to cross into another cell.
3. Prioritize those destination cells for sensing.
4. Use the remaining sensing budget on the stalest cells.
5. Update the Digital Twin knowledge.
6. Make mobility-related decisions using the updated knowledge.
```

If the CDT fails to anticipate a transition and encounters stale knowledge, it can perform an on-demand re-sensing operation. This recovers the correct state but introduces one additional step of latency.

## Simulation Parameters

The notebook uses the following default configuration:

| Parameter           | Default | Description                                       |
| ------------------- | ------: | ------------------------------------------------- |
| `GRID`              |      15 | Number of grid cells along each dimension         |
| Total cells         |     225 | (15 \times 15) sensing/knowledge cells            |
| `N_NODES`           |     100 | Number of mobile nodes                            |
| `AREA`              |  1500 m | Width/height of the simulated area                |
| `CELL_SIZE`         |   100 m | Size of each grid cell                            |
| `B_BUDGET`          |      30 | Default sensing budget per step                   |
| `T_SYNC`            |       8 | Periodic full-sync interval                       |
| `LOOKAHEAD`         |       2 | Look-ahead distance used by the predictive policy |
| Default random seed |       7 | Initial NumPy random generator seed               |

Individual experiments use additional random seeds to average results across multiple simulation runs.

## Mobility Model

Nodes follow a **random-waypoint mobility model**.

Each node:

1. Starts at a random position in the simulated area.
2. Receives a randomly selected waypoint.
3. Moves toward that waypoint according to the specified speed.
4. Receives a new random waypoint after reaching the current destination.

The simulation converts continuous node positions into grid-cell indices to determine which part of the Digital Twin is relevant to each node.

## Environmental Dynamics

Environmental changes are modeled using random **unknown events**.

For every cell and every simulation step, a state change occurs with probability:

```python
event_rate
```

When an event occurs, the binary true state of the corresponding cell is flipped.

The Digital Twin does not automatically know about this change. Therefore:

```text
Physical state changes
        ↓
Digital Twin becomes stale
        ↓
Sensing/update operation
        ↓
Digital Twin becomes synchronized again
```

This mechanism represents changes such as new obstacles, scatterers, or other environmental conditions that may invalidate previously stored knowledge.

## Metrics

The simulation records several performance metrics.

### 1. Global DT Synchronization Error

The fraction of all grid cells for which the Digital Twin state differs from the true state:

```text
Synchronization Error =
    Incorrect DT Cells / Total Cells
```

Returned as:

```python
err_hist
avg_error
```

### 2. Decision-Relevant Knowledge Error

Instead of considering the entire grid, this metric measures knowledge errors specifically at the cells currently occupied by mobile nodes.

Returned as:

```python
node_err_hist
avg_node_error
```

This provides a more decision-relevant measure of Digital Twin accuracy because most grid cells may not affect any active node at a particular time.

### 3. Sensing Overhead

The number of cells sensed during each simulation step.

Returned as:

```python
overhead_hist
avg_overhead
```

The metric allows comparison of the sensing cost required by different policies.

### 4. Decision Success Rate

A decision occurs when a node transitions from one grid cell to another.

For the periodic, predictive, and VoI policies, a decision succeeds when the Digital Twin already contains the correct state for the destination cell.

The reactive policy performs an on-demand sensing operation before every transition decision, so its decisions are corrected at the expense of latency.

The CDT can also perform a fallback re-sensing operation when an unanticipated transition encounters stale knowledge.

Returned as:

```python
decision_success_rate
```

### 5. Decision Latency

Decision latency represents additional simulation steps caused by reactive sensing.

The reactive policy incurs a one-step sense-then-decide delay for transitions.

The CDT incurs the same additional step only when its predictive sensing mechanism misses a required cell and a fallback re-sensing operation is necessary.

Returned as:

```python
avg_latency
```

## Main Simulation Function

The core simulation is implemented in:

```python
run_policy(
    policy,
    speed,
    event_rate,
    T=200,
    budget=B_BUDGET,
    tsync=T_SYNC,
    rng=rng
)
```

### Arguments

| Argument     | Description                                                            |
| ------------ | ---------------------------------------------------------------------- |
| `policy`     | Policy to simulate: `periodic`, `predict`, `voi`, `reactive`, or `cdt` |
| `speed`      | Mobile-node speed                                                      |
| `event_rate` | Probability of an unknown event occurring in a cell during each step   |
| `T`          | Number of simulation steps                                             |
| `budget`     | Maximum sensing budget for budgeted policies                           |
| `tsync`      | Synchronization interval for the periodic policy                       |
| `rng`        | NumPy random-number generator                                          |

### Returned Results

The function returns:

```python
{
    "err_hist": ...,
    "node_err_hist": ...,
    "overhead_hist": ...,
    "avg_overhead": ...,
    "avg_error": ...,
    "avg_node_error": ...,
    "decision_success_rate": ...,
    "avg_latency": ...
}
```

## Experiments

The notebook performs three main experiments.

### Figure 1 — Decision-Relevant Knowledge Error vs. Time

This experiment evaluates how accurately each policy maintains knowledge at node locations over time.

Default operating point:

```python
speed_mid = 8.0
event_mid = 0.01
N_SEEDS = 6
```

Each policy is simulated for 200 steps over six random seeds.

The mean knowledge-error curve is calculated and smoothed using a five-step moving average.

The notebook produces:

```text
fig_knowledge_error.pdf
fig_knowledge_error_full.pdf
fig_knowledge_error_zoomed.pdf
```

The full-scale figure compares all five policies, while the zoomed figure excludes the fixed-radius predictive policy to make the closely clustered policies easier to inspect.

### Figure 2 — Minimum Sensing Overhead for 95% Decision Success

The second experiment evaluates the sensing overhead required to maintain a target decision success rate of:

```text
95%
```

Environmental dynamics are varied using:

```python
event_rates = np.linspace(0.002, 0.03, 8)
```

For each event rate, the simulation searches candidate sensing budgets or synchronization intervals until the policy reaches the target success rate.

Candidate periodic synchronization intervals:

```python
TSYNC_CANDIDATES = [
    40, 30, 20, 15, 10, 8, 6, 4, 3, 2, 1
]
```

Candidate sensing budgets:

```python
BUDGET_CANDIDATES = [
    5, 10, 15, 20, 25, 30,
    40, 50, 70, 100, 150, 200, 225
]
```

The resulting figure is saved as:

```text
fig_overhead_dynamics.pdf
```

### Figure 3 — Decision Success Rate vs. Mobility

The third experiment studies the effect of node mobility.

Node speed varies between:

```text
2 m/s and 20 m/s
```

using:

```python
speeds = np.linspace(2.0, 20.0, 8)
```

Each policy is evaluated across four random seeds, and the mean decision success rate is reported.

The figure is saved as:

```text
fig_success_mobility.pdf
```

## Summary Experiment

The final notebook cell evaluates all policies at the moderate operating point:

```python
speed = 8.0
event_rate = 0.01
T = 300
```

Results are averaged across six random seeds.

The output table reports:

```text
Policy
NodeErr
Overhead
Success%
Latency
```

The raw data used for the sensing-overhead experiment is also printed.

## Requirements

The simulation requires Python and the following packages:

```text
numpy
matplotlib
```

When running in Google Colab, the notebook installs these packages automatically using:

```bash
pip install -q numpy matplotlib
```

The notebook also uses:

```python
IPython.display
```

for notebook display functionality.

## Running the Simulation

### Google Colab

1. Upload the notebook to Google Colab.
2. Open the notebook.
3. Select **Runtime → Run all**.
4. Wait for all simulation experiments to complete.
5. The generated PDF figures will be available in the Colab working directory.

### Local Jupyter Environment

Install the required dependencies:

```bash
pip install numpy matplotlib jupyter
```

Start Jupyter:

```bash
jupyter notebook
```

Open the simulation notebook and run all cells from top to bottom.

## Generated Files

After completing all experiments, the notebook generates the following PDF figures:

```text
fig_knowledge_error.pdf
fig_knowledge_error_full.pdf
fig_knowledge_error_zoomed.pdf
fig_overhead_dynamics.pdf
fig_success_mobility.pdf
```

It also prints a numerical summary of the policies and the raw overhead-versus-event-rate data used in Figure 2.

## Reproducibility

The simulation uses explicitly initialized NumPy random-number generators.

For example:

```python
rng = np.random.default_rng(7)
```

Individual experiments also create deterministic generators using fixed seed ranges such as:

```python
100 + s
200 + s
300 + s
400 + s
```

This makes experiments reproducible when the same code, parameters, and software environment are used.

## Project Structure

A minimal repository structure can be organized as:

```text
project/
│
├── README.md
├── cdt_sim.ipynb
│
└── results/
    ├── fig_knowledge_error.pdf
    ├── fig_knowledge_error_full.pdf
    ├── fig_knowledge_error_zoomed.pdf
    ├── fig_overhead_dynamics.pdf
    └── fig_success_mobility.pdf
```

> Note: the current notebook saves figures directly to the working directory. Moving them into `results/` would require changing the paths used in `plt.savefig()`.

## Interpretation

The simulation is designed to investigate whether maintaining a perfectly synchronized Digital Twin everywhere is necessary for successful network decisions.

The proposed CDT/PRISM approach instead emphasizes **decision-relevant knowledge**. It predicts which environmental cells are likely to become important for imminent node transitions and prioritizes sensing those locations while allocating unused sensing capacity to stale cells.

This enables the simulation to compare globally oriented synchronization strategies with a decision-centric sensing strategy in terms of both **decision performance and sensing cost**.

## Notes

* Cell states are represented as binary values.
* Unknown environmental events are modeled as random state flips.
* Mobility follows a random-waypoint model.
* Results are averaged across multiple deterministic random seeds.
* The sensing budget is measured in cells sensed per simulation step.
* The simulation is system-level and abstracts away detailed physical-layer sensing and communication procedures.


