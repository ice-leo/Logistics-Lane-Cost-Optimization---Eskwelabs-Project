# Supply Chain Synthetic Dataset Generator

**Eskwelabs Innovation Fellowship — Capstone Project**
**Authors:** Isaiah John L. Mariano and Edrian Abagat

---

## Overview

This tool generates synthetic supply chain logistics datasets modelled after the seven-table relational schema of the Brunel University London Supply Chain Logistics Problem. It is designed for supply chain and operations analysts who need scalable, reproducible, constraint-compliant benchmark data for testing route cost optimization algorithms, particularly Linear Programming and Mixed-Integer Programming models, at arbitrary network scales and under configurable distributional assumptions.

The core idea is that every distributional assumption in the generator is fully exposed and user-controlled. You decide the network size, the shape of the demand distribution, how freight rates are structured, what fraction of customers are VMI-restricted, and how warehouse costs are drawn. The generator then produces a dataset that is internally consistent, referentially complete, and guaranteed to satisfy all hard network constraints by construction.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Repository Structure](#repository-structure)
3. [How the Algorithm Works](#how-the-algorithm-works)
4. [Dataset Schema](#dataset-schema)
5. [Configuration Reference](#configuration-reference)
6. [Quick Start](#quick-start)
7. [Alternate Configuration Examples](#alternate-configuration-examples)
8. [Validation Suite](#validation-suite)
9. [Limitations and Known Gaps](#limitations-and-known-gaps)
10. [References](#references)

---

## Problem Statement

Supply chain or operations analysts who want to test optimization models at different network scales, demand volatilities, or constraint densities face two compounding difficulties.

First, publicly available benchmark datasets, including the Brunel University London dataset, represent a single static snapshot of one specific network configuration. A researcher cannot study how an algorithm scales from a 5-warehouse to a 50-warehouse network, or how it degrades under high demand volatility, because the dataset cannot be varied.

Second, edge cases or unusual constraint combinations that stress-test an optimizer's feasibility handling appear rarely or not at all in fixed datasets. A configurable generator lets you amplify constraint density, skew the demand distribution, or force high VMI ratios until the edge cases you care about appear with sufficient frequency.

This generator addresses both problems: it produces datasets at any scale with any distributional shape, while guaranteeing that every generated instance is a valid, feasible input to a downstream optimization model.

---

## Repository Structure

```
logistics-lane-cost-optimization/
|
|-- Synthetic_Dataset_Generator.ipynb      # Generator: GeneratorConfig + SupplyChainDataGenerator
|-- Synthetic_Supply_Chain_EDA.ipynb       # EDA and statistical validation of a generated dataset
|-- IMRAD_Paper.docx                       # Formal research paper documenting methodology and results
|-- Capstone_Slides.pdf                    # EIF capstone presentation slides
|-- README.md                              # This file
|
|-- supply_chain_output/                   # Default output directory (created at runtime)
    |-- OrderList.csv
    |-- FreightRates.csv
    |-- WhCosts.csv
    |-- WhCapacities.csv
    |-- ProductsPerPlant.csv
    |-- VmiCustomers.csv
    |-- PlantPorts.csv
```

---

## How the Algorithm Works

The generator is implemented as two Python objects: `GeneratorConfig`, a dataclass that holds every parameter, and `SupplyChainDataGenerator`, a class that consumes a config and runs a dependency-ordered pipeline of seven sub-generators via its `run()` method.

### Design principles

**Dependency-ordered generation.** Tables are generated in an order that ensures every foreign key exists before it is referenced. Reference tables with no dependencies (PlantPorts, WhCosts) are generated first. The central fact table (OrderList) is generated last, after all lookup tables are in place. WhCapacities is generated after OrderList because its capacity values are derived from the actual order load.

**Constraints enforced by construction, not post-hoc.** The most important design decision is that the three hard referential constraints: plant-product authorization, plant-port connectivity, and VMI routing, are enforced during row assembly in the OrderList loop, not checked and discarded afterward. This means no generated row is ever infeasible. The validation suite's role is to confirm this, not to filter bad rows.

**Full reproducibility via seeded RNG.** All random draws throughout the entire pipeline flow through a single `numpy.random.default_rng` object initialized from the user-supplied `seed` parameter. Fixing the seed and the config produces identical output across runs and machines.

### Pipeline steps

| Step | Table | Method | What it does |
|---|---|---|---|
| 1 | `ProductsPerPlant` | `generate_products_per_plant()` | Assigns each plant a random subset of the product catalogue. Runs two post-processing passes to guarantee every product is covered by at least one plant, and at least one VMI plant. |
| 2 | `VmiCustomers` | `generate_vmi_customers()` | Designates a fraction of customers as VMI customers and links each to 1-2 VMI plants. |
| 3 | `PlantPorts` | `generate_plant_ports()` | Connects each plant to 1-3 ports drawn without replacement, producing a sparse bipartite graph. |
| 4 | `WhCosts` | `generate_wh_costs()` | Draws a per-unit storage cost for each plant from a uniform or lognormal distribution. |
| 5 | `FreightRates` | `generate_freight_rates()` | Builds a weight-bracket rate table for every (carrier, route, mode, service code, bracket) combination, with volume discounts applied per bracket tier. |
| 6 | `OrderList` | `generate_order_list()` | The most complex step. Pre-draws dimension-independent quantities in batch, then resolves plant, product, and port for each row in a constraint-aware loop. |
| 7 | `WhCapacities` | `generate_wh_capacities()` | Sets each plant's daily capacity to its actual peak daily load from OrderList, multiplied by a buffer. This guarantees capacity constraints are satisfiable. |

### The OrderList constraint-aware loop

`OrderList` generation is split into two phases to balance performance and correctness.

**Phase 1: Vectorized pre-draws.** All dimension-independent quantities are sampled in batch using NumPy:

- Customer index drawn uniformly from the customer pool.
- Order date drawn uniformly over the configured date range.
- Service level drawn from a user-specified PMF over `{DTP, DTD, CRF}`.
- TPT drawn from a user-specified PMF over `{0, 1, 2, 3, 4}` days.
- Ship-ahead and ship-late day counts each drawn from user-specified PMFs.
- Unit quantity drawn from the user-selected distribution (uniform, lognormal, or Zipf).
- Unit weight drawn from the user-selected distribution (uniform or gamma).
- Carrier drawn uniformly from the carrier pool.

**Phase 2: Per-row constraint resolution.** For each order, the product, plant, and port are resolved sequentially:

1. If the customer is a VMI customer, the product is drawn from the subset of products that are stocked by at least one VMI-eligible plant. Otherwise it is drawn from the full catalogue.
2. The plant is drawn from the set of plants that stock the selected product according to `ProductsPerPlant`. If the customer is VMI, this set is further restricted to VMI-designated plants.
3. The origin port is drawn from the ports linked to the selected plant in `PlantPorts`.

Because all three lookups are pre-computed as dictionaries before the loop, per-row resolution is fast despite being sequential.

### FreightRates construction

The `FreightRates` table is the most structurally complex because it expands across three dimensions: routes, weight brackets, and service codes.

**Carrier route map.** Before rate construction, `_build_carrier_routes()` assigns each carrier a primary transport mode drawn from `{AIR, GROUND, SEA}` with a user-specified probability vector. A carrier becomes multimodal with a configurable probability, in which case a second mode is drawn from the remaining two. Each carrier then receives between `mn_ports` and `mx_ports` origin ports and between 1 and 3 destination ports, drawn without replacement from the port identifier set.

**Weight brackets.** The bracket list is derived from the user-supplied `weight_brackets_kg` tuple of upper bounds. The final bracket is left open (upper bound set to 99999.99 kg). Each bracket represents one row per (carrier, route, mode, service code) combination.

**Volume discount rate construction.** For each (carrier, mode, route, service code) group, a base rate is drawn uniformly from a mode-specific range. Rates for each successive weight bracket are computed by applying a 5% compounding discount:

```
rate[i] = base_rate * (0.95)^i    for bracket index i = 0, 1, ..., K
```

This encodes the tiered pricing structure found in real LTL and air cargo rate cards, where heavier shipments receive lower per-kg rates.

**Service code assignment by mode.** Service codes are assigned to modes as follows: AIR carriers offer `DTD` and `DTP`; GROUND carriers offer `DTP` and `CRF`; SEA carriers offer `DTD`, `DTP`, and `CRF`. This reflects real-world specialization where express carriers dominate DTD and sea freight is predominantly used for deferred delivery.

**Transit days.** Per-carrier transit days are drawn from a normal distribution parameterized by mode, then clipped to zero:

```
TPT = max(0, round(Normal(mu_mode, sigma_mode)))
```

Default parameters: AIR (mu=2, sigma=1), GROUND (mu=3, sigma=1), SEA (mu=20, sigma=5).

### WhCapacities derivation

Unlike all other tables, `WhCapacities` is not drawn from a distribution. It is derived from the actual order load in `OrderList`:

```
load_p     = ceil( sum(Unit_qty | Plant = p) / n_days )
Capacity_p = max(capacity_floor, ceil(load_p * (1 + capacity_buffer)))
```

Deriving capacity from actual load is what guarantees that the daily capacity constraint is feasible by construction. A generator that drew capacities independently from order volumes would frequently produce infeasible instances.

---

## Dataset Schema

The generator produces all seven tables of the Brunel University London supply chain schema. The tables form a hub-and-spoke relational structure with `OrderList` at the centre.

| Table | Role | Primary Key |
|---|---|---|
| `OrderList` | Central transactional fact table; one row per customer order | Order_ID |
| `FreightRates` | Carrier x lane x weight-bracket rate lookup | Carrier + Lane + Weight range |
| `PlantPorts` | Permitted warehouse-to-port shipping links | Plant_Code + Port |
| `ProductsPerPlant` | Authorized warehouse-product combinations | Plant_Code + Product_ID |
| `VmiCustomers` | VMI customer restrictions per warehouse | Plant_Code + Customer |
| `WhCapacities` | Maximum daily order throughput per warehouse | Plant_ID |
| `WhCosts` | Storage cost per unit per warehouse (USD) | WH |

### Hard constraints

Any valid routing solution must simultaneously satisfy all six constraint tables. The generator guarantees all of the following hold with zero violations:

- Every plant code in `OrderList` exists in `ProductsPerPlant` and the ordered product is authorized for that plant.
- Every origin port in `OrderList` is linked to the order's plant in `PlantPorts`.
- Every VMI customer in `OrderList` is served by a VMI-designated plant.
- Every carrier in `OrderList` has a corresponding entry in `FreightRates`.
- No plant's daily order volume exceeds its capacity in `WhCapacities`.
- All order dates fall within the configured date range.

---

## Configuration Reference

All parameters are set in the `GeneratorConfig` dataclass. Invalid configurations raise a `ValueError` at instantiation time with a descriptive error message.

### Scale

| Parameter | Default | Description |
|---|---|---|
| `n_orders` | 10,000 | Number of rows in `OrderList` |
| `n_customers` | 500 | Number of unique customer identifiers |
| `n_products` | 200 | Size of the product catalogue |
| `n_plants` | 20 | Number of warehouses |
| `n_ports` | 8 | Number of shipping ports |
| `n_carriers` | 12 | Number of freight carriers |
| `start_date` | `"2026-01-01"` | Start of the order date range (YYYY-MM-DD) |
| `end_date` | `"2026-12-31"` | End of the order date range (YYYY-MM-DD) |
| `seed` | 42 | NumPy RNG seed |
| `output_format` | `"csv"` | Output file format: `"csv"` or `"parquet"` |

### VMI and constraint density

| Parameter | Default | Description |
|---|---|---|
| `vmi_plant_ids` | `None` | Explicit tuple of VMI plant IDs. If `None`, ~20% of plants are auto-selected. |
| `vmi_customer_ratio` | 0.15 | Fraction of customers designated as VMI customers |
| `min_products_per_plant` | 0.30 | Each plant carries at least this fraction of the catalogue |
| `max_products_per_plant` | 0.70 | Each plant carries at most this fraction of the catalogue |
| `capacity_buffer` | 0.20 | Warehouse capacity = peak daily load x (1 + buffer) |
| `wh_capacity_floor` | 5 | Minimum daily capacity regardless of load |

### Order quantity distribution

Three distribution families are supported, selected via `order_qty_dist`.

| Parameter | Default | Description |
|---|---|---|
| `order_qty_dist` | `"uniform"` | `"uniform"`, `"lognormal"`, or `"zipf"` |
| `min_order_qty` | 5 | Lower bound applied to all distributions |
| `max_order_qty` | 50 | Upper bound applied to all distributions |
| `order_qty_lognormal_mean` | 3.5 | Log-scale mean (lognormal mode) |
| `order_qty_lognormal_sigma` | 0.8 | Log-scale standard deviation (lognormal mode) |
| `order_qty_zipf_a` | 1.5 | Zipf exponent (zipf mode) |

Use `"uniform"` for symmetric, bounded demand. Use `"lognormal"` to produce right-skewed demand typical of B2B supply chains. Use `"zipf"` for heavy-tailed demand typical of e-commerce, where a small number of products account for most order volume.

### Unit weight distribution

| Parameter | Default | Description |
|---|---|---|
| `weight_per_unit_dist` | `"uniform"` | `"uniform"` or `"gamma"` |
| `weight_per_unit_min` | 0.1 | Minimum kg per unit (uniform mode) |
| `weight_per_unit_max` | 5.0 | Maximum kg per unit (uniform mode) |
| `weight_per_unit_gamma_shape` | 2.0 | Gamma shape k (gamma mode) |
| `weight_per_unit_gamma_scale` | 1.5 | Gamma scale theta (gamma mode); mean = k x theta |

Total order weight is the product of unit weight and unit quantity, so the weight column reflects the combined distribution of both.

### Service level and timing distributions

Each of the following is a probability mass function and must sum to 1.0.

| Parameter | Default | Description |
|---|---|---|
| `service_level_probs` | `(0.67, 0.23, 0.10)` | PMF over [DTP, DTD, CRF] |
| `tpt_probs` | `(0.05, 0.23, 0.70, 0.01, 0.01)` | PMF over TPT values [0, 1, 2, 3, 4] days |
| `ship_ahead_probs` | `(0.25, 0.05, 0.05, 0.40, 0.10, 0.10, 0.05)` | PMF over ship-ahead day counts [0..6] |
| `ship_late_probs` | `(0.95, 0.01, 0.01, 0.01, 0.01, 0.005, 0.005)` | PMF over ship-late day counts [0..6] |

### Carrier and freight rate parameters

| Parameter | Default | Description |
|---|---|---|
| `carrier_mode_probs` | `(0.50, 0.30, 0.20)` | PMF over [AIR, GROUND, SEA] for primary mode assignment |
| `carrier_multimodal_prob` | 0.25 | Probability a carrier operates more than one mode |
| `ports_per_carrier_range` | `(1, 4)` | Min and max origin ports covered per carrier |
| `weight_brackets_kg` | `(5,10,25,50,100,300,500,1000,5000)` | Upper bounds (kg) for freight rate tiers; a final open bracket is added automatically |
| `air_rate_range` | `(0.50, 5.00)` | Base rate range (USD/kg) for AIR |
| `ground_rate_range` | `(0.05, 0.80)` | Base rate range (USD/kg) for GROUND |
| `sea_rate_range` | `(0.02, 0.30)` | Base rate range (USD/kg) for SEA |
| `air_min_cost_range` | `(30.0, 120.0)` | Flat minimum cost range (USD) for AIR |
| `ground_min_cost_range` | `(5.0, 30.0)` | Flat minimum cost range (USD) for GROUND |
| `sea_min_cost_range` | `(50.0, 500.0)` | Flat minimum cost range (USD) for SEA |
| `air_tpt_days` | `(2.0, 1.0)` | Normal (mean, std) for AIR transit days |
| `ground_tpt_days` | `(3.0, 1.0)` | Normal (mean, std) for GROUND transit days |
| `sea_tpt_days` | `(20.0, 5.0)` | Normal (mean, std) for SEA transit days |

### Warehouse cost distribution

| Parameter | Default | Description |
|---|---|---|
| `wh_cost_dist` | `"lognormal"` | `"uniform"` or `"lognormal"` |
| `wh_cost_min` | 0.30 | Minimum USD/unit (uniform mode) |
| `wh_cost_max` | 2.50 | Maximum USD/unit (uniform mode) |
| `wh_cost_lognormal_mean` | 0.0 | Log-scale mean (lognormal mode); exp(0) ~ $1/unit |
| `wh_cost_lognormal_sigma` | 0.6 | Log-scale standard deviation (lognormal mode) |

---

## Quick Start

### Requirements

```bash
pip install numpy pandas scipy matplotlib seaborn
```

For parquet output, also install `pyarrow`. If `pyarrow` is not found, the generator falls back to CSV automatically.

### Running the generator

1. Open `Synthetic_Dataset_Generator.ipynb` in Jupyter.
2. In Section 5 ("Run the generator"), edit the `GeneratorConfig(...)` block to set your parameters.
3. Run all cells. The seven tables are stored in the `tables` dict and, if `gen.save(tables)` is called, written to `supply_chain_output/`.

### Minimal usage in Python

```python
# Default configuration
config = GeneratorConfig()
gen = SupplyChainDataGenerator(config)
tables = gen.run()
run_validation(tables, config)

# Access any table as a pandas DataFrame
order_list = tables["OrderList"]
freight_rates = tables["FreightRates"]

# Save all tables to disk
gen.save(tables)
```

### Scaling up the network

```python
config = GeneratorConfig(
    n_orders=100_000,
    n_plants=50,
    n_carriers=30,
    n_ports=15,
    n_customers=2_000,
    n_products=500,
    seed=7,
)
```

### Changing the demand distribution

```python
# Right-skewed demand (lognormal) — typical of B2B manufacturing
config = GeneratorConfig(
    order_qty_dist="lognormal",
    order_qty_lognormal_mean=3.5,
    order_qty_lognormal_sigma=1.0,
    min_order_qty=1,
    max_order_qty=500,
)

# Heavy-tailed demand (Zipf) — typical of e-commerce / retail
config = GeneratorConfig(
    order_qty_dist="zipf",
    order_qty_zipf_a=1.5,
    min_order_qty=1,
    max_order_qty=200,
)
```

### Increasing constraint density

```python
# High VMI ratio — more routing restrictions
config = GeneratorConfig(
    vmi_customer_ratio=0.50,
    min_products_per_plant=0.10,
    max_products_per_plant=0.30,
)
```

---

## Alternate Configuration Examples

The notebook's Section 8 includes ready-to-run configuration blocks for common use cases.

### Example A: Right-skewed quantities with gamma weights

Useful for manufacturing supply chains where most orders are small but occasional bulk orders dominate volume.

```python
config = GeneratorConfig(
    n_orders=5_000,
    n_customers=200,
    n_products=100,
    n_plants=10,
    n_ports=6,
    n_carriers=8,
    order_qty_dist="lognormal",
    order_qty_lognormal_mean=3.5,
    order_qty_lognormal_sigma=1.0,
    min_order_qty=1,
    max_order_qty=500,
    weight_per_unit_dist="gamma",
    weight_per_unit_gamma_shape=2.0,
    weight_per_unit_gamma_scale=1.5,
    wh_cost_dist="lognormal",
    service_level_probs=(0.60, 0.30, 0.10),
    carrier_mode_probs=(0.40, 0.40, 0.20),
    seed=99,
)
```

### Example B: High constraint density for feasibility stress testing

Useful for testing whether an optimizer handles dense VMI restrictions and narrow product coverage without producing infeasible routes.

```python
config = GeneratorConfig(
    n_orders=10_000,
    n_plants=20,
    n_products=200,
    vmi_customer_ratio=0.50,
    min_products_per_plant=0.10,
    max_products_per_plant=0.25,
    carrier_multimodal_prob=0.05,
    ports_per_carrier_range=(1, 2),
    seed=42,
)
```

---

## Validation Suite

After `gen.run()` completes, call `run_validation(tables, config)` to confirm all 19 checks pass. The function raises an `AssertionError` on the first failure, printing which check failed and why.

### What is checked

| Check | Table | What is verified |
|---|---|---|
| 1 | `OrderList` | Row count matches `n_orders` |
| 2 | `OrderList` | No nulls in critical columns |
| 3 | `OrderList` | All `Unit quantity` and `Weight` values are positive |
| 4 | `OrderList` | All order dates fall within configured range |
| 5 | `OrderList` | All `Service Level` values are in `{DTP, DTD, CRF}` |
| 6 | `OrderList` | All `Plant Code` values exist in `ProductsPerPlant` |
| 7 | `OrderList` | VMI customers are served only by VMI-designated plants |
| 8 | `OrderList` | All ordered products have at least one plant in `ProductsPerPlant` |
| 9 | `FreightRates` | No negative `rate` or `minimum cost` values |
| 10 | `FreightRates` | No duplicate weight-bracket entries per carrier-route-mode-service group |
| 11 | `FreightRates` | All `mode_dsc` values are in `{AIR, GROUND, SEA}` |
| 12 | `FreightRates` | All `tpt_day_cnt` values are non-negative |
| 13 | `WhCosts` / `WhCapacities` | Both tables have exactly `n_plants` rows |
| 14 | `WhCosts` / `WhCapacities` | All cost values are positive; all capacities meet the floor |
| 15 | `ProductsPerPlant` | Every product in the catalogue appears in at least one plant |
| 16 | `ProductsPerPlant` | No duplicate (Plant Code, Product ID) rows |
| 17 | `VmiCustomers` | All VMI plant codes exist in the configured plant set |
| 18 | `VmiCustomers` | All VMI customer IDs exist in the configured customer set |
| 19 | `PlantPorts` | Every plant has at least one port link |

### Expected output on success

```
All 19 validation checks passed.
    OrderList       :   10,000 rows
    FreightRates    :    1,610 rows
    WhCosts         :       20 rows
    WhCapacities    :       20 rows
    ProductsPerPlant:    1,932 rows
    VmiCustomers    :      113 rows
    PlantPorts      :       42 rows
```

---

## Limitations and Known Gaps

The following structural properties of the generated data diverge from real-world observations. They do not affect the dataset's suitability for optimization benchmarking, but should be understood before using it for predictive modelling or behavioural analysis.

**Demand seasonality.** Order dates are drawn from a uniform distribution, producing flat monthly volume with no seasonal peaks. Real supply chains exhibit monthly CVs of 15-40% for B2B and 40-80%+ for seasonal consumer goods. To add seasonality, replace the uniform date offset with a time-varying Poisson process or a seasonal mixture model.

**Warehouse economies of scale.** Warehouse cost per unit is drawn independently of warehouse capacity, so no cost-capacity correlation is produced. Real warehouse cost models predict that per-unit costs decrease with facility size due to fixed cost amortization. A correlated generation approach — sampling costs inversely proportional to capacity with additive noise — would correct this.

**Carrier service specialization.** The generator assigns service codes to modes (AIR: DTD/DTP; GROUND: DTP/CRF; SEA: DTD/DTP/CRF), but within each mode, service codes are sampled uniformly. Real carriers exhibit stronger specialization: express air couriers almost exclusively handle DTD, while sea carriers are predominantly DTP or CRF. A carrier-level specialization map would improve fidelity.

**Uniform weight brackets across carriers.** All carriers use the same weight bracket boundaries. In the Brunel dataset, bracket structures vary by carrier, reflecting different vehicle capacity profiles and pricing strategies. Per-carrier bracket sampling would improve the structural fidelity of `FreightRates`.

**Infeasibility under extreme parameter combinations.** Very high VMI ratios combined with very low product-plant coverage can cause the constraint-aware loop to fall back to the full plant set for some orders, which may violate a constraint. The generator does not currently guarantee dataset production for all valid configurations. Future versions should add a feasibility score that quantifies constraint satisfaction when soft targets cannot be fully met.

---

## References

anisseezzebdi (2023). Supply Chain Logistics Problem [Dataset]. Kaggle. https://www.kaggle.com/datasets/anisseezzebdi/supply-chain-logistics-problem

Baker, P., and Canessa, M. (2009). Warehouse design: A structured approach. *European Journal of Operational Research*, 193(2), 425-436.

Holguín-Veras, J., Jaller, M., Destro, L., Ban, X., Lawson, C., and Levinson, H. S. (2011). Freight generation, freight trip generation, and the perils of using constant trip rates. *Transportation Research Record*, 2224(1), 68-81.

Ivars-Dzalbs, A. (2019). Supply Chain Logistics Problem Dataset. Brunel University London. https://doi.org/10.17633/rd.brunel.7558679.v1

NumPy Developers (2023). numpy.random.default_rng — NumPy v1.26 Manual. https://numpy.org/doc/stable/reference/random/generator.html

---

## License

This project was developed as part of the Eskwelabs Innovation Fellowship internship program. All code and documentation are available for academic and research use.
