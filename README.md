# Simulating the Team-Level Cost of Boredom in Manual Order Picking

A runnable hybrid simulation prototype for estimating the operational and monetary effects of employee boredom in manual picker-to-parts order picking.

The model combines:

- **Discrete-event simulation** with SimPy for order arrivals, queueing, picking, errors, rework, and shift execution.
- **Agent-based behavioural modelling** with plain Python classes for boredom development, performance degradation, conversations, peer interruption, and emotional contagion.
- **Monte Carlo experiments** with common random numbers for controlled comparisons across scenarios.

The prototype was developed for the master's thesis topic:

> **Simulating the Team-Level Cost of Boredom in Manual Order Picking**

## Research Questions

The model is designed to support two main research questions:

1. **RQ1:** How large is the team-level productivity loss caused by a given share of bored employees?
2. **RQ2:** What share of the total loss is indirect, meaning caused by interactions, interruptions, and contagion among colleagues, rather than by the bored employees' own reduced performance?

## Important Disclaimer

This is a **research prototype**, not a validated decision-support system.

All behavioural parameters, including boredom build-up, boredom decay, slowdown, error sensitivity, conversation frequency, conversation duration, and contagion strength, are explicitly treated as:

> **PLACEHOLDER, not empirically calibrated**

The default results demonstrate the behaviour of the model. They must not be interpreted as empirical estimates of the cost of boredom in a real warehouse until the model has been calibrated and validated with operational and behavioural data.

## Main Features

- Configurable eight-hour shift
- Configurable team size, default: 8 pickers
- One shared FIFO order queue
- Non-homogeneous Poisson order arrivals
- Hourly demand profile with peak and low-demand periods
- Random order sizes between 1 and 8 lines
- Lognormal travel and picking times
- Line-level picking errors and rework
- Continuous boredom state `B` in the interval `[0, 1]`
- One-minute boredom updates
- Boredom build-up under underload
- Boredom decay during periods of higher utilisation
- Optional monotony and intervention extensions
- Random-mixing or neighbour-based social contacts
- Boredom contagion through peer effects
- Boredom-dependent processing times
- Boredom-dependent error probabilities
- Boredom-dependent off-task conversations
- Immediate interruption of idle partners
- Delayed invitations for busy partners at line boundaries
- Direct and indirect loss decomposition
- Capacity-cost and paid-time-cost views
- Monte Carlo confidence intervals
- Parameter sweeps
- One-at-a-time sensitivity analysis
- Automatic sanity checks
- CSV result exports
- PNG visualisations

## Repository Structure

The prototype is intentionally implemented as a single Python file:

```text
boredom_sim.py
```

After execution, the program creates an output directory containing CSV tables and PNG plots.

## Requirements

- Python 3.10 or newer
- SimPy
- NumPy
- pandas
- Matplotlib

No Mesa installation is required. The agent layer is implemented using plain Python classes so that it can later be ported to Mesa if required.

## Installation

### 1. Create a virtual environment

```bash
python -m venv .venv
```

Activate it on Linux or macOS:

```bash
source .venv/bin/activate
```

Activate it in Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

### 2. Install the dependencies

```bash
python -m pip install simpy numpy pandas matplotlib
```

## Running the Simulation

### Quick mode

Use quick mode for installation checks, development, and preliminary analysis:

```bash
python boredom_sim.py --quick
```

Quick mode uses 10 replications per scenario or experimental setting and is intended to finish in less than approximately 30 seconds on a typical laptop, although runtime depends on hardware and Python environment.

### Full mode

Run the default experiment set with 100 replications per scenario or setting:

```bash
python boredom_sim.py
```

The target runtime is less than approximately three minutes on a typical laptop. Runtime may vary with processor performance and model modifications.

### Custom output directory

```bash
python boredom_sim.py --quick --output-dir thesis_results
```

### Custom random seed

```bash
python boredom_sim.py --quick --seed 20261105
```

### Command-line help

```bash
python boredom_sim.py --help
```

## Model Architecture

### 1. Discrete-Event Layer

The discrete-event layer is implemented with SimPy.

Orders arrive according to a piecewise-constant, non-homogeneous Poisson process. Each shift hour has its own order-arrival rate.

