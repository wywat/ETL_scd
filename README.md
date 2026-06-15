# Semantic-Aware Physical Storage Model for Slowly Changing Dimensions

This repository contains the experimental scripts and summarized results for the study:

**A Semantic-Aware Physical Storage Model for Slowly Changing Dimensions in Big Data Warehouses**

The experiment evaluates a semantic-aware physical storage model for slowly changing dimensions (SCDs) and compares it with the conventional SCD Type 2 baseline. The main objective is to examine whether semantic grouping and attribute-aware historical maintenance can reduce storage redundancy, reduce written historical data volume, and preserve query compatibility in data warehouse environments.

## 1. Repository Structure

```text
scd-experiment/
├── README.md
├── requirements.txt
├── scripts/
│   ├── 01_convert_tpcds_to_parquet.py
│   ├── 02_build_base_tables.py
│   ├── 03_generate_scd_events.py
│   ├── 04_build_scd2_baseline.py
│   ├── 05_build_proposed_model.py
│   ├── 06_run_benchmarks.py
│   └── 07_plot_figures.py
└── results/
    ├── experiment_results.csv
    └── figures/
        ├── figure3_storage_size.png
        ├── figure4_storage_reduction.png
        ├── figure5_update_time.png
        ├── figure6_written_volume.png
        └── figure7_query_time.png
```

The large experimental data files are not stored in this GitHub repository. They are available from Zenodo:

```text
https://sandbox.zenodo.org/uploads/513446
```

## 2. Data Organization

The full experimental data are organized as follows:

```text
data/
├── raw/
│   └── tpcds_sf10/
├── base/
├── events/
│   ├── rate_0_001/
│   ├── rate_0_005/
│   ├── rate_0_01/
│   └── rate_0_05/
├── baseline_scd2/
│   ├── rate_0_001/
│   ├── rate_0_005/
│   ├── rate_0_01/
│   └── rate_0_05/
├── proposed/
│   ├── rate_0_001/
│   ├── rate_0_005/
│   ├── rate_0_01/
│   └── rate_0_05/
└── results/
    ├── experiment_results.csv
    └── figures/
```

### 2.1 `data/raw/`

This directory contains the original TPC-DS source data files. In this experiment, the raw dataset is stored under:

```text
data/raw/tpcds_sf10/
```

The raw files are converted into Parquet format before further processing.

### 2.2 `data/base/`

This directory contains the base tables constructed from the raw TPC-DS data. The main generated tables include:

* `customer_base.parquet`: the base customer dimension table used to construct SCD records;
* `sales_fact_10m.parquet`: the fact table used for analytical query evaluation;
* `item_dim.parquet`: the item dimension table;
* `store_dim.parquet`: the store dimension table;
* `date_dim.parquet`: the date dimension table.

These base tables provide the common input for both the SCD Type 2 baseline and the proposed semantic-aware model.

### 2.3 `data/events/`

This directory contains simulated SCD change events under different daily change rates.

The four change-rate settings are:

| Directory    | Daily change rate |
| ------------ | ----------------: |
| `rate_0_001` |              0.1% |
| `rate_0_005` |              0.5% |
| `rate_0_01`  |                1% |
| `rate_0_05`  |                5% |

Each directory contains a `scd_change_events.parquet` file. The file records simulated customer attribute changes, including the changed customer, changed attribute, new value, change date, semantic category, and attribute group.

### 2.4 `data/baseline_scd2/`

This directory contains the generated results of the conventional SCD Type 2 baseline.

For each change-rate setting, the baseline constructs a full-row historical version whenever a tracked customer attribute changes. The main output is:

* `customer_scd2.parquet`: the SCD Type 2 historical customer dimension table;
* `update_metrics.json`: update and storage metrics for the corresponding change-rate setting.

### 2.5 `data/proposed/`

This directory contains the generated results of the proposed semantic-aware physical storage model.

For each change-rate setting, the proposed model organizes customer attributes according to semantic groups and maintains historical records using semantic-aware storage structures. The main outputs include:

* current-state customer tables;
* semantic-grouped historical dimension tables;
* metadata or reconstruction-related files;
* `update_metrics.json`: update and storage metrics for the corresponding change-rate setting.

Compared with SCD Type 2, the proposed model aims to avoid redundant full-row historical versioning and reduce written historical data volume.

### 2.6 `data/results/`

This directory contains the summarized experimental results.

The main file is:

* `experiment_results.csv`: summarized benchmark results, including storage size, storage reduction, update time, written historical data volume, and query execution time.

The figures used in the paper are generated from this CSV file.

## 3. Experimental Workflow

The experiment contains seven main steps.

### Step 1: Convert TPC-DS raw files to Parquet

```bash
python3 scripts/01_convert_tpcds_to_parquet.py \
  --input_dir data/raw/tpcds_sf10 \
  --output_dir data/raw_parquet
```

### Step 2: Build base tables

```bash
python3 scripts/02_build_base_tables.py \
  --input_dir data/raw_parquet \
  --output_dir data/base \
  --customers 500000 \
  --items 50000 \
  --facts 10000000
```

### Step 3: Generate SCD change events

```bash
python3 scripts/03_generate_scd_events.py \
  --customer_base data/base/customer_base.parquet \
  --output_dir data/events \
  --days 180 \
  --rates 0.001 0.005 0.01 0.05
```

### Step 4: Build the SCD Type 2 baseline

```bash
python3 scripts/04_build_scd2_baseline.py \
  --customer_base data/base/customer_base.parquet \
  --events data/events/rate_0_001/scd_change_events.parquet \
  --output_dir data/baseline_scd2/rate_0_001
```

The same command should be executed for all four change-rate settings:

```text
rate_0_001
rate_0_005
rate_0_01
rate_0_05
```

### Step 5: Build the proposed semantic-aware model

```bash
python3 scripts/05_build_proposed_model.py \
  --customer_base data/base/customer_base.parquet \
  --events data/events/rate_0_001/scd_change_events.parquet \
  --output_dir data/proposed/rate_0_001
```

The same command should be executed for all four change-rate settings.

### Step 6: Run benchmark experiments

```bash
python3 scripts/06_run_benchmarks.py \
  --base_dir data/base \
  --baseline_dir data/baseline_scd2 \
  --proposed_dir data/proposed \
  --events_dir data/events \
  --output data/results/experiment_results.csv \
  --repeat 5
```

### Step 7: Plot figures

```bash
python3 scripts/07_plot_figures.py \
  --input data/results/experiment_results.csv \
  --output_dir data/results/figures
```

## 4. Evaluation Metrics

The experiment evaluates the following metrics.

| Metric                               | Description                                                                                      |
| ------------------------------------ | ------------------------------------------------------------------------------------------------ |
| Storage size                         | Final storage size of the generated dimension storage structures under each change-rate setting. |
| Storage reduction                    | Relative storage reduction of the proposed model compared with SCD Type 2.                       |
| Written historical data volume       | Historical data volume written during SCD maintenance.                                           |
| Update time                          | Batch construction or update time measured during model generation.                              |
| Current-state query time             | Query time for retrieving active/current dimension records.                                      |
| Historical point-in-time query time  | Query time for reconstructing dimension states at a given historical time point.                 |
| Fact-dimension analytical query time | Query time for analytical queries involving the fact table and historical dimension states.      |

## 5. Summary of Results

The summarized experimental results are provided in:

```text
results/experiment_results.csv
```

The generated figures are located in:

```text
results/figures/
```

The results show that the proposed semantic-aware model reduces storage size under medium and high change rates and significantly reduces written historical data volume across all change-rate settings. The proposed model also preserves query compatibility and improves query execution time in the current experimental setup. However, the batch construction time of the proposed model may be higher because additional semantic grouping, current-state materialization, and validity-interval maintenance are required.

## 6. Requirements

The scripts were developed and tested using Python 3. The main dependencies include:

```text
pandas
duckdb
pyarrow
numpy
matplotlib
tqdm
```

They can be installed using:

```bash
pip install pandas duckdb pyarrow numpy matplotlib tqdm
```

Alternatively, install all dependencies using:

```bash
pip install -r requirements.txt
```

## 7. Reproducibility Notes

Due to the large size of the raw and generated experimental data, only scripts, summarized results, and figures are included in this GitHub repository. The full data files are archived separately on Zenodo.

To reproduce the experiment from scratch, download the data archive from Zenodo and place the `data/` directory under the root of this repository:

```text
scd-experiment/
├── data/
├── scripts/
├── results/
└── README.md
```

Then run the scripts in numerical order from `01` to `07`.

## 8. Citation

If you use this repository or the associated dataset, please cite the corresponding paper and Zenodo archive.

```text
Citation information will be added after publication.
```

