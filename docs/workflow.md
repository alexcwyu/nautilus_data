# Workflows

This document describes the major data workflows in `nautilus_data`, from raw data acquisition through catalog generation and benchmarking.

## Data Ingestion Flow

The primary workflow downloads raw FX tick data and transforms it into a NautilusTrader-compatible Parquet catalog. This is orchestrated by the `main()` function in `src/nautilus_data/hist_data_to_catalog.py`.

```mermaid
sequenceDiagram
    participant User
    participant Main as main()
    participant DL as download()
    participant GitHub as GitHub Raw
    participant Load as load_fx_hist_data()
    participant TIP as TestInstrumentProvider
    participant CSV as CSVTickDataLoader
    participant Wrangler as QuoteTickDataWrangler
    participant Catalog as ParquetDataCatalog

    User->>Main: python -m nautilus_data.hist_data_to_catalog
    Main->>DL: download(github_url)
    DL->>GitHub: HTTP GET .csv.gz
    GitHub-->>DL: Compressed CSV bytes
    DL->>DL: Write to local file

    Main->>Load: load_fx_hist_data(filename, "EUR/USD", catalog_path)
    Load->>TIP: default_fx_ccy("EUR/USD")
    TIP-->>Load: Instrument object

    Load->>Wrangler: QuoteTickDataWrangler(instrument)
    Load->>CSV: load(filename, index_col=0, datetime_format="%Y%m%d %H%M%S%f")
    CSV-->>Load: DataFrame[timestamp, col1, col2, col3]
    Load->>Load: Rename columns to [bid_price, ask_price, size]
    Load->>Wrangler: process(df)
    Wrangler-->>Load: List[QuoteTick]

    Load->>Catalog: ParquetDataCatalog(catalog_path)
    Load->>Catalog: write_data([instrument])
    Load->>Catalog: write_data(ticks)
    Catalog-->>Load: Parquet files written
```

### Step-by-Step

1. **Download**: `download()` fetches the gzip-compressed CSV from GitHub's raw content URL. The file is written to the current working directory with the filename extracted from the URL.

2. **Instrument Resolution**: `TestInstrumentProvider.default_fx_ccy("EUR/USD")` creates a fully specified FX instrument with venue, price precision, size precision, and other trading parameters.

3. **CSV Loading**: `CSVTickDataLoader.load()` reads the compressed CSV with:
   - Column 0 as the index (timestamps)
   - Datetime format: `%Y%m%d %H%M%S%f` (e.g., `20200102 170000123456`)

4. **Column Mapping**: The raw DataFrame columns are renamed to `["bid_price", "ask_price", "size"]` to match the wrangler's expected input format.

5. **Wrangling**: `QuoteTickDataWrangler.process()` converts each row into a `QuoteTick` object with proper fixed-point price encoding and nanosecond timestamps.

6. **Catalog Write**: Both the instrument definition and tick data are written separately to the `ParquetDataCatalog`.

## Custom Data Loading Flow

For loading data from local files or different currency pairs, the `load_fx_hist_data()` function accepts custom parameters.

```mermaid
flowchart TD
    START["Start"] --> INPUT["Input:<br/>filename, currency, catalog_path"]
    INPUT --> RESOLVE["Resolve instrument<br/>TestInstrumentProvider.default_fx_ccy(currency)"]
    RESOLVE --> CREATE_WRANGLER["Create QuoteTickDataWrangler<br/>bound to instrument"]
    CREATE_WRANGLER --> LOAD_CSV["Load CSV via CSVTickDataLoader<br/>- index_col=0<br/>- datetime_format=%Y%m%d %H%M%S%f"]
    LOAD_CSV --> RENAME["Rename columns:<br/>bid_price, ask_price, size"]
    RENAME --> PROCESS["Wrangle DataFrame to QuoteTick list"]
    PROCESS --> OPEN_CATALOG["Open ParquetDataCatalog(catalog_path)"]
    OPEN_CATALOG --> WRITE_INST["Write instrument definition"]
    WRITE_INST --> WRITE_TICKS["Write tick data"]
    WRITE_TICKS --> DONE["Done"]

    LOAD_CSV -->|File not found| ERROR["FileNotFoundError"]
    RESOLVE -->|Unknown currency| ERROR2["Instrument resolution error"]
```

### Usage Example

```python
from nautilus_data.hist_data_to_catalog import load_fx_hist_data

# Load GBP/USD data from a local file
load_fx_hist_data(
    filename="/path/to/raw_data/fx_hist_data/DAT_ASCII_GBPUSD_T_202001.csv.gz",
    currency="GBP/USD",
    catalog_path="/home/user/my_catalog",
)
```

## Benchmark Data Preparation Flow

The benchmark utilities in `bench_data/` provide a workflow for preparing and validating test datasets.