Each order contains:

- Arrival time
- Random number of order lines
- Pre-generated travel times
- Pre-generated picking times
- Pre-generated error random numbers
- Pre-generated rework durations
- Start time
- Completion time
- Number of errors

Each picker repeatedly:

1. Checks for a pending conversation invitation.
2. May initiate a conversation at an eligible decision point.
3. Requests the next order from the shared FIFO queue.
4. Processes the order line by line.
5. Applies a boredom-dependent time multiplier.
6. Draws line-level picking errors.
7. Performs rework when an error occurs.
8. Performs the fixed handover or packing activity.
9. Records the completed order.

### 2. Agent-Based Boredom Layer

Each picker is represented as a plain Python `Picker` object.

The picker has an individual continuous boredom state:

```text
0 <= B <= 1
```

Boredom is updated every minute using an Euler step. The implemented conceptual form is:

```text
dB = a_up * U * (1 - B)
     - a_down * (1 - U) * B
     + c * max(0, mean_B_contacts - B) * (1 - B)
```

where:

- `B` is the current boredom level.
- `U` is the underload signal.
- `a_up` is the boredom build-up coefficient.
- `a_down` is the boredom decay coefficient.
- `c` is the contagion coefficient.
- `mean_B_contacts` is the mean boredom level among the selected contacts.

The underload signal is calculated as:

```text
underload = 1 - rolling utilisation
```

Rolling utilisation is estimated over approximately the previous 15 minutes.

The state is clipped after each update:

```text
B = clip(B, 0, 1)
```

### 3. Behavioural Effects

Boredom affects four mechanisms.

#### Processing speed

Travel and picking times are multiplied by:

```text
1 + slowdown_strength * B
```

#### Error probability

The error probability per order line is:

```text
base_error_probability + error_boredom_slope * B
```

The value is capped below 1.

#### Conversation initiation

The hourly conversation initiation rate is:

```text
conversation_lambda0 + conversation_lambda1 * B
```

Conversation opportunities are evaluated at eligible decision points rather than continuously during an order line.

#### Conversation duration

Conversation duration follows a lognormal distribution. Its mean increases with the initiator's boredom level.

## Conversation and Interruption Logic

A conversation involves an initiator and, in the full scenario, a partner.

### Idle partner

An idle partner can accept immediately. If the partner is waiting for an order, the waiting process is interrupted safely. The outstanding queue request is cancelled before the picker enters the conversation.

### Busy partner

A busy partner cannot be interrupted halfway through an order line. Instead, the model stores a pending invitation. The invitation is handled at the next line boundary.

This design avoids:

- Losing an order during an interrupt
- Processing the same order twice
- Leaving an obsolete queue request active
- Deadlocks between workers
- Mid-line pre-emption inconsistencies

## Scenarios

The model runs three principal scenarios using the same random streams.

### Baseline

- Boredom is forced to zero.
- No boredom-related slowdown occurs.
- No boredom-related error increase occurs.
- No conversations occur.
- No partner interruptions occur.
- No contagion occurs.

### Direct Only

- Boredom affects the worker's own travel and picking speed.
- Boredom affects the worker's own error probability.
- A bored worker may initiate a conversation and lose their own time.
- The conversation does not interrupt a partner.
- Contagion is disabled.

### Full

- All own-performance effects are active.
- Conversation partners also lose time.
- Idle partners may be interrupted immediately.
- Busy partners may accept at the next line boundary.
- Boredom contagion is active.

## Common Random Numbers

For each replication, NumPy `SeedSequence.spawn()` creates independent random-number streams for:

1. Order arrivals
2. Order content
3. Travel and picking times
4. Errors and rework
5. Conversations and social contacts

The scenarios in a replication use the same seed and therefore share equivalent exogenous random inputs.

This common-random-number design reduces Monte Carlo noise when calculating differences between scenarios.

## Loss Decomposition

The primary decomposition is:

```text
direct loss   = cost(direct_only) - cost(baseline)
indirect loss = cost(full) - cost(direct_only)
total loss    = cost(full) - cost(baseline)
```

The spillover ratio is:

```text
spillover ratio = indirect loss / total loss
```

If total loss is effectively zero, the spillover ratio is reported as missing rather than dividing by a near-zero value.

