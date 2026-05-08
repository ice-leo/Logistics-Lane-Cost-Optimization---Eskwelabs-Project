# Logistics Lane Cost Optimization

**Eskwelabs Innovation Fellowship — Capstone Project**
**Author:** Isaiah John L. Mariano
**Collaborator:** Edrian Abagat

---

## Overview

This repository contains all materials for the Logistics Lane Cost Optimization capstone project completed during the Eskwelabs Innovation Fellowship (EIF). The project addresses a core challenge in supply chain research: the absence of a parametric, scalable benchmark dataset for testing route cost optimization algorithms.

The deliverable is a fully configurable Python synthetic data generator that produces a seven-table relational supply chain dataset modelled after the Brunel University London Supply Chain Logistics Problem benchmark (Kaggle: `anisseezzebdi/supply-chain-logistics-problem`). The generator is accompanied by a comprehensive exploratory data analysis notebook, a formal IMRAD-format research paper, and a data dictionary.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Repository Structure](#repository-structure)
3. [Dataset Schema](#dataset-schema)
4. [Generator Architecture](#generator-architecture)
5. [Configuration Reference](#configuration-reference)
6. [Quick Start](#quick-start)
7. [EDA Summary](#eda-summary)
8. [Key Results](#key-results)
9. [Limitations and Future Work](#limitations-and-future-work)
10. [References](#references)

---

## Problem Statement

Supply chain or operations analysts who want to test optimization models at different network scales, demand volatilities, or constraint densities face two compounding difficulties.

First, publicly available benchmark datasets — including the Brunel University London dataset — represent a single static snapshot of one specific network configuration. Researchers cannot study how an algorithm scales from a 5-warehouse to a 50-warehouse network, how it degrades under high demand volatility, or how constraint density affects solution quality, because the dataset cannot be varied.

Second, real operational supply chain data is commercially sensitive. Actual freight rate cards, warehouse cost structures, and VMI agreements are not published, making it impossible to obtain multiple independent samples for stress-testing.

A configurable synthetic data generator that faithfully reproduces the statistical structure and constraint topology of a real-world benchmark dataset addresses both problems simultaneously. It gives researchers unlimited, reproducible data at any network scale while guaranteeing that every generated instance is a valid, feasible input to a downstream optimization model.

---

## Repository Structure

```
logistics-lane-cost-optimization/
|
|-- Synthetic_Data_Generator.ipynb      # Main generator notebook
|-- Synthetic_Supply_Chain_EDA.ipynb    # Exploratory data analysis notebook
|-- IMRAD_Paper.docx                    # Formal research paper (IMRAD format)
|-- Capstone_Slides.pdf                 # EIF capstone presentation slides
|-- data_dictionary.md                  # Feature definitions for all seven tables
|-- README.md                           # This file
|
|-- supply_chain_output/                # Default output directory (generated at runtime)
    |-- OrderList.csv
    |-- FreightRates.csv
    |-- WhCosts.csv
    |-- WhCapacities.csv
    |-- ProductsPerPlant.csv
    |-- VmiCustomers.csv
    |-- PlantPorts.csv
```

---

## Dataset Schema

The generator produces all seven tables of the Brunel University London supply chain schema. The tables form a hub-and-spoke relational structure with `OrderList` at the centre.

| Table | Role | Rows (default scale) | Primary Key |
|---|---|---|---|
| `OrderList` | Central transactional fact table; one row per customer order | 10,000 | Order_ID |
| `FreightRates` | Carrier x lane x weight-bracket rate lookup | ~1,610 | Carrier + Lane + Weight range |
| `PlantPorts` | Permitted warehouse-to-port shipping links | 42 | Plant_Code + Port |
| `ProductsPerPlant` | Authorized warehouse-product combinations | ~1,932 | Plant_Code + Product_ID |
| `VmiCustomers` | VMI customer restrictions per warehouse | 113 | Plant_Code + Customer |
| `WhCapacities` | Maximum daily order throughput per warehouse | 20 | Plant_ID |
| `WhCosts` | Storage cost per unit per warehouse (USD) | 20 | WH |

### Hard Constraints

Any valid routing solution must simultaneously satisfy all six constraint tables. The generator enforces the following constraints by construction during row assembly — not post-hoc:

- **Plant-product constraint**: the plant selected for an order must stock the ordered product, as defined in `ProductsPerPlant`.
- **Plant-port constraint**: the origin port assigned to an order must be linked to the selected plant in `PlantPorts`.
- **VMI constraint**: if the customer is a VMI customer, the product must be drawn from the subset stocked by VMI-eligible plants, and the plant must be one of the VMI-designated plants linked to that customer in `VmiCustomers`.
- **Capacity constraint**: warehouse daily capacity is set to exceed the actual daily order load by a configurable buffer, guaranteeing feasibility.
- **Carrier coverage constraint**: every carrier referenced in `OrderList` has a corresponding entry in `FreightRates`.
- **Date range constraint**: all order dates fall within the configured `start_date` and `end_date`.

The post-generation validation suite runs 19 automated checks and confirms zero violations in the default configuration.

---

## Generator Architecture

The generator is implemented as two Python objects defined in `Synthetic_Data_Generator.ipynb`:

- `GeneratorConfig`: a `dataclass` that exposes every distributional parameter, scale parameter, and structural setting as a typed field with validation logic in `__post_init__`.
- `SupplyChainDataGenerator`: a class that consumes a `GeneratorConfig` and executes a dependency-ordered generation pipeline via its `run()` method.

### Generation Pipeline

Tables are generated in strict dependency order to ensure that all foreign keys exist before they are referenced:

| Step | Table | Method | Depends on |
|---|---|---|---|
| 1 | `ProductsPerPlant` | `generate_products_per_plant()` | Plant IDs, Product IDs, VMI plant list |
| 2 | `VmiCustomers` | `generate_vmi_customers()` | Customer IDs, VMI plant list |
| 3 | `PlantPorts` | `generate_plant_ports()` | Plant IDs, Port IDs |
| 4 | `WhCosts` | `generate_wh_costs()` | Plant IDs |
| 5 | `FreightRates` | `generate_freight_rates()` | Carrier route map |
| 6 | `OrderList` | `generate_order_list()` | All tables above |
| 7 | `WhCapacities` | `generate_wh_capacities()` | OrderList (actual daily load) |

All random draws use a single seeded `numpy.random.default_rng` object, guaranteeing full reproducibility from a given seed value.

### OrderList Row Assembly

`OrderList` generation is the most complex step because it must jointly satisfy all three hard referential constraints. Generation proceeds in two phases:

1. **Vectorized pre-draws**: dimension-independent quantities (customer, date, service level, TPT, ship-ahead days, ship-late days, unit quantity, unit weight) are sampled in batch using NumPy for performance.
2. **Constraint-aware loop**: for each order, the product, plant, and port are selected sequentially, checking VMI eligibility, product-plant authorization, and plant-port connectivity at each step.

This approach ensures that no infeasible row is ever written to the output DataFrame.

### FreightRates Construction

The `FreightRates` table is built from a carrier route map constructed by `_build_carrier_routes()`. Each carrier is assigned:

- A primary transport mode drawn from `{AIR, GROUND, SEA}` with probability `(0.50, 0.30, 0.20)`.
- A second mode with probability `0.25` (multimodal carriers).
- Between 1 and 4 origin ports and 1 to 3 destination ports.

For each (carrier, mode, route, service code, weight bracket) combination, the rate is computed with a 5% per-bracket compounding volume discount applied to a base rate drawn from a mode-specific range:

```
rate[i] = base_rate * (0.95)^i    for bracket index i = 0, 1, ..., K
```

This encodes the real-world tiered pricing structure found in LTL and air cargo rate cards.

---

## Configuration Reference

All parameters are set in the `GeneratorConfig` dataclass. The table below documents every configurable parameter, its default value, and its effect.

### Scale Parameters

| Parameter | Default | Description |
|---|---|---|
| `n_orders` | 10,000 | Total number of rows in `OrderList` |
| `n_customers` | 500 | Number of unique customer identifiers |
| `n_products` | 200 | Size of the product catalogue |
| `n_plants` | 20 | Number of warehouses |
| `n_ports` | 8 | Number of shipping ports |
| `n_carriers` | 12 | Number of freight carriers |
| `start_date` | `"2026-01-01"` | Start of the order date range |
| `end_date` | `"2026-01-02"` | End of the order date range |
| `seed` | 42 | NumPy RNG seed for reproducibility |
| `output_format` | `"csv"` | Output file format: `"csv"` or `"parquet"` |

### VMI and Coverage Parameters

| Parameter | Default | Description |
|---|---|---|
| `vmi_plant_ids` | `None` | Explicit tuple of VMI plant IDs. If `None`, ~20% of plants are auto-selected. |
| `vmi_customer_ratio` | `0.15` | Fraction of customers that are VMI customers |
| `min_products_per_plant` | `0.30` | Each plant carries at least this fraction of the product catalogue |
| `max_products_per_plant` | `0.70` | Each plant carries at most this fraction of the product catalogue |
| `capacity_buffer` | `0.20` | Warehouse capacity is set to (peak daily load) x (1 + buffer) |

### Order Quantity Distribution

| Parameter | Default | Description |
|---|---|---|
| `order_qty_dist` | `"uniform"` | Distribution family: `"uniform"`, `"lognormal"`, or `"zipf"` |
| `min_order_qty` | `5` | Minimum order quantity (all distributions) |
| `max_order_qty` | `50` | Maximum order quantity (all distributions) |
| `order_qty_lognormal_mean` | `3.5` | Log-scale mean for lognormal mode |
| `order_qty_lognormal_sigma` | `0.8` | Log-scale standard deviation for lognormal mode |
| `order_qty_zipf_a` | `1.5` | Zipf exponent for Zipf mode |

### Unit Weight Distribution

| Parameter | Default | Description |
|---|---|---|
| `weight_per_unit_dist` | `"uniform"` | Distribution family: `"uniform"` or `"gamma"` |
| `weight_per_unit_min` | `0.1` | Minimum weight per unit in kg (uniform mode) |
| `weight_per_unit_max` | `5.0` | Maximum weight per unit in kg (uniform mode) |
| `weight_per_unit_gamma_shape` | `2.0` | Gamma shape parameter k (gamma mode) |
| `weight_per_unit_gamma_scale` | `1.5` | Gamma scale parameter theta (gamma mode); mean = k x theta |

### Service Level and Timing Distributions

| Parameter | Default | Description |
|---|---|---|
| `service_level_probs` | `(0.67, 0.23, 0.10)` | PMF over [DTP, DTD, CRF] |
| `tpt_probs` | `(0.05, 0.23, 0.70, 0.01, 0.01)` | PMF over TPT values [0, 1, 2, 3, 4] days |
| `ship_ahead_probs` | `(0.25, 0.05, 0.05, 0.40, 0.10, 0.10, 0.05)` | PMF over ship-ahead day counts [0..6] |
| `ship_late_probs` | `(0.95, 0.01, 0.01, 0.01, 0.01, 0.005, 0.005)` | PMF over ship-late day counts [0..6] |

### Carrier and Freight Rate Parameters

| Parameter | Default | Description |
|---|---|---|
| `carrier_mode_probs` | `(0.50, 0.30, 0.20)` | PMF over [AIR, GROUND, SEA] for primary mode assignment |
| `carrier_multimodal_prob` | `0.25` | Probability a carrier operates in more than one mode |
| `ports_per_carrier_range` | `(1, 4)` | Min and max number of origin ports covered by a carrier |
| `weight_brackets_kg` | `(5,10,25,50,100,300,500,1000,5000)` | Upper bounds (kg) for freight rate weight tiers |
| `air_rate_range` | `(0.50, 5.00)` | Base rate range (USD/kg) for AIR mode |
| `ground_rate_range` | `(0.05, 0.80)` | Base rate range (USD/kg) for GROUND mode |
| `sea_rate_range` | `(0.02, 0.30)` | Base rate range (USD/kg) for SEA mode |
| `air_tpt_days` | `(2.0, 1.0)` | Normal distribution (mean, std) for AIR transit days |
| `ground_tpt_days` | `(3.0, 1.0)` | Normal distribution (mean, std) for GROUND transit days |
| `sea_tpt_days` | `(20.0, 5.0)` | Normal distribution (mean, std) for SEA transit days |

### Warehouse Cost and Capacity Parameters

| Parameter | Default | Description |
|---|---|---|
| `wh_cost_dist` | `"lognormal"` | Distribution family: `"uniform"` or `"lognormal"` |
| `wh_cost_min` | `0.30` | Minimum warehouse cost per unit in USD (uniform mode) |
| `wh_cost_max` | `2.50` | Maximum warehouse cost per unit in USD (uniform mode) |
| `wh_cost_lognormal_mean` | `0.0` | Log-scale mean for lognormal mode (exp(0) ~ $1/unit) |
| `wh_cost_lognormal_sigma` | `0.6` | Log-scale standard deviation for lognormal mode |
| `wh_capacity_floor` | `5` | Minimum daily capacity regardless of order load |

---

## Quick Start

### Requirements

```bash
pip install numpy pandas scipy matplotlib seaborn
```

### Running the Generator

1. Open `Synthetic_Data_Generator.ipynb` in Jupyter.
2. Modify the `GeneratorConfig` dataclass fields to set your desired scale, distributions, and constraints.
3. Run all cells. The generator will produce all seven tables and save them to the `supply_chain_output/` directory.

### Minimal Example

```python
from dataclasses import dataclass

# Use default configuration
cfg = GeneratorConfig()

# Or override specific parameters
cfg = GeneratorConfig(
    n_orders=50_000,
    n_plants=40,
    n_carriers=20,
    order_qty_dist="lognormal",
    weight_per_unit_dist="gamma",
    seed=123,
    output_format="parquet"
)

generator = SupplyChainDataGenerator(cfg)
tables = generator.run()
```

### Validation

After generation, run the validation suite to confirm all 19 checks pass:

```python
generator.run_validation(tables)
# Expected output: All 19 validation checks passed.
```

---

## EDA Summary

The EDA notebook (`Synthetic_Supply_Chain_EDA.ipynb`) contains 10 sections covering distributional analysis, temporal patterns, freight rate structure, warehouse network properties, constraint compliance verification, cost estimation, and a structured realism assessment.

### Notebook Structure

| Section | Contents |
|---|---|
| 1. Setup and Data Loading | Imports, data loading, datetime parsing |
| 2. Dataset Overview and Schema Validation | Row counts, null checks, referential integrity |
| 3. OrderList Deep Dive | Descriptive statistics, distribution plots for all 14 fields |
| 4. Temporal Analysis | Monthly, weekly, and day-of-week order volume patterns |
| 5. FreightRates Analysis | Rate distributions by mode, volume discount structure, TPT by mode |
| 6. Warehouse Network Analysis | Plant utilization, HHI concentration index, cost-capacity relationship |
| 7. Constraint Compliance Checks | VMI routing, plant-product, plant-port verification |
| 8. Cost Estimation | Freight cost vs. warehouse cost split, per-order cost distribution |
| 9. Statistical Tests | Eight formal hypothesis tests with interpretation |
| 10. Realism Assessment | Structured scoring against published logistics literature benchmarks |

### Statistical Tests Applied

| Test | Variable(s) | Null Hypothesis | Result |
|---|---|---|---|
| Shapiro-Wilk (n=500 sample) | Order weight | Weights are normally distributed | Rejected (W=0.954, p<0.001) |
| Kolmogorov-Smirnov | Order weight vs. log-normal | Weights follow a log-normal distribution | Not rejected (D=0.015, p=0.267) |
| Spearman rank correlation | Weight vs. unit quantity | No monotonic association | r=0.003, p=0.75 (no association) |
| Kruskal-Wallis H-test | Freight rate across transport modes | Rates are equal across modes | Rejected (H=1186.4, p<10^-250) |
| Chi-squared independence | Carrier vs. service level | Carrier choice is independent of service level | Rejected (chi-sq=48.5, p=0.001) |
| Spearman rank correlation | AIR rate vs. weight bracket midpoint | No monotonic association | r=-0.48, p<0.001 (volume discount confirmed) |
| Mann-Whitney U | Order weight: late vs. on-time orders | Weight distributions are equal | Not rejected (p=0.52) |
| Pearson correlation | Warehouse capacity vs. cost per unit | No linear association | r=-0.04, p=0.86 (no association) |

---

## Key Results

### Constraint Compliance

The validation suite confirmed zero violations across all five constraint checks in the default configuration. The daily peak load per plant never exceeded the assigned capacity. The maximum observed peak was 8 orders per day against capacities ranging from 10 to 99 orders per day.

### OrderList Descriptive Statistics

| Field | Mean | Std | Min | Median | Max |
|---|---|---|---|---|---|
| Weight (kg) | 70.82 | 55.00 | 0.55 | 56.79 | 248.03 |
| Unit quantity | 27.67 | 13.35 | 5 | 28 | 50 |
| TPT (days) | 1.69 | 0.62 | 0 | 2 | 4 |
| Ship-late (days) | 0.10 | 0.61 | 0 | 0 | 6 |

Service level split: DTP 66.2%, DTD 23.5%, CRF 10.3%.
On-time delivery rate: 95.1%.

### Freight Rate Summary

| Mode | Mean rate (USD/kg) | Median TPT (days) |
|---|---|---|
| AIR | 2.17 | 2 |
| GROUND | 0.33 | 4 |
| SEA | 0.13 | 18 |

Rates differ significantly across modes (Kruskal-Wallis H=1,186.4, p<10^-250). Volume discounts confirmed within AIR mode (Spearman r=-0.48, p<0.001).

### Warehouse Network

- Herfindahl-Hirschman Index (HHI): 0.062 — well below the 0.15 concentrated-network threshold, confirming a dispersed order allocation.
- VMI restrictions applied to 4 plants (20% of the network), covering 75 unique customers (15% of all customers).
- Freight cost accounts for approximately 95% of total per-order cost, consistent with international supply chains where transport dominates storage cost.

### Realism Score

The structured realism assessment yielded an overall score of **7.5 out of 10**, benchmarked against published logistics literature. The primary areas of departure from real-world data are documented in the Limitations section below.

---

## Limitations and Future Work

Three structural properties of the generated data diverge from real-world observations and should be considered before using the dataset for behavioural or predictive modelling (as opposed to optimization benchmarking).

### Demand Seasonality

The monthly coefficient of variation is 6.0%, substantially lower than the 15-40% range reported for B2B supply chains and the 40-80%+ range in seasonal consumer goods. This flatness arises from the uniform date sampling in the current implementation. To introduce realistic seasonality, replace the uniform date offset with a seasonal mixture model or a Poisson process with a time-varying rate function (e.g., a von Mises distribution peaked at Q4 for consumer goods).

### Warehouse Economies of Scale

The Pearson correlation between warehouse capacity and storage cost is r=-0.04 (p=0.86), indicating no statistically significant relationship. Real warehouse cost models predict that per-unit costs decrease with facility size due to fixed cost amortization. A future version should sample costs inversely proportional to capacity with additive noise to reproduce this property.

### Carrier Service Specialization

The chi-squared test detected a statistically significant but practically weak association between carrier choice and service level. Real freight carriers exhibit strong specialization: express air couriers almost exclusively handle DTD shipments, while sea carriers are predominantly DTP or CRF. The generator can be extended with a carrier specialization map that restricts service codes more tightly by mode.

### Weight Brackets

The current implementation uses identical weight bracket boundaries for all carriers. In the original Brunel dataset, carriers have heterogeneous bracket structures that reflect their pricing strategies and vehicle capacity profiles. Implementing per-carrier bracket sampling would improve the structural fidelity of the `FreightRates` table.

### Guaranteed Dataset Production

The current implementation may encounter infeasibility under extreme parameter combinations (e.g., very high VMI ratios combined with low product-plant coverage). A future version should guarantee dataset production for any valid configuration, potentially by adding a feasibility score that quantifies how closely the output matches the target parameters when soft constraints cannot be fully satisfied.

---

## References

anisseezzebdi (2023). Supply Chain Logistics Problem [Dataset]. Kaggle. https://www.kaggle.com/datasets/anisseezzebdi/supply-chain-logistics-problem

Baker, P., and Canessa, M. (2009). Warehouse design: A structured approach. *European Journal of Operational Research*, 193(2), 425-436.

Gartner (2023). *Gartner Supply Chain Top 25 for 2023*. Gartner Research.

Holguín-Veras, J., Jaller, M., Destro, L., Ban, X., Lawson, C., and Levinson, H. S. (2011). Freight generation, freight trip generation, and the perils of using constant trip rates. *Transportation Research Record*, 2224(1), 68-81.

Ivars-Dzalbs, A. (2019). Supply Chain Logistics Problem Dataset. Brunel University London. https://doi.org/10.17633/rd.brunel.7558679.v1

NumPy Developers (2023). numpy.random.default_rng — NumPy v1.26 Manual. https://numpy.org/doc/stable/reference/random/generator.html

---

## License

This project was developed as part of the Eskwelabs Innovation Fellowship internship program. All code and documentation are available for academic and research use.