```mermaid
flowchart TD
    subgraph Preparation["1. Prepare Benchmark Data"]
        SOURCE["Source Parquet files"] --> EXTRACT_GRP["extract_groups.py<br/>Re-partition with custom<br/>row group size (e.g., 5000)"]
        EXTRACT_GRP --> OUTPUT["Re-partitioned Parquet<br/>bench_data/multi_stream_data/"]
    end

    subgraph Validation["2. Validate Data Integrity"]
        OUTPUT --> EXTRACT_TS["extract_ts_init.py<br/>Extract ts_init boundaries<br/>per row group"]
        EXTRACT_TS --> STATS_CSV["CSV: index, start_ts,<br/>end_ts, group_size"]
        STATS_CSV --> CHECK["check_invariant.py<br/>Verify monotonic ordering<br/>and non-overlapping groups"]
        CHECK -->|Pass| VALID["Data Valid"]
        CHECK -->|Fail| INVALID["Report violations"]
    end

    subgraph Statistics["3. Generate Statistics"]
        OUTPUT --> GEN_STATS["gen_data_stats.py<br/>Walk directory tree"]
        GEN_STATS --> STATS_OUT["stats.csv:<br/>file_name, file_size_kb,<br/>total_rows, max_row_group_size"]
    end
```

### Row Group Re-Partitioning

The `extract_groups.py` script re-writes a Parquet file with a specified number of rows per row group:

```bash
python bench_data/extract_groups.py input.parquet output.parquet 5000
```

This uses a streaming write pattern:
1. Read the full DataFrame from the input file
2. Determine the appropriate schema (quote or trade) based on filename
3. Open a `ParquetWriter` with Snappy compression
4. Write batches of `rows_per_row_group` rows as individual row groups
5. Close the writer

### Timestamp Boundary Extraction

The `extract_ts_init.py` script reads each row group and records the first and last `ts_init` values:

```bash
python bench_data/extract_ts_init.py data.parquet boundaries.csv
```

Output format:
```csv
index,start_ts,end_ts,group_size
0,1667260800000000000,1667261234567890000,5000
1,1667261234567890001,1667262000000000000,5000
```

### Invariant Checking

The `check_invariant.py` script validates two properties on the boundary CSV:

1. **Monotonic `start_ts`**: All row groups have strictly increasing start timestamps
2. **Non-overlapping boundaries**: `end_ts[i] <= start_ts[i+1]` for consecutive row groups

```bash
python bench_data/check_invariant.py boundaries.csv
```

### Statistics Generation

The `gen_data_stats.py` script walks a directory tree and records per-file statistics:

```bash
python bench_data/gen_data_stats.py bench_data/ bench_data/stats.csv
```

Output includes file size, total row count, and maximum row group size for each Parquet file found.

## Docker Build Flow

The Dockerfile automates the entire data pipeline in a reproducible container build.

```mermaid
sequenceDiagram
    participant Docker as Docker Build
    participant Base as python:3.13-slim
    participant Builder as Builder Stage
    participant App as Application Stage

    Docker->>Base: FROM python:3.13-slim AS base
    Base->>Base: Set PYTHONUNBUFFERED, WORKDIR

    Docker->>Builder: FROM base AS builder
    Builder->>Builder: apt-get install curl clang gcc git libssl-dev make
    Builder->>Builder: Install uv (version from uv-version file)
    Builder->>Builder: COPY . .
    Builder->>Builder: uv pip install . --system

    Docker->>App: FROM base AS application
    App->>App: Set CATALOG_PATH=/catalog
    App->>App: COPY site-packages from builder
    App->>App: COPY nautilus_data source
    App->>App: mkdir -p /opt/pysetup/catalog/backtest
    App->>App: RUN python -m nautilus_data.hist_data_to_catalog
    Note over App: Catalog is pre-generated<br/>in the final image
```

Key aspects of the Docker workflow:
- **Multi-stage build** keeps the final image small (no build tools)
- **Catalog pre-generation** at build time means the image is ready to use immediately
- **uv version pinning** via the `uv-version` file ensures reproducible builds

## End-to-End Backtesting Data Workflow

Combining all components, here is the full workflow from raw data to backtesting:

```mermaid
flowchart LR
    subgraph Acquire["Acquire"]
        DL["Download from<br/>GitHub/local"]
    end

    subgraph Transform["Transform"]
        LOAD["CSV Load +<br/>Column Rename"]
        WRANGLE["Wrangle to<br/>QuoteTick objects"]
    end

    subgraph Store["Store"]
        CATALOG["Write to<br/>ParquetDataCatalog"]
    end

    subgraph Validate["Validate"]
        CHECK["Check invariants"]
        STATS["Generate stats"]
    end

    subgraph Consume["Consume"]
        NT["NautilusTrader<br/>Backtesting Engine"]
    end

    DL --> LOAD --> WRANGLE --> CATALOG --> CHECK
    CATALOG --> STATS
    CATALOG --> NT
```

## File Reference

| File | Role in Workflow |
|------|-----------------|
| `src/nautilus_data/hist_data_to_catalog.py` | Core pipeline: download, load, wrangle, catalog |
| `bench_data/extract_groups.py` | Row group re-partitioning for benchmark prep |
| `bench_data/extract_ts_init.py` | Row group boundary extraction |
| `bench_data/check_invariant.py` | Timestamp ordering validation |
| `bench_data/gen_data_stats.py` | Parquet file statistics generation |
| `bench_data/stats.csv` | Pre-computed statistics for benchmark data |
| `raw_data/fx_hist_data/DAT_ASCII_EURUSD_T_202001.csv.gz` | Source EUR/USD tick data |
| `Dockerfile` | Containerized pipeline with pre-built catalog |

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