### Interpretation

- **Direct loss** represents the cost associated with bored workers' own slowdown, elevated errors, and own conversation time.
- **Indirect loss** represents the additional cost after partner interruption and contagion are enabled.
- **Total loss** represents the difference between the full behavioural model and the no-boredom baseline.

### Path Dependence

The decomposition depends on the order in which mechanisms are activated.

In the implemented ordering, own effects are activated first and social mechanisms second. Interaction effects between own performance and social mechanisms are therefore assigned to the indirect component.

A thesis-level extension should calculate a second ordering, such as contagion first, or use a Shapley-value decomposition across mechanisms.

## Cost Definitions

### Capacity View

The capacity view is the primary cost measure:

```text
capacity cost =
    errors * error cost
    + total completed-order flow time * delay cost per order-minute
    + unfinished orders * unfinished-order penalty
```

This view captures quality losses, congestion, delayed completion, and end-of-shift backlog.

### Paid-Time View

The paid-time view is:

```text
paid-time cost =
    wage rate * (conversation minutes + slowdown minutes) / 60
```

This view resembles a conventional labour-cost calculation but does not fully capture queueing and service-level effects.

## Default Demand Profile

The default hourly demand profile is:

```text
22, 34, 50, 64, 60, 36, 26, 20 orders/hour
```

Using the default expected order content and process-time assumptions, the analytical utilisation estimates are approximately:

| Shift hour | Orders/hour | Expected utilisation |
|---:|---:|---:|
| 1 | 22 | 30.3% |
| 2 | 34 | 46.8% |
| 3 | 50 | 68.8% |
| 4 | 64 | 88.1% |
| 5 | 60 | 82.5% |
| 6 | 36 | 49.5% |
| 7 | 26 | 35.8% |
| 8 | 20 | 27.5% |

The profile therefore includes both low-demand periods and a peak close to the target utilisation range of 85% to 90%.

## Experiments

### Main Monte Carlo Comparison

The baseline, direct-only, and full scenarios are run for:

- 10 replications in quick mode
- 100 replications in full mode

The program reports means and normal-approximation 95% Monte Carlo confidence intervals.

### Sweep 1: Share of Initially Bored Employees

```text
0%, 12.5%, 25%, 50%, 75%
```

Main outputs:

- Total loss
- Spillover ratio

This sweep directly supports RQ1 and RQ2.

### Sweep 2: Demand Level

```text
0.6, 0.8, 1.0, 1.2
```

These values scale the complete hourly demand profile.

The experiment evaluates whether indirect effects become more important when colleagues are busier and interruptions have stronger queueing consequences.

### Sweep 3: Team Size

```text
4, 8, 12, 16 pickers
```

This experiment explores whether social spillovers and team-level aggregation change with team size.

### One-at-a-Time Sensitivity Analysis

The following uncertain behavioural parameters are tested at 50% and 150% of their default values:

- Slowdown strength
- Error sensitivity to boredom
- Boredom-dependent conversation rate
- Contagion strength
- Boredom build-up rate
- Mean conversation duration

The sensitivity outputs are:

- Total capacity loss
- Spillover ratio

### Optional Intervention

The configuration includes a simple threshold-based intervention:

- If boredom exceeds a configurable threshold, boredom is reduced by a configurable fraction.

This is a placeholder for mechanisms such as:

- Job rotation
- Task enrichment
- A micro-break
- Temporary reassignment
- Cross-training activity

The intervention is disabled by default.

## Output Metrics

Each scenario run records:

- Orders arrived
- Orders completed
- Orders unfinished
- Order lines completed
- Team picks per hour
- Picks per picker-hour
- Number of errors
- Error rate
- Idle minutes
- Conversation minutes
- Slowdown minutes
- Mean order-flow time
- Mean boredom over the shift
- Capacity cost
- Paid-time cost
- Order-conservation status
- Boredom-bound status
- Conversation-accounting status

## Output Files

The default output directory is:

```text
boredom_outputs/
```

The program writes the following files:

```text
boredom_outputs/
├── expected_utilisation.csv
├── main_runs.csv
├── loss_decomposition_runs.csv
├── scenario_summary.csv
├── loss_summary.csv
├── sweep_share_bored_runs.csv
├── sweep_share_bored_summary.csv
├── sweep_demand_scale_runs.csv
├── sweep_demand_scale_summary.csv
├── sweep_team_size_runs.csv
├── sweep_team_size_summary.csv
├── sensitivity_summary.csv
├── 01_mean_boredom.png
├── 02_direct_indirect_loss.png
├── 03_spillover_share_bored.png
├── 04_spillover_demand_scale.png
└── 05_tornado_total_loss.png
```

## Plot Interpretation

### `01_mean_boredom.png`

Shows mean team boredom over the shift for the baseline, direct-only, and full scenarios.

Use it to inspect:

- Whether low-demand periods raise boredom
- Whether boredom decays during busy periods
- Whether contagion raises team-average boredom

### `02_direct_indirect_loss.png`

Shows direct and indirect capacity loss as a stacked bar in euros per shift.

This is the main visual summary for RQ2.

### `03_spillover_share_bored.png`

Shows the estimated spillover ratio against the initially bored share, including 95% confidence intervals.

This connects the population prevalence of boredom to the importance of team interactions.

### `04_spillover_demand_scale.png`

Shows the spillover ratio at different demand levels.

This tests whether partner interruption becomes more costly when the operation is highly utilised.

### `05_tornado_total_loss.png`

Shows the effect of increasing or decreasing uncertain behavioural parameters by 50%.

This identifies which placeholder assumptions most strongly affect the total-loss estimate.

## Built-In Sanity Checks

The following checks run automatically before the main experiments:

1. **Effects-off equality**  
   With all boredom coefficients and conversation rates disabled, `baseline`, `direct_only`, and `full` must produce equivalent key results.

2. **Boredom bounds**  
   Every recorded boredom state must remain in `[0, 1]`.

3. **Order conservation**  
   Every arrived order must be either completed or counted as unfinished.

4. **Conversation accounting**  
   Partner conversation time must be represented in total participant time.

5. **Reproducibility**  
   Running the same scenario with the same seed must return identical metrics.

A failed check raises an assertion and stops the experiment.

## Mapping Outputs to the Research Questions

### RQ1: Team-Level Productivity Loss

Use:

- `total_loss` in `loss_summary.csv`
- `sweep_share_bored_summary.csv`
- Team picks per hour
- Picks per picker-hour
- Error rate
- Mean order-flow time
- Unfinished orders

The share-bored sweep estimates how total operational loss changes as the proportion of initially bored workers increases.

### RQ2: Indirect Share of Total Loss

Use:

- `indirect_loss`
- `spillover_ratio`
- `02_direct_indirect_loss.png`
- `03_spillover_share_bored.png`
- `04_spillover_demand_scale.png`

The spillover ratio expresses the share of total loss attributed to the additional activation of partner interruption and contagion.

## Interpreting Example Results

Results generated with the default configuration are illustrative only.

A positive total loss means the full model is more costly than the no-boredom baseline. A positive indirect loss means partner interruption and contagion increase costs beyond the worker's own behavioural effects.

A negative estimate may occur in a limited Monte Carlo sample because of stochastic queueing effects. It should not automatically be interpreted as evidence that boredom is beneficial. Increase the number of replications and inspect confidence intervals and replication-level distributions.

## Configuration

All parameters are stored in the `Config` dataclass near the beginning of `boredom_sim.py`.

Example:

```python
@dataclass(frozen=True)
class Config:
    shift_hours: float = 8.0
    team_size: int = 8
    share_bored: float = 0.25
    b0_bored: float = 0.60
    slowdown_strength: float = 0.35
    error_boredom_slope: float = 0.025
    contagion_per_min: float = 0.018
```

To change an experimental assumption, edit the corresponding default or create a modified configuration with `dataclasses.replace()`.

## Main Limitations

