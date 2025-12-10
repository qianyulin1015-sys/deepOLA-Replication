# DeepOLA (SIGMOD '23) Replication Experiment

This repository contains the reproduction scripts, experimental results, and custom fixes for the **TPC-H Benchmark** experiments from the paper *"A Step Toward Deep Online Aggregation"* (SIGMOD 2023).

We evaluate and compare the performance of three systems:
1.  **Wake (DeepOLA)**: An online aggregation engine.
2.  **Polars**: A high-performance in-memory DataFrame library.
3.  **PostgreSQL**: A traditional disk-based relational database (Baseline).

---

## 1. Project Structure

The repository is organized as follows. Note the **custom utility scripts** (`.py` files) added to resolve data formatting and container stability issues encountered during replication.

```text
DeepOLA/
├── baselines/                  # Scripts for baseline systems (Postgres & Polars)
├── scripts/                    # DeepOLA (Wake) scripts and visualization tools
│   ├── experiment_wake_tpch.sh
│   ├── plot_tpch.py
│   └── ...
├── results/                    # Output directory for Logs, CSVs, and Plots
│   ├── wake/
│   ├── polars/
│   ├── postgres/
│   └── viz/                    # Final Figures (fig7_tpch.png, etc.)
├── experiment/                 # Generated TPC-H dataset
├── fix_postgres.py             # [CUSTOM] Parses raw Postgres logs into clean CSVs
└── clean_polars_disk.py        # [CUSTOM] Cleans Polars CSVs (fixes NaN/data type errors)
````

-----

## 2\. Dependencies

Ensure your environment meets the following requirements before running the experiments:

  * **Operating System**: Linux (Ubuntu 20.04/22.04 recommended).
  * **Docker**: Required for running isolated system environments.
      * *Note:* Ensure your user is added to the `docker` group to run commands without sudo.
  * **Python 3.8+**: Required for data cleaning and plotting.
      * Dependencies: `pandas`, `matplotlib`
      * Install: `pip install pandas matplotlib`
  * **Hardware**:
      * RAM: 16GB+ recommended (for in-memory engines).
      * Disk: 20GB+ free space.

-----

## 3\. How to Run

Follow these steps sequentially to reproduce the results.

### Step 0: Environment Setup

Export the environment variables used by the scripts.

```bash
export SCALE=10
export PARTITION=10
export DATA_DIR=$(pwd)/experiment/dataset
export QUERY_DIR=$(pwd)/resources/tpc-h/queries
```

### Step 1: Data Generation

Generate the TPC-H dataset (Scale 10).

```bash
./data-gen.sh ${DATA_DIR} ${SCALE} ${PARTITION}
```

### Step 2: Run PostgreSQL (Baseline)

> **Note:** The original scripts had issues with container lifecycle management (killing containers prematurely). We use a **manual start approach** for stability.

1.  **Start the container manually:**

    ```bash
    export POSTGRES_DIR=$(pwd)/tmp/postgres/scale=${SCALE}/partition=${PARTITION}
    mkdir -p ${POSTGRES_DIR}

    docker run --name pgsql \
        -v ${POSTGRES_DIR}:/var/lib/postgresql/data \
        -v ${DATA_DIR}:/data:ro \
        -v ${QUERY_DIR}:/dataset/tpch/queries:ro \
        -e POSTGRES_PASSWORD=postgres \
        -d postgres:13
    ```

2.  **Load data and run benchmark:**

    ```bash
    # Setup tables and load data
    ./baselines/postgres/experiment-setup.sh ${DATA_DIR} ${QUERY_DIR} ${POSTGRES_DIR} ${SCALE} ${PARTITION}

    # Run queries (10 runs)
    export OUTPUT_DIR=$(pwd)/results/postgres/scale=${SCALE}/
    ./baselines/postgres/experiment-time.sh ${QUERY_DIR} ${OUTPUT_DIR} ${POSTGRES_DIR} ${SCALE} ${PARTITION} 10 0 1 22
    ```

3.  **Process Logs (Custom Fix):**
    Run the custom script to parse raw Postgres logs (`.log`) into a structured CSV format required for plotting.

    ```bash
    python3 fix_postgres.py
    ```

### Step 3: Run Polars

Run the Polars experiment using the provided Docker image.

1.  **Execute Benchmark:**

    ```bash
    docker run --rm \
        -v ${DATA_DIR}:/dataset/tpch:rw \
        -v $(pwd)/results/polars:/results/polars \
        --name polars deepola-polars:sigmod2023 \
        bash experiment.sh /dataset/tpch /results/polars ${SCALE} ${PARTITION} 10 0 1 22
    ```

2.  **Clean Data (Custom Fix):**
    Run the custom script to fix non-numeric data type issues and hidden characters in Polars output CSVs.

    ```bash
    python3 clean_polars_disk.py
    ```

### Step 4: Run Wake (DeepOLA)

Run the Wake engine experiment.

```bash
docker run --rm \
    -v ${DATA_DIR}:/dataset:rw \
    -v $(pwd)/results/wake:/saved-outputs:rw \
    --name wake deepola-wake:sigmod2023 \
    bash scripts/experiment_wake_tpch.sh /dataset ${SCALE} ${PARTITION} 10 0 1 22
```

### Step 5: Visualization

Generate the comparison plots. We mount local scripts to the container to ensure all fixes are applied during plotting.

```bash
# Get container working directory
WORK_DIR=$(docker run --rm deepola-viz:sigmod2023 pwd | tr -d '\r')

# Run plotting script
docker run --rm \
    -v $(pwd)/results/wake:/results/wake:rw \
    -v $(pwd)/results/polars:/results/polars:rw \
    -v $(pwd)/results/postgres:/results/postgres:rw \
    -v $(pwd)/results/viz:/results/viz:rw \
    -v $(pwd)/scripts:${WORK_DIR}/scripts:rw \
    --name viz deepola-viz:sigmod2023 \
    python3 scripts/plot_tpch.py ${SCALE} ${PARTITION} 10
```

-----

## 4\. Results

After running Step 5, the following charts will be available in the `results/viz/` directory:

  * **`fig7_tpch.png`**: Total execution time comparison (Wake vs. Polars vs. Postgres).
  * **`fig8_tpch_error.png`**: Relative error convergence over time for Wake.

-----

## 5\. Troubleshooting & Custom Fixes

This replication includes several stability fixes developed during the reproduction process:

1.  **Postgres Stability**: Switched to persistent container mode to avoid `postmaster.pid` lock errors caused by original scripts aggressively stopping containers.
2.  **Data Cleaning**: Added `clean_polars_disk.py` to handle `NaN` values and string formatting issues in Polars output that originally caused `DataError: No numeric types`.
3.  **Log Parsing**: Added `fix_postgres.py` to correctly extract timing data from raw Postgres logs, as the original extraction script failed to locate the output files.

-----

*Replication performed on [Current Date]. Original code by [illinoisdata/DeepOLA](https://github.com/illinoisdata/DeepOLA).*

```
```
