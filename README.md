# Presto Benchmarking for TPC-H and TPC-DS Data

![Presto Benchmarking Architecture](./architecture.png)

This repository provides a standardized deployment for Presto (Java) optimized for executing TPC-H and TPC-DS benchmarks on Iceberg data lakes stored in Google Cloud Storage (GCS).

## Data Preparation Workflow

The datasets in this environment were prepared using a three-layer "Raw-to-Gold" architecture:

1.  **Raw Generation**: TPC-H and TPC-DS datasets were generated at Scale Factor 10 using standard generators (`tpch-gen` and `tpcds-gen`).
2.  **Schema Mapping (Raw Layer)**: External Hive tables were created to map the raw `.dat` files on GCS. This layer handles CSV parsing and trailing delimiters.
3.  **Optimization (Gold Layer)**: Data was migrated into managed Iceberg tables using `INSERT` statements with explicit type casting. This ensures the Parquet files are optimized for analytical performance.
4.  **Statistics Collection**: Immediately following migration, `ANALYZE` was performed on all tables to enable Cost-Based Optimization (CBO) in Presto.

## System Architecture

The environment is orchestrated via Docker Compose and consists of the following components:

*   **Presto Coordinator**: Manages query orchestration and metadata.
*   **Presto Worker**: Java-based execution node with 12GB heap and spilling-to-disk enabled.
*   **Hive Metastore**: Thrift-based metadata management service.
*   **PostgreSQL 15**: Dedicated backend database for the Hive Metastore.
*   **MySQL 8.0**: Persistent backend for storing pbench benchmark result summaries.
*   **Grafana**: Visualization dashboard for performance analysis.

## Service Endpoints

| Service | Host Port | Internal Port | Description |
| :--- | :--- | :--- | :--- |
| Presto UI | 8080 | 8080 | Coordinator status and query monitoring |
| Grafana | 3000 | 3000 | Benchmarking dashboards (admin/admin) |
| MySQL | 3307 | 3306 | Metrics database (pbench/pbench_password) |
| Metastore | 9083 | 9083 | Thrift Metastore endpoint |

## Prerequisites

*   **Docker & Docker Compose**: Minimum 16GB RAM allocation recommended.
*   **GCS Access**: A Google Cloud service account key with storage access. Place the file as `gcs-key.json` in the root directory.
*   **PBench Binary**: The included `pbench` binary is compiled for **macOS (Darwin ARM64)**. Linux users must replace this with a Linux-compatible pbench binary.
*   **Iceberg Data**: TPC-H and TPC-DS datasets previously generated and stored in an Iceberg/GCS bucket.

## Deployment Instructions

### 1. Initialize the Cluster
```bash
docker-compose up -d
```
Ensure all services reach a `healthy` state before proceeding.

### 2. Schema and Table Initialization
If your Iceberg data already exists on GCS, you must register the schema and tables in the Metastore:

```sql
-- Connect via presto-cli
docker exec -it presto-coordinator presto-cli

-- Create the target schema
CREATE SCHEMA IF NOT EXISTS iceberg.tpcds_sf10;

-- Register an existing Iceberg table
CALL iceberg.system.register_table(
  schema_name => 'tpcds_sf10',
  table_name => 'customer',
  table_location => 'gs://your-bucket-name/path/to/tpcds_sf10/customer'
);
```

### 3. Execute Benchmarks
Navigate to the `pbench` directory and execute the suite:
```bash
cd pbench
./pbench run --mysql mysql.json tpch_sf10_iceberg.json
```

## Configuration Specifications

### Memory Management
The worker node is configured for high-concurrency analytical queries:
*   **JVM Heap**: 12GB (`-Xmx12G`)
*   **Query Max Memory**: 6GB (User memory per node)
*   **Disk Spilling**: Enabled via `/tmp/presto-spill` (mounted to the host) to handle joins that exceed physical memory.

### GCS Connectivity
The GCS connector is configured in `hive-metastore/core-site.xml`. Ensure the `fs.gs.impl` and authentication properties match your environment requirements.

## Visualization
Benchmark results are streamed in real-time to Grafana:
[http://localhost:3000/d/benchmarking-dashboard](http://localhost:3000/d/benchmarking-dashboard)

![TPC-H Results](./TPC-H%20Grafana%20Dashboard.png)
![TPC-DS Results](./TPC-DS%20Grafana%20Dashboard.png)