1. Behavioural parameters are not empirically calibrated.
2. Initial boredom is imposed exogenously.
3. Underload is represented mainly through rolling utilisation.
4. Workers are homogeneous in their baseline productivity and error risk.
5. Conversations are treated as time losses and do not provide coordination or recovery benefits.
6. The social network is simplified.
7. Travel congestion and warehouse-zone interactions are omitted.
8. Replenishment delays, equipment constraints, and picker-routing decisions are omitted.
9. Error discovery is assumed to be immediate.
10. End-of-shift backlog is represented with a reduced-form monetary penalty.
11. Confidence intervals reflect simulation uncertainty, not full parameter uncertainty.
12. The direct-indirect decomposition is path-dependent.
13. The model has not yet been validated against warehouse data.

## Recommended Thesis Extensions

### 1. Empirical Calibration

Calibrate the model using warehouse-management-system timestamps, order-line records, error logs, idle periods, and repeated short boredom surveys.

A multilevel model could separate:

- Between-worker productivity differences
- Within-worker boredom effects
- Order-complexity effects
- Shift and time-of-day effects

### 2. Independent Validation

Validate against held-out shifts using:

- Picks per hour
- Utilisation
- Error rate
- Flow-time distribution
- Queue length
- Unfinished orders
- Peak-period performance

Calibration and validation should not use the same shifts.

### 3. Empirical Social Networks

Replace random contacts with a network estimated from:

- Work-zone assignments
- Shift rosters
- Break overlap
- Structured observation
- Proximity sensors
- Self-reported interaction partners

Compare random, ring, small-world, clustered, and empirical networks.

### 4. Positive and Negative Conversation Effects

Distinguish among:

- Off-task social conversations
- Work coordination
- Problem solving
- Training
- Safety communication
- Recovery-oriented social contact

Some conversations may create an immediate time loss but reduce future errors or improve later performance.

### 5. Intervention Experiments

Evaluate:

- Job rotation
- Task enrichment
- Cross-training
- Micro-breaks
- Dynamic reassignment
- Demand smoothing
- Low-demand secondary tasks

Report service-level risk and distributional outcomes, not only average cost.

## Suggested Calibration Data

A useful empirical dataset would contain at least:

| Data category | Example fields |
|---|---|
| Orders | Order ID, arrival time, completion time, priority |
| Order lines | SKU, quantity, zone, scan timestamps |
| Workers | Anonymous worker ID, shift, assigned area |
| Quality | Error type, discovery time, rework time, cost |
| Workload | Queue length, active workers, orders/hour |
| Behaviour | Short boredom ratings, perceived monotony, attention |
| Interaction | Conversation timing, partner, purpose, duration |
| Context | Breaks, equipment failures, replenishment delays |

Behavioural observation should follow applicable research-ethics, privacy, employee-representation, and data-protection requirements.

## Reproducibility Notes

- The simulation has no global model state.
- Each replication receives an explicit integer seed.
- Random streams are spawned independently by mechanism.
- Scenarios within one replication receive the same seed.
- Output tables contain replication identifiers and scenario names.
- Re-running the same version with the same configuration and seed should reproduce the same metrics.

For formal thesis archiving, also record:

- Python version
- Dependency versions
- Operating system
- Git commit hash
- Configuration values
- Base seed
- Execution date

Example dependency capture:

```bash
python --version
python -m pip freeze > requirements-lock.txt
```

## Potential Methodological Improvements

For a more rigorous loss attribution, consider:

- Shapley-value decomposition across slowdown, errors, conversations, and contagion
- Bootstrap confidence intervals
- Global sensitivity analysis using Sobol indices
- Bayesian calibration
- Approximate Bayesian computation
- Gaussian-process emulation for expensive parameter searches
- Variance-reduction diagnostics for common random numbers
- Distributional comparison rather than mean-only comparison

## License

No license is included in this prototype. Add an explicit license before distributing the code publicly.

For an academic repository, possible choices include:

- MIT License for permissive reuse
- BSD 3-Clause License for permissive academic and commercial reuse
- GNU GPLv3 if derivative distributions should remain open source

## Citation

If this prototype becomes part of a thesis or publication, replace the placeholder below with the final bibliographic entry:

```text
Author. (Year). Simulating the Team-Level Cost of Boredom in Manual Order Picking.
Master's thesis, University.
```

## Status

**Prototype status:** runnable research prototype  
**Calibration status:** not calibrated  
**Validation status:** not validated  
**Recommended use:** methodological development, experiment design, sensitivity analysis, and preparation for empirical calibration
