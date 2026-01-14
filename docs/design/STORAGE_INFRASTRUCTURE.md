# Storage Infrastructure Design

**Scalable Storage for Traces, Metrics, and ML Features**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Storage Requirements](#storage-requirements)
3. [Architecture](#architecture)
4. [Schema Design](#schema-design)
5. [Data Lifecycle](#data-lifecycle)
6. [Query Patterns](#query-patterns)
7. [Scalability](#scalability)
8. [Implementation Options](#implementation-options)

---

## Overview

The storage infrastructure must handle multiple data types with different characteristics:

| Data Type | Volume | Query Pattern | Retention |
|-----------|--------|---------------|-----------|
| **Raw Traces** | High (10s of GB/day) | Rare, by ID | Short (30 days) |
| **Aggregated Stats** | Medium (GBs/day) | Frequent, analytics | Long (1+ years) |
| **Metrics Time-Series** | High (millions/day) | Range queries | Medium (90 days) |
| **ML Features** | Medium (GBs/day) | Batch for training | Long (permanent) |
| **Metadata Catalog** | Low (MBs/day) | Frequent lookups | Long (permanent) |

### Design Goals

1. **Efficient ingestion** - Handle burst of data from CI runs
2. **Fast queries** - Sub-second for common queries
3. **Cost-effective** - Tiered storage for different data ages
4. **ML-ready** - Easy to export for model training
5. **Standard formats** - Queryable with standard tools

---

## Storage Requirements

### Capacity Estimation

```
Per CI Run (daily):
├── 10 SoCs
├── 50 models per SoC
├── 1000 frames per model (100 sampled with full trace)
├── 100 operators per model
│
├── Raw Traces (sampled 10%):
│   └── 10 × 50 × 100 frames × 50KB = 2.5 GB
│
├── Aggregated Stats (all frames):
│   └── 10 × 50 × 1000 frames × 1KB = 500 MB
│
├── Metrics Time-Series:
│   └── 10 SoCs × 100 metrics × 10000 samples × 20B = 200 MB
│
├── ML Feature Vectors:
│   └── 10 × 50 × 1000 frames × 500B = 250 MB
│
└── Total per run: ~3.5 GB

Per Month (30 runs):
└── ~100 GB
```

### Query Requirements

| Query Type | Latency Target | Frequency |
|------------|---------------|-----------|
| Get trace by ID | < 100ms | Low |
| List runs by date | < 500ms | Medium |
| Aggregate latency by model | < 1s | High |
| Compare SoC performance | < 2s | High |
| Export ML features | < 1 min | Low (batch) |
| Find regressions | < 5s | High |

---

## Architecture

### High-Level Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Storage Architecture                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         Ingestion Layer                                 │ │
│  │                                                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │ │
│  │  │ Trace        │  │ Metric       │  │ Metadata                     │  │ │
│  │  │ Receiver     │  │ Receiver     │  │ Receiver                     │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │ │
│  │         │                 │                          │                  │ │
│  │         ▼                 ▼                          ▼                  │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │                     Write Buffer / Queue                          │  │ │
│  │  │                     (Kafka / Redis)                               │  │ │
│  │  └──────────────────────────────┬───────────────────────────────────┘  │ │
│  │                                 │                                       │ │
│  └─────────────────────────────────┼───────────────────────────────────────┘ │
│                                    ▼                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         Processing Layer                                │ │
│  │                                                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │ │
│  │  │ Trace        │  │ Aggregation  │  │ Feature                      │  │ │
│  │  │ Processor    │  │ Engine       │  │ Extractor                    │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │ │
│  │         │                 │                          │                  │ │
│  └─────────┼─────────────────┼──────────────────────────┼──────────────────┘ │
│            │                 │                          │                    │
│            ▼                 ▼                          ▼                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         Storage Layer                                   │ │
│  │                                                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │ │
│  │  │ Object Store │  │ Time-Series  │  │ Relational   │  │ Vector DB  │  │ │
│  │  │ (S3/MinIO)   │  │ DB           │  │ DB           │  │ (optional) │  │ │
│  │  │              │  │ (TimescaleDB │  │ (PostgreSQL) │  │            │  │ │
│  │  │ Raw traces   │  │  /InfluxDB)  │  │              │  │ Embeddings │  │ │
│  │  │ Parquet files│  │              │  │ Metadata     │  │ for search │  │ │
│  │  │              │  │ Metrics      │  │ Catalog      │  │            │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │ │
│  │                                                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         Query Layer                                     │ │
│  │                                                                         │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │                     Query Router / API                            │  │ │
│  │  │                                                                   │  │ │
│  │  │  /traces/* → Object Store    /metrics/* → Time-Series DB         │  │ │
│  │  │  /runs/* → Relational DB     /search/* → Vector DB               │  │ │
│  │  │  /ml/* → Parquet files       /aggregate/* → Pre-computed         │  │ │
│  │  │                                                                   │  │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Storage Components

| Component | Technology Options | Purpose |
|-----------|-------------------|---------|
| **Object Store** | S3, MinIO, GCS | Raw traces, Parquet files |
| **Time-Series DB** | TimescaleDB, InfluxDB, Prometheus | Metrics, counters |
| **Relational DB** | PostgreSQL | Metadata catalog, aggregates |
| **Vector DB** | Milvus, Pinecone, pgvector | ML embeddings (optional) |
| **Cache** | Redis | Hot data, recent queries |

---

## Schema Design

### Object Store Layout

```
s3://exo-traces/
├── raw/
│   ├── runs/
│   │   └── {run_id}/
│   │       ├── manifest.json
│   │       └── socs/
│   │           └── {soc_id}/
│   │               └── models/
│   │                   └── {model_id}/
│   │                       ├── traces/
│   │                       │   ├── frame_00000.perfetto
│   │                       │   ├── frame_00001.perfetto
│   │                       │   └── ...
│   │                       └── metrics/
│   │                           └── metrics.parquet
│   │
├── aggregated/
│   └── runs/
│       └── {run_id}/
│           ├── summary.parquet
│           ├── operator_stats.parquet
│           └── per_frame_stats.parquet
│
├── features/
│   └── {run_id}/
│       ├── model_features.parquet
│       ├── operator_features.parquet
│       └── graph_embeddings.npy
│
└── models/
    └── {model_type}/
        └── {version}/
            ├── model.pkl
            └── metadata.json
```

### Relational Schema (PostgreSQL)

```sql
-- ============================================================================
-- Core Entities
-- ============================================================================

CREATE TABLE benchmark_runs (
    run_id VARCHAR(100) PRIMARY KEY,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'running',
    
    -- Source info
    git_commit VARCHAR(40),
    git_branch VARCHAR(255),
    git_repo VARCHAR(255),
    
    -- Configuration
    config JSONB,
    
    -- Storage paths
    raw_trace_path VARCHAR(500),
    aggregated_path VARCHAR(500),
    feature_path VARCHAR(500),
    
    -- Summary stats (denormalized for fast queries)
    total_socs INT,
    total_models INT,
    total_frames BIGINT,
    
    -- Indexes
    INDEX idx_runs_created (created_at),
    INDEX idx_runs_git_commit (git_commit),
    INDEX idx_runs_status (status)
);

CREATE TABLE socs (
    soc_id VARCHAR(100) PRIMARY KEY,
    soc_name VARCHAR(255) NOT NULL,
    vendor VARCHAR(255),
    generation INT,
    
    -- Hardware specs
    specs JSONB,
    
    -- Capabilities
    available_counters JSONB,
    
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE soc_executions (
    soc_exec_id VARCHAR(200) PRIMARY KEY,  -- "{run_id}/{soc_id}"
    run_id VARCHAR(100) REFERENCES benchmark_runs(run_id),
    soc_id VARCHAR(100) REFERENCES socs(soc_id),
    
    firmware_version VARCHAR(100),
    hardware_config JSONB,
    
    -- Summary
    total_models INT,
    total_frames BIGINT,
    avg_latency_ms FLOAT,
    
    -- Paths
    trace_path VARCHAR(500),
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_soc_exec_run (run_id),
    INDEX idx_soc_exec_soc (soc_id)
);

CREATE TABLE models (
    model_id VARCHAR(100) PRIMARY KEY,
    model_name VARCHAR(255) NOT NULL,
    model_format VARCHAR(50),      -- tflite, onnx, etc.
    architecture VARCHAR(100),      -- resnet, transformer, etc.
    
    -- Model characteristics
    parameter_count BIGINT,
    total_flops BIGINT,
    input_shape JSONB,
    output_shape JSONB,
    operator_graph JSONB,
    
    -- Metadata
    version VARCHAR(50),
    source_url VARCHAR(500),
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_models_arch (architecture),
    INDEX idx_models_name (model_name)
);

CREATE TABLE model_runs (
    model_run_id VARCHAR(300) PRIMARY KEY,  -- "{soc_exec_id}/{model_id}"
    soc_exec_id VARCHAR(200) REFERENCES soc_executions(soc_exec_id),
    model_id VARCHAR(100) REFERENCES models(model_id),
    run_id VARCHAR(100) REFERENCES benchmark_runs(run_id),
    
    -- Configuration
    batch_size INT,
    config JSONB,
    
    -- Frame counts
    total_frames INT,
    sampled_frames INT,
    
    -- Aggregated performance
    avg_latency_ms FLOAT,
    p50_latency_ms FLOAT,
    p95_latency_ms FLOAT,
    p99_latency_ms FLOAT,
    min_latency_ms FLOAT,
    max_latency_ms FLOAT,
    std_latency_ms FLOAT,
    
    -- Throughput
    avg_fps FLOAT,
    
    -- Resource utilization (averages)
    avg_npu_utilization FLOAT,
    avg_memory_bandwidth FLOAT,
    
    -- Paths
    trace_path VARCHAR(500),
    stats_path VARCHAR(500),
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_model_run_soc_exec (soc_exec_id),
    INDEX idx_model_run_model (model_id),
    INDEX idx_model_run_run (run_id),
    INDEX idx_model_run_latency (avg_latency_ms)
);

-- ============================================================================
-- Frame-Level Data (aggregated, not raw traces)
-- ============================================================================

CREATE TABLE frame_stats (
    frame_stat_id BIGSERIAL PRIMARY KEY,
    model_run_id VARCHAR(300) REFERENCES model_runs(model_run_id),
    frame_index INT NOT NULL,
    
    -- Timing
    latency_ms FLOAT NOT NULL,
    preprocess_ms FLOAT,
    inference_ms FLOAT,
    postprocess_ms FLOAT,
    
    -- Resource utilization
    npu_utilization FLOAT,
    memory_bandwidth_gbps FLOAT,
    dma_utilization FLOAT,
    
    -- Flags
    is_outlier BOOLEAN DEFAULT FALSE,
    has_full_trace BOOLEAN DEFAULT FALSE,
    trace_path VARCHAR(500),
    
    -- Timestamp
    collected_at TIMESTAMP,
    
    INDEX idx_frame_stats_model_run (model_run_id),
    INDEX idx_frame_stats_latency (latency_ms),
    INDEX idx_frame_stats_outlier (is_outlier)
) PARTITION BY RANGE (collected_at);

-- Create partitions by month
CREATE TABLE frame_stats_2026_01 PARTITION OF frame_stats
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- ============================================================================
-- Operator-Level Statistics
-- ============================================================================

CREATE TABLE operator_stats (
    op_stat_id BIGSERIAL PRIMARY KEY,
    model_run_id VARCHAR(300) REFERENCES model_runs(model_run_id),
    
    -- Operator identity
    op_name VARCHAR(255) NOT NULL,
    op_type VARCHAR(100) NOT NULL,
    op_index INT,
    
    -- Characteristics
    flops BIGINT,
    memory_bytes BIGINT,
    input_shape JSONB,
    output_shape JSONB,
    
    -- Timing statistics
    avg_duration_us FLOAT,
    p50_duration_us FLOAT,
    p95_duration_us FLOAT,
    min_duration_us FLOAT,
    max_duration_us FLOAT,
    std_duration_us FLOAT,
    
    -- Percentage of total
    time_percentage FLOAT,
    
    INDEX idx_op_stats_model_run (model_run_id),
    INDEX idx_op_stats_type (op_type),
    INDEX idx_op_stats_duration (avg_duration_us DESC)
);

-- ============================================================================
-- Metric Registry
-- ============================================================================

CREATE TABLE metric_registry (
    metric_id SERIAL PRIMARY KEY,
    metric_name VARCHAR(255) UNIQUE NOT NULL,
    metric_type VARCHAR(50) NOT NULL,  -- counter, gauge, histogram
    unit VARCHAR(50),
    description TEXT,
    labels JSONB,
    
    -- Ownership
    owner VARCHAR(255),
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ============================================================================
-- ML Features Cache
-- ============================================================================

CREATE TABLE ml_features (
    feature_id BIGSERIAL PRIMARY KEY,
    model_run_id VARCHAR(300) REFERENCES model_runs(model_run_id),
    frame_index INT,
    
    -- Feature vectors (stored as arrays)
    model_features FLOAT[],
    hardware_features FLOAT[],
    
    -- Labels
    latency_ms FLOAT,
    throughput_fps FLOAT,
    
    -- Paths to full data
    graph_embedding_path VARCHAR(500),
    
    INDEX idx_ml_features_model_run (model_run_id)
);

-- ============================================================================
-- Views for Common Queries
-- ============================================================================

-- Latest run for each SoC+Model combination
CREATE VIEW latest_model_performance AS
SELECT DISTINCT ON (s.soc_name, m.model_name)
    br.run_id,
    s.soc_name,
    m.model_name,
    mr.avg_latency_ms,
    mr.p95_latency_ms,
    mr.avg_fps,
    mr.avg_npu_utilization,
    br.created_at
FROM model_runs mr
JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
JOIN socs s ON se.soc_id = s.soc_id
JOIN models m ON mr.model_id = m.model_id
JOIN benchmark_runs br ON mr.run_id = br.run_id
WHERE br.status = 'completed'
ORDER BY s.soc_name, m.model_name, br.created_at DESC;

-- Performance comparison across SoCs
CREATE VIEW soc_comparison AS
SELECT 
    m.model_name,
    s.soc_name,
    AVG(mr.avg_latency_ms) as avg_latency,
    AVG(mr.avg_fps) as avg_throughput,
    COUNT(*) as run_count
FROM model_runs mr
JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
JOIN socs s ON se.soc_id = s.soc_id
JOIN models m ON mr.model_id = m.model_id
JOIN benchmark_runs br ON mr.run_id = br.run_id
WHERE br.status = 'completed'
    AND br.created_at > NOW() - INTERVAL '30 days'
GROUP BY m.model_name, s.soc_name;
```

### Time-Series Schema (TimescaleDB)

```sql
-- ============================================================================
-- Metrics Time-Series
-- ============================================================================

CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    run_id VARCHAR(100),
    soc_id VARCHAR(100),
    model_id VARCHAR(100),
    metric_name VARCHAR(255),
    value DOUBLE PRECISION,
    labels JSONB
);

-- Convert to hypertable
SELECT create_hypertable('metrics', 'time');

-- Indexes for common queries
CREATE INDEX idx_metrics_run ON metrics (run_id, time DESC);
CREATE INDEX idx_metrics_name ON metrics (metric_name, time DESC);

-- ============================================================================
-- Hardware Counters Time-Series
-- ============================================================================

CREATE TABLE hw_counters (
    time TIMESTAMPTZ NOT NULL,
    run_id VARCHAR(100),
    soc_id VARCHAR(100),
    model_id VARCHAR(100),
    frame_index INT,
    counter_name VARCHAR(255),
    value BIGINT
);

SELECT create_hypertable('hw_counters', 'time');

-- ============================================================================
-- Continuous Aggregates (Materialized Views)
-- ============================================================================

-- Per-minute aggregates
CREATE MATERIALIZED VIEW metrics_1min
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 minute', time) AS bucket,
    run_id,
    soc_id,
    metric_name,
    AVG(value) as avg_value,
    MIN(value) as min_value,
    MAX(value) as max_value,
    COUNT(*) as sample_count
FROM metrics
GROUP BY bucket, run_id, soc_id, metric_name
WITH NO DATA;

-- Refresh policy
SELECT add_continuous_aggregate_policy('metrics_1min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute');

-- Per-hour aggregates
CREATE MATERIALIZED VIEW metrics_1h
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) AS bucket,
    run_id,
    soc_id,
    metric_name,
    AVG(value) as avg_value,
    MIN(value) as min_value,
    MAX(value) as max_value,
    COUNT(*) as sample_count
FROM metrics
GROUP BY bucket, run_id, soc_id, metric_name
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_1h',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Parquet Schemas

```python
# schemas.py - PyArrow schemas for Parquet files

import pyarrow as pa

# Frame statistics schema
frame_stats_schema = pa.schema([
    ('model_run_id', pa.string()),
    ('frame_index', pa.int32()),
    ('latency_ms', pa.float64()),
    ('preprocess_ms', pa.float64()),
    ('inference_ms', pa.float64()),
    ('postprocess_ms', pa.float64()),
    ('npu_utilization', pa.float64()),
    ('memory_bandwidth_gbps', pa.float64()),
    ('dma_utilization', pa.float64()),
    ('collected_at', pa.timestamp('us')),
])

# Operator statistics schema
operator_stats_schema = pa.schema([
    ('model_run_id', pa.string()),
    ('op_name', pa.string()),
    ('op_type', pa.string()),
    ('op_index', pa.int32()),
    ('flops', pa.int64()),
    ('memory_bytes', pa.int64()),
    ('input_shape', pa.list_(pa.int64())),
    ('output_shape', pa.list_(pa.int64())),
    ('avg_duration_us', pa.float64()),
    ('p50_duration_us', pa.float64()),
    ('p95_duration_us', pa.float64()),
    ('min_duration_us', pa.float64()),
    ('max_duration_us', pa.float64()),
    ('std_duration_us', pa.float64()),
    ('time_percentage', pa.float64()),
])

# ML features schema
ml_features_schema = pa.schema([
    ('model_run_id', pa.string()),
    ('frame_index', pa.int32()),
    ('model_features', pa.list_(pa.float32())),
    ('hardware_features', pa.list_(pa.float32())),
    ('operator_features', pa.list_(pa.list_(pa.float32()))),
    ('latency_ms', pa.float64()),
    ('bottleneck', pa.string()),
])
```

---

## Data Lifecycle

### Ingestion Pipeline

```python
class IngestionPipeline:
    """Pipeline for ingesting benchmark data."""
    
    def __init__(self, config):
        self.object_store = ObjectStore(config.s3_endpoint)
        self.postgres = PostgresClient(config.postgres_url)
        self.timeseries = TimeseriesClient(config.timescale_url)
        self.queue = QueueClient(config.kafka_url)
    
    async def ingest_run(self, run_id: str, data_path: str):
        """Ingest a complete benchmark run."""
        
        # 1. Parse manifest
        manifest = self._load_manifest(data_path)
        
        # 2. Create run record
        await self.postgres.execute("""
            INSERT INTO benchmark_runs (run_id, git_commit, config)
            VALUES ($1, $2, $3)
        """, run_id, manifest['git_commit'], manifest['config'])
        
        # 3. Process each SoC
        for soc_data in manifest['socs']:
            await self._ingest_soc(run_id, soc_data)
        
        # 4. Mark complete
        await self.postgres.execute("""
            UPDATE benchmark_runs 
            SET status = 'completed', completed_at = NOW()
            WHERE run_id = $1
        """, run_id)
    
    async def _ingest_soc(self, run_id: str, soc_data: dict):
        """Ingest data for one SoC."""
        
        soc_exec_id = f"{run_id}/{soc_data['soc_id']}"
        
        # Upsert SoC
        await self._upsert_soc(soc_data)
        
        # Create SoC execution
        await self.postgres.execute("""
            INSERT INTO soc_executions (soc_exec_id, run_id, soc_id, firmware_version)
            VALUES ($1, $2, $3, $4)
        """, soc_exec_id, run_id, soc_data['soc_id'], soc_data['firmware_version'])
        
        # Process each model
        for model_data in soc_data['models']:
            await self._ingest_model_run(soc_exec_id, model_data)
    
    async def _ingest_model_run(self, soc_exec_id: str, model_data: dict):
        """Ingest model run data."""
        
        model_run_id = f"{soc_exec_id}/{model_data['model_id']}"
        
        # 1. Upload raw traces to object store
        if model_data.get('traces'):
            trace_path = f"raw/runs/{model_run_id}/traces/"
            await self.object_store.upload_directory(
                model_data['traces_local_path'],
                trace_path
            )
        
        # 2. Compute and store aggregated stats
        stats = self._compute_stats(model_data)
        stats_path = f"aggregated/runs/{model_run_id}/stats.parquet"
        await self._write_parquet(stats, stats_path)
        
        # 3. Insert model run record
        await self.postgres.execute("""
            INSERT INTO model_runs (
                model_run_id, soc_exec_id, model_id,
                avg_latency_ms, p50_latency_ms, p95_latency_ms,
                avg_fps, avg_npu_utilization,
                trace_path, stats_path
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        """, model_run_id, soc_exec_id, model_data['model_id'],
            stats['avg_latency'], stats['p50_latency'], stats['p95_latency'],
            stats['avg_fps'], stats['avg_npu_util'],
            trace_path, stats_path)
        
        # 4. Insert frame-level stats
        await self._insert_frame_stats(model_run_id, stats['frames'])
        
        # 5. Insert operator stats
        await self._insert_operator_stats(model_run_id, stats['operators'])
        
        # 6. Ingest metrics to time-series DB
        await self._ingest_metrics(model_run_id, model_data['metrics'])
        
        # 7. Extract and store ML features
        features = self._extract_features(model_data)
        await self._store_features(model_run_id, features)
```

### Retention Policies

```python
# retention_config.yaml

retention_policies:
  raw_traces:
    hot:
      storage: s3_standard
      duration: 7 days
    warm:
      storage: s3_infrequent_access
      duration: 30 days
    cold:
      storage: s3_glacier
      duration: 90 days
    delete_after: 180 days
    
  aggregated_stats:
    storage: s3_standard
    duration: forever
    
  metrics_timeseries:
    raw:
      duration: 7 days
    1min_aggregates:
      duration: 30 days
    1hour_aggregates:
      duration: 365 days
    1day_aggregates:
      duration: forever
      
  ml_features:
    storage: s3_standard
    duration: forever
```

### Lifecycle Manager

```python
class DataLifecycleManager:
    """Manage data retention and transitions."""
    
    async def run_lifecycle_jobs(self):
        """Run all lifecycle management jobs."""
        
        await self._transition_cold_data()
        await self._delete_expired_data()
        await self._compact_partitions()
        await self._update_aggregates()
    
    async def _transition_cold_data(self):
        """Move old data to cheaper storage tiers."""
        
        # Find runs older than 7 days for transition to IA
        old_runs = await self.postgres.fetch("""
            SELECT run_id, raw_trace_path
            FROM benchmark_runs
            WHERE created_at < NOW() - INTERVAL '7 days'
            AND storage_class = 'STANDARD'
        """)
        
        for run in old_runs:
            await self.object_store.transition_to_ia(run['raw_trace_path'])
            await self.postgres.execute("""
                UPDATE benchmark_runs SET storage_class = 'INFREQUENT_ACCESS'
                WHERE run_id = $1
            """, run['run_id'])
    
    async def _delete_expired_data(self):
        """Delete data past retention period."""
        
        # Delete raw traces older than 180 days
        expired_runs = await self.postgres.fetch("""
            SELECT run_id, raw_trace_path
            FROM benchmark_runs
            WHERE created_at < NOW() - INTERVAL '180 days'
            AND raw_traces_deleted = FALSE
        """)
        
        for run in expired_runs:
            await self.object_store.delete(run['raw_trace_path'])
            await self.postgres.execute("""
                UPDATE benchmark_runs SET raw_traces_deleted = TRUE
                WHERE run_id = $1
            """, run['run_id'])
        
        # Drop old metrics partitions
        await self.timeseries.execute("""
            SELECT drop_chunks('metrics', older_than => INTERVAL '7 days')
        """)
```

---

## Query Patterns

### Query API

```python
from dataclasses import dataclass
from typing import List, Optional

class ExoQueryClient:
    """Client for querying Exo storage."""
    
    def __init__(self, config):
        self.postgres = PostgresClient(config.postgres_url)
        self.timeseries = TimeseriesClient(config.timescale_url)
        self.object_store = ObjectStore(config.s3_endpoint)
    
    # =========================================================================
    # Run Queries
    # =========================================================================
    
    async def list_runs(
        self,
        start_date: Optional[datetime] = None,
        end_date: Optional[datetime] = None,
        git_branch: Optional[str] = None,
        limit: int = 100
    ) -> List[BenchmarkRun]:
        """List benchmark runs with filters."""
        
        query = """
            SELECT * FROM benchmark_runs
            WHERE status = 'completed'
        """
        params = []
        
        if start_date:
            query += f" AND created_at >= ${len(params)+1}"
            params.append(start_date)
        
        if end_date:
            query += f" AND created_at <= ${len(params)+1}"
            params.append(end_date)
        
        if git_branch:
            query += f" AND git_branch = ${len(params)+1}"
            params.append(git_branch)
        
        query += f" ORDER BY created_at DESC LIMIT ${len(params)+1}"
        params.append(limit)
        
        rows = await self.postgres.fetch(query, *params)
        return [BenchmarkRun.from_row(r) for r in rows]
    
    # =========================================================================
    # Performance Queries
    # =========================================================================
    
    async def get_model_performance(
        self,
        model_name: str,
        soc_name: Optional[str] = None,
        last_n_runs: int = 10
    ) -> pd.DataFrame:
        """Get performance history for a model."""
        
        query = """
            SELECT 
                br.created_at,
                br.git_commit,
                s.soc_name,
                mr.avg_latency_ms,
                mr.p95_latency_ms,
                mr.avg_fps,
                mr.avg_npu_utilization
            FROM model_runs mr
            JOIN models m ON mr.model_id = m.model_id
            JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
            JOIN socs s ON se.soc_id = s.soc_id
            JOIN benchmark_runs br ON mr.run_id = br.run_id
            WHERE m.model_name = $1
        """
        params = [model_name]
        
        if soc_name:
            query += f" AND s.soc_name = ${len(params)+1}"
            params.append(soc_name)
        
        query += """
            ORDER BY br.created_at DESC
            LIMIT ${}
        """.format(len(params)+1)
        params.append(last_n_runs)
        
        rows = await self.postgres.fetch(query, *params)
        return pd.DataFrame(rows)
    
    async def compare_socs(
        self,
        model_name: str,
        soc_names: List[str],
        run_id: Optional[str] = None
    ) -> pd.DataFrame:
        """Compare model performance across SoCs."""
        
        if run_id:
            run_filter = "br.run_id = $3"
        else:
            run_filter = """
                br.run_id = (
                    SELECT run_id FROM benchmark_runs 
                    WHERE status = 'completed' 
                    ORDER BY created_at DESC LIMIT 1
                )
            """
        
        query = f"""
            SELECT 
                s.soc_name,
                mr.avg_latency_ms,
                mr.p50_latency_ms,
                mr.p95_latency_ms,
                mr.avg_fps,
                mr.avg_npu_utilization,
                mr.avg_memory_bandwidth
            FROM model_runs mr
            JOIN models m ON mr.model_id = m.model_id
            JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
            JOIN socs s ON se.soc_id = s.soc_id
            JOIN benchmark_runs br ON mr.run_id = br.run_id
            WHERE m.model_name = $1
            AND s.soc_name = ANY($2)
            AND {run_filter}
        """
        
        params = [model_name, soc_names]
        if run_id:
            params.append(run_id)
        
        rows = await self.postgres.fetch(query, *params)
        return pd.DataFrame(rows)
    
    # =========================================================================
    # Regression Detection
    # =========================================================================
    
    async def detect_regressions(
        self,
        current_run_id: str,
        baseline_run_id: str,
        threshold_pct: float = 5.0
    ) -> List[Regression]:
        """Detect performance regressions between runs."""
        
        query = """
            WITH current AS (
                SELECT model_id, soc_exec_id, avg_latency_ms
                FROM model_runs WHERE run_id = $1
            ),
            baseline AS (
                SELECT model_id, soc_exec_id, avg_latency_ms
                FROM model_runs WHERE run_id = $2
            )
            SELECT 
                c.model_id,
                se.soc_id,
                b.avg_latency_ms as baseline_latency,
                c.avg_latency_ms as current_latency,
                ((c.avg_latency_ms - b.avg_latency_ms) / b.avg_latency_ms * 100) as pct_change
            FROM current c
            JOIN baseline b ON c.model_id = b.model_id 
                AND c.soc_exec_id LIKE '%' || SPLIT_PART(b.soc_exec_id, '/', 2)
            JOIN soc_executions se ON c.soc_exec_id = se.soc_exec_id
            WHERE ((c.avg_latency_ms - b.avg_latency_ms) / b.avg_latency_ms * 100) > $3
            ORDER BY pct_change DESC
        """
        
        rows = await self.postgres.fetch(query, current_run_id, baseline_run_id, threshold_pct)
        return [Regression.from_row(r) for r in rows]
    
    # =========================================================================
    # Trace Queries
    # =========================================================================
    
    async def get_trace(self, model_run_id: str, frame_index: int) -> bytes:
        """Get raw trace for a specific frame."""
        
        # Get path from database
        row = await self.postgres.fetchrow("""
            SELECT trace_path FROM frame_stats
            WHERE model_run_id = $1 AND frame_index = $2 AND has_full_trace = TRUE
        """, model_run_id, frame_index)
        
        if not row:
            raise TraceNotFoundError(f"No trace for {model_run_id} frame {frame_index}")
        
        # Download from object store
        return await self.object_store.download(row['trace_path'])
    
    async def get_outlier_traces(
        self,
        model_run_id: str,
        limit: int = 10
    ) -> List[FrameStats]:
        """Get traces for outlier frames."""
        
        rows = await self.postgres.fetch("""
            SELECT * FROM frame_stats
            WHERE model_run_id = $1 
            AND is_outlier = TRUE 
            AND has_full_trace = TRUE
            ORDER BY latency_ms DESC
            LIMIT $2
        """, model_run_id, limit)
        
        return [FrameStats.from_row(r) for r in rows]
    
    # =========================================================================
    # Metrics Queries
    # =========================================================================
    
    async def get_metrics_timeseries(
        self,
        run_id: str,
        metric_names: List[str],
        start_time: datetime,
        end_time: datetime,
        resolution: str = '1min'
    ) -> pd.DataFrame:
        """Get metrics time-series data."""
        
        # Use appropriate aggregate based on resolution
        if resolution == '1min':
            table = 'metrics_1min'
        elif resolution == '1h':
            table = 'metrics_1h'
        else:
            table = 'metrics'
        
        query = f"""
            SELECT 
                bucket as time,
                metric_name,
                avg_value as value
            FROM {table}
            WHERE run_id = $1
            AND metric_name = ANY($2)
            AND bucket BETWEEN $3 AND $4
            ORDER BY bucket
        """
        
        rows = await self.timeseries.fetch(query, run_id, metric_names, start_time, end_time)
        return pd.DataFrame(rows).pivot(index='time', columns='metric_name', values='value')
    
    # =========================================================================
    # ML Feature Queries
    # =========================================================================
    
    async def get_training_data(
        self,
        filters: Optional[dict] = None,
        limit: Optional[int] = None
    ) -> pd.DataFrame:
        """Get training data for ML models."""
        
        query = """
            SELECT 
                mf.model_run_id,
                mf.frame_index,
                mf.model_features,
                mf.hardware_features,
                mf.latency_ms,
                m.model_name,
                m.architecture,
                s.soc_name
            FROM ml_features mf
            JOIN model_runs mr ON mf.model_run_id = mr.model_run_id
            JOIN models m ON mr.model_id = m.model_id
            JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
            JOIN socs s ON se.soc_id = s.soc_id
            WHERE TRUE
        """
        params = []
        
        if filters:
            if 'model_architecture' in filters:
                query += f" AND m.architecture = ANY(${len(params)+1})"
                params.append(filters['model_architecture'])
            if 'soc_name' in filters:
                query += f" AND s.soc_name = ANY(${len(params)+1})"
                params.append(filters['soc_name'])
        
        if limit:
            query += f" LIMIT ${len(params)+1}"
            params.append(limit)
        
        rows = await self.postgres.fetch(query, *params)
        return pd.DataFrame(rows)
```

---

## Scalability

### Horizontal Scaling

```yaml
# Kubernetes deployment for scaled storage

# PostgreSQL with read replicas
postgresql:
  primary:
    resources:
      cpu: 4
      memory: 16Gi
    storage: 500Gi
  replicas:
    count: 2
    resources:
      cpu: 2
      memory: 8Gi

# TimescaleDB cluster
timescaledb:
  replicas: 3
  resources:
    cpu: 4
    memory: 16Gi
  storage:
    size: 1Ti
    class: ssd

# MinIO cluster
minio:
  replicas: 4
  resources:
    cpu: 2
    memory: 8Gi
  storage:
    size: 10Ti
    class: hdd

# Redis for caching
redis:
  mode: cluster
  replicas: 6
  resources:
    cpu: 1
    memory: 4Gi
```

### Query Optimization

```sql
-- Materialized view for common comparison queries
CREATE MATERIALIZED VIEW mv_model_soc_latest AS
SELECT 
    m.model_name,
    m.architecture,
    s.soc_name,
    s.vendor,
    mr.avg_latency_ms,
    mr.p95_latency_ms,
    mr.avg_fps,
    mr.avg_npu_utilization,
    br.created_at as run_date,
    mr.model_run_id
FROM model_runs mr
JOIN models m ON mr.model_id = m.model_id
JOIN soc_executions se ON mr.soc_exec_id = se.soc_exec_id
JOIN socs s ON se.soc_id = s.soc_id
JOIN benchmark_runs br ON mr.run_id = br.run_id
WHERE br.status = 'completed'
AND br.created_at > NOW() - INTERVAL '30 days';

CREATE INDEX idx_mv_model_soc ON mv_model_soc_latest (model_name, soc_name);

-- Refresh periodically
CREATE OR REPLACE FUNCTION refresh_model_soc_view()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_model_soc_latest;
END;
$$ LANGUAGE plpgsql;

-- Schedule refresh every hour
SELECT cron.schedule('refresh-mv', '0 * * * *', 'SELECT refresh_model_soc_view()');
```

---

## Implementation Options

### Option 1: Lightweight (Single Machine)

For smaller teams or development:

```yaml
components:
  database: PostgreSQL (with TimescaleDB extension)
  object_store: Local filesystem or MinIO single-node
  cache: Redis single-node
  
deployment: Docker Compose

estimated_capacity:
  runs_per_month: 100
  storage: 500GB
  cost: ~$100/month (cloud VM)
```

### Option 2: Production (Cloud-Native)

For production use:

```yaml
components:
  database: 
    metadata: Amazon RDS PostgreSQL
    timeseries: TimescaleDB Cloud or Amazon Timestream
  object_store: Amazon S3
  cache: Amazon ElastiCache Redis
  queue: Amazon MSK (Kafka)
  
deployment: Kubernetes (EKS/GKE)

estimated_capacity:
  runs_per_month: 10,000+
  storage: 50TB+
  cost: ~$2,000-5,000/month
```

### Option 3: Self-Hosted Enterprise

For air-gapped or on-premise:

```yaml
components:
  database: PostgreSQL + TimescaleDB on bare metal
  object_store: MinIO cluster
  cache: Redis cluster
  queue: Kafka on Kubernetes
  
deployment: Kubernetes (on-premise)

estimated_capacity:
  runs_per_month: 50,000+
  storage: 500TB+
  cost: Hardware + operations
```

---

## Summary

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Raw trace format** | Perfetto in object store | Compact, good tooling |
| **Aggregates** | Parquet + PostgreSQL | Efficient analytics |
| **Metrics** | TimescaleDB | Time-series optimized |
| **ML features** | Parquet + PostgreSQL | Easy export for training |
| **Tiered storage** | Hot/warm/cold | Cost optimization |

### API Endpoints Summary

| Endpoint | Description |
|----------|-------------|
| `POST /runs` | Create new benchmark run |
| `GET /runs` | List runs with filters |
| `GET /runs/{id}/performance` | Get performance summary |
| `GET /runs/{id}/compare/{baseline}` | Compare with baseline |
| `GET /models/{name}/history` | Performance history |
| `GET /socs/compare` | Cross-SoC comparison |
| `GET /traces/{id}` | Get raw trace |
| `GET /metrics/timeseries` | Get metrics data |
| `GET /ml/features` | Export ML features |
