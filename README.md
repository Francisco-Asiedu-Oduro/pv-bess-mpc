# Model Predictive Control of a Grid-Connected Solar PV and Battery Energy Storage System

## Overview

This project investigates the use of Model Predictive Control (MPC) to reduce peak grid demand in a grid-connected solar PV and battery energy storage system.

The study combines campus electricity-demand data from the UNICON dataset with photovoltaic generation data from the UNISOLAR dataset for the Bundoora campus.

Three operating strategies are evaluated:

- No-battery baseline
- Rule-based battery controller
- Model Predictive Control (MPC)

The project focuses on developing a complete data-driven battery-control pipeline, including electricity-demand preparation, PV generation reconstruction, battery sizing, battery modelling, controller development, receding-horizon optimization, horizon sensitivity analysis, and forecast-uncertainty evaluation.

## Dataset

The project combines two datasets collected from La Trobe University campuses.

### UNICON

UNICON provides electricity, gas, water, building, campus, and weather information.

The project uses the electricity NMI data to construct campus-level grid demand.

Dataset source:
[UNICON — CDAC Lab](https://github.com/CDAC-lab/UNICON/tree/main) [1]

### UNISOLAR

UNISOLAR contains photovoltaic generation measurements from 42 PV installations across La Trobe University campuses.

The project uses the PV installations located at Bundoora campus.

Dataset source:
[UNISOLAR — CDAC Lab](https://github.com/CDAC-lab/UNISOLAR/tree/main) [2]

Bundoora is identified as `campus_id = 1` in UNICON and contains multiple electricity meters and PV installations, allowing electricity demand and PV generation to be combined for the same physical campus.

## Control Objective

The primary objective is to reduce peak electricity demand drawn from the grid while respecting battery operating constraints.

The grid power balance is represented as:

$$
P_{grid}=P_{load}-P_{PV}+P_{charge}-P_{discharge}
$$

The battery is controlled every 15 minutes using a receding-horizon optimization approach.

The MPC controller considers future demand and PV generation over a six-hour prediction horizon, optimizes battery operation, and implements only the first control action before solving the problem again at the next interval.

## Data Preprocessing

The data preparation pipeline includes:

- Selection of Bundoora campus electricity meters
- Standardization of electricity-demand data to 15-minute intervals
- Investigation of missing and anomalous demand observations
- Meter-level missing-data treatment
- Aggregation of Bundoora NMI meters into campus-level grid demand
- Selection and aggregation of 27 Bundoora PV installations
- Conversion of 15-minute PV energy measurements to average power
- Historical PV generation-share calculation
- Reconstruction of missing PV generation
- Validation of the PV reconstruction threshold
- Alignment of electricity demand and PV generation
- Construction of gross campus load

PV power is calculated from the 15-minute generation measurements as:

$$
P_{PV}=4\times E_{PV}
$$

where the factor of four converts kWh measured over a 15-minute interval into average kW.

## PV Generation Reconstruction

Several Bundoora PV sites contain missing observations.

Instead of treating periods with missing sites as zero generation, the project estimates total PV generation using historical generation shares of the individual PV sites.

Reconstruction is permitted when the missing historical generation share is no greater than 20%.

The threshold was selected from reconstruction validation. Reconstruction error remained relatively low for timestamps with 15–20% missing generation share but increased substantially for the 20–30% range.

This provides a data-driven quality-control threshold for the final PV reconstruction.

## Battery Configuration [3]

A representative lithium-ion battery configuration was selected based on the observed Bundoora demand profile.

The final configuration is:

- Battery power: 1 MW
- Battery capacity: 2 MWh
- Minimum SOC: 20%
- Maximum SOC: 80%
- Initial SOC: 50%
- Round-trip efficiency: 95%
- Charging efficiency: approximately 97.47%
- Discharging efficiency: approximately 97.47%
- Simulation interval: 15 minutes

The charging and discharging efficiencies are obtained by decomposing the 95% round-trip efficiency symmetrically.

The resulting usable energy range is 400–1600 kWh.

## Battery Model

Battery operation is represented using an energy-balance model:

$$E_{k+1}=E_k+\eta_c P_{c,k}\Delta t-\frac{P_{d,k}}{\eta_d}\Delta t,$$

Battery charging and discharging power are constrained by the 1 MW power rating, while the stored energy is constrained by the minimum and maximum SOC limits.

The battery model is also used by the rule-based and MPC controllers so that the strategies are evaluated under the same physical constraints.

## Controllers

### No-Battery Baseline

The battery is not used and the grid demand remains equal to the measured campus grid demand.

### Rule-Based Controller

A threshold-based controller is implemented as a benchmark.

The controller:

- Charges when grid demand is below the selected lower threshold
- Discharges when grid demand exceeds the selected upper threshold
- Remains idle between the thresholds
- Enforces battery power and SOC constraints

The charging threshold is based on the 25th percentile of grid demand, while the discharge threshold is based on the 95th percentile.

### Model Predictive Control

The MPC controller optimizes battery operation over a finite prediction horizon.

A six-hour horizon is used in the main analysis:

- 15-minute control interval
- 24 optimization steps
- 6-hour prediction horizon

At every control interval:

1. Future load and PV generation are provided to the optimizer.
2. Battery charging and discharging are optimized over the prediction horizon.
3. Only the first control action is implemented.
4. The battery state is updated.
5. The horizon moves forward by one interval.
6. The optimization is solved again.

The optimization minimizes the maximum grid demand over the prediction horizon while applying a small penalty to battery throughput.

## Train / Simulation Setup

The MPC study does not train a machine-learning controller.

Instead, the control problem is solved repeatedly using the measured load and PV time series.

Two main simulation settings are evaluated:

- Perfect-forecast MPC
- MPC with synthetic forecast uncertainty

The perfect-forecast case provides an idealized benchmark for the controller, while the uncertainty experiments examine how performance changes when future load and PV generation are imperfectly predicted.

## Results

### Annual Controller Comparison

| Strategy | Peak (kW) | P90 (kW) | P95 (kW) | P97.5 (kW) | P99 (kW) |
|---|---:|---:|---:|---:|---:|
| No BESS | 4572.37 | 3307.20 | 3501.25 | 3638.98 | 3779.94 |
| Rule-based | 4572.37 | 3307.20 | 3501.25 | 3517.39 | 3684.69 |
| MPC | **3773.23** | **3190.57** | **3332.61** | **3445.40** | **3569.68** |

Under the perfect-forecast benchmark, MPC reduced the observed peak grid demand by approximately:

- 799.14 kW
- 17.5%

compared with the no-battery baseline.

The MPC controller maintained the specified battery SOC and power constraints during the simulation.

## Prediction Horizon Sensitivity

The MPC prediction horizon was evaluated using horizons from 2 to 12 hours.

| Horizon | Steps | Peak Reduction |
|---|---:|---:|
| 2 hours | 8 | 4.91% |
| 4 hours | 16 | 7.95% |
| 6 hours | 24 | 9.22% |
| 8 hours | 32 | 9.26% |
| 12 hours | 48 | 9.26% |

The six-hour horizon was selected for the main analysis because extending the horizon beyond six hours produced only small additional improvements in peak reduction while increasing computational time.

## Forecast Uncertainty

Perfect forecasts are unrealistic for practical MPC operation.

To evaluate sensitivity to prediction errors, temporally correlated synthetic forecast errors are introduced using an AR(1)-style process.

The forecast-error model is:

$$\epsilon_k=\rho\epsilon_{k-1}+\sqrt{1-\rho^2}\,z_k,$$

with:

$$
\rho=0.8
$$

The synthetic error level controls the magnitude of the forecast error and is not equivalent to MAPE. Realized forecast errors are calculated separately after generating the forecasts.

The sensitivity analysis evaluates error levels of 5%, 10%, and 20% in addition to the perfect-forecast case.

## Key Findings

- MPC reduced peak grid demand substantially under the perfect-forecast benchmark.
- The six-hour prediction horizon provided most of the peak-shaving benefit observed in the horizon sensitivity analysis.
- The rule-based controller reduced some upper-percentile demand levels but did not reduce the annual peak because the battery reached its minimum SOC before the largest peak event.
- MPC was able to anticipate future demand within the prediction horizon and allocate battery energy across high-demand periods.
- MPC performance deteriorated as forecast uncertainty increased.
- Battery SOC, charging power, and discharging power remained within the specified operating constraints.
- The study demonstrates the importance of coordinating battery operation with both future electricity demand and PV generation rather than responding only to the current grid demand.

## Limitations

The main limitations of the study are:

- The primary annual MPC benchmark assumes perfect knowledge of future load and PV generation.
- Synthetic forecast errors are used to study uncertainty rather than forecasts from a dedicated forecasting model.
- The AR(1) correlation parameter is a scenario assumption rather than an empirically estimated parameter.
- The 1 MW / 2 MWh battery is a representative configuration rather than the result of a full techno-economic optimization.
- Electricity tariffs and demand-charge structures are not explicitly optimized.
- Battery degradation is not explicitly modelled.
- Grid export is not considered in the MPC formulation.
- Network-level constraints such as voltage and feeder loading are not included.

## Tools

Python, Pandas, NumPy, Matplotlib, Seaborn, CVXPY, CLARABEL, Jupyter

## References

[1] H. Moraliyage, N. Mills, P. Rathnayake, D. De Silva, and A. Jennings, "UNICON: An Open Dataset of Electricity, Gas and Water Consumption in a Large Multi-Campus University Setting," 2022 15th International Conference on Human System Interaction (HSI), 2022, pp. 1–8, doi: 10.1109/HSI55341.2022.9869498.

[2] S. Wimalaratne, D. Haputhanthri, S. Kahawala, G. Gamage, D. Alahakoon, and A. Jennings, "UNISOLAR: An Open Dataset of Photovoltaic Solar Energy Generation in a Large Multi-Campus University Setting," 2022 15th International Conference on Human System Interaction (HSI), 2022, pp. 1–5, doi: 10.1109/HSI55341.2022.9869474.

[3] H. A. Banawi, M. O. Bahabri, F. A. Hariri, and M. N. Ajour, “Hybrid Wind–Solar–Fuel Cell–Battery Power System with PI Control for Low-Emission Marine Vessels in Saudi Arabia,” *Automation*, vol. 6, no. 4, Art. 69, 2025. doi: 10.3390/automation6040069.
