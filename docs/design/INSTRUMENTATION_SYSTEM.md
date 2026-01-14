# Instrumentation System Design

**Comprehensive Trace, Event, and Metric Collection for AI Benchmarks**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Data Model](#data-model)
2. [Critical Review](#critical-review)
3. [Instrumentation Architecture](#instrumentation-architecture)
4. [Trace Collection](#trace-collection)
5. [Event Collection](#event-collection)
6. [Metric Collection](#metric-collection)
7. [User-Defined Instrumentation](#user-defined-instrumentation)
8. [Collection Profiles](#collection-profiles)
9. [Data Normalization](#data-normalization)
10. [Implementation](#implementation)

---

## Data Model

### Hierarchy

```
CI Benchmark Run
├── Run Metadata (git commit, timestamp, config)
├── SoC 1
│   ├── SoC Metadata (name, firmware version, hw config)
│   ├── Model A
│   │   ├── Model Metadata (name, format, params)
│   │   ├── Frame 1 → Trace (spans + events + metrics)
│   │   ├── Frame 2 → Trace
│   │   ├── ...
│   │   └── Frame N → Trace
│   ├── Model B
│   │   ├── Frame 1 → Trace
│   │   └── ...
│   └── ...
├── SoC 2
│   └── ...
└── SoC N
    └── ...
```

### Entity Relationships

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Data Model                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐                                                           │
│  │ BenchmarkRun │ 1                                                         │
│  │              │───────┐                                                   │
│  │ - run_id     │       │                                                   │
│  │ - timestamp  │       │ has many                                          │
│  │ - git_commit │       │                                                   │
│  │ - config     │       ▼                                                   │
│  └──────────────┘  ┌──────────────┐                                         │
│                    │ SoCExecution │ 1                                        │
│                    │              │───────┐                                  │
│                    │ - soc_id     │       │                                  │
│                    │ - soc_name   │       │ has many                         │
│                    │ - fw_version │       │                                  │
│                    │ - hw_config  │       ▼                                  │
│                    └──────────────┘  ┌──────────────┐                        │
│                                      │ ModelRun     │ 1                      │
│                                      │              │───────┐                │
│                                      │ - model_id   │       │                │
│                                      │ - model_name │       │ has many       │
│                                      │ - batch_size │       │                │
│                                      │ - config     │       ▼                │
│                                      └──────────────┘  ┌──────────────┐      │
│                                                        │ FrameTrace   │      │
│                                                        │              │      │
│                                                        │ - trace_id   │      │
│                                                        │ - frame_idx  │      │
│                                                        │ - timestamp  │      │
│                                                        │              │      │
│                                                        │ contains:    │      │
│                                                        │ - Spans[]    │      │
│                                                        │ - Events[]   │      │
│                                                        │ - Metrics[]  │      │
│                                                        │ - Counters[] │      │
│                                                        └──────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Identifiers

```python
# Hierarchical ID structure
run_id      = "ci-2026-01-13-abc123"           # CI run
soc_exec_id = "{run_id}/soc-npu-v2-001"        # SoC within run
model_run_id = "{soc_exec_id}/resnet50-bs1"    # Model within SoC
trace_id    = "{model_run_id}/frame-00042"     # Frame trace

# W3C Trace Context compatible
trace_id_w3c = "4bf92f3577b34da6a3ce929d0e0e4736"  # 128-bit hex
```

---

## Critical Review

### Strengths of This Approach

| Aspect | Strength |
|--------|----------|
| **Hierarchical structure** | Natural mapping to benchmark organization |
| **Per-frame traces** | Enables statistical analysis across frames |
| **Multi-SoC support** | Enables cross-platform comparison |
| **Extensible metrics** | User can add domain-specific measurements |

### Concerns and Mitigations

#### 1. Data Volume

**Concern:** This can generate massive amounts of data.

```
Example calculation:
- 10 SoCs × 50 models × 1000 frames = 500,000 traces
- Each trace: ~100 spans × 200 bytes = 20KB
- Total per run: 500,000 × 20KB = 10GB
- Daily CI: 10 runs × 10GB = 100GB/day
```

**Mitigations:**
- Sampling strategies (not every frame)
- Aggregation (store aggregates, sample details)
- Tiered storage (hot/warm/cold)
- Retention policies (delete old detailed data, keep aggregates)

```python
# Sampling configuration
collection_config = {
    "trace_sampling": {
        "warmup_frames": 10,      # Skip first N frames
        "sample_rate": 0.1,       # Collect 10% of frames with full detail
        "always_collect": {
            "first_frame": True,  # Always collect first frame
            "last_frame": True,   # Always collect last frame
            "outliers": True,     # Always collect if latency > 3σ
        }
    },
    "aggregation": {
        "per_frame_metrics": True,   # Always store per-frame metrics
        "per_frame_traces": False,   # Only for sampled frames
        "aggregate_operators": True, # Store aggregated op stats
    }
}
```

#### 2. Trace Format Standardization

**Concern:** Raw traces from different SoCs may have different formats.

**Analysis:**

| Format | Pros | Cons |
|--------|------|------|
| **Perfetto** | Excellent visualization, system-level support, counter tracks | Binary format, less suited for distributed tracing |
| **OTLP** | Industry standard, many backends, distributed tracing | Weaker timeline visualization, no native counter tracks |
| **Chrome Trace** | Simple, universal support | JSON overhead, limited features |

**Recommendation:** Use a **two-layer approach**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Internal Canonical Model                      │
│                                                                  │
│  Exo's internal representation that captures ALL semantics:     │
│  - Spans (with full OpenTelemetry semantics)                    │
│  - Events (timestamped, with attributes)                        │
│  - Metrics (counters, gauges, histograms)                       │
│  - Counters (hardware performance counters as time series)      │
│  - Track assignments (which HW unit executed what)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
     ┌───────────┐        ┌───────────┐        ┌───────────┐
     │ Perfetto  │        │   OTLP    │        │  Chrome   │
     │ Export    │        │  Export   │        │  Export   │
     └───────────┘        └───────────┘        └───────────┘
     
     For timeline        For backends         For quick
     visualization       (Jaeger, Tempo)      debugging
```

#### 3. Hardware Metric Availability

**Concern:** Different SoCs expose different hardware counters.

**Analysis:**
- Some SoCs have rich PMU (Performance Monitoring Unit)
- Others have minimal or no visibility
- Cannot assume uniform counter availability

**Recommendation:** Abstract with capability discovery:

```python
# SoC capabilities declaration
soc_capabilities = {
    "soc_id": "npu-accelerator-v2",
    "available_counters": [
        {"name": "npu.cycles", "unit": "cycles", "type": "counter"},
        {"name": "npu.utilization", "unit": "percent", "type": "gauge"},
        {"name": "dram.bandwidth", "unit": "GB/s", "type": "gauge"},
        {"name": "dma.transactions", "unit": "count", "type": "counter"},
    ],
    "unavailable_counters": [
        {"name": "cache.misses", "reason": "not exposed by hardware"},
    ],
    "sampling_capabilities": {
        "min_interval_us": 100,
        "max_counters_simultaneous": 4,
    }
}
```

#### 4. User-Defined Metrics Governance

**Concern:** Allowing arbitrary user metrics can lead to chaos.

**Analysis:**
- Need discoverability (what metrics exist?)
- Need validation (is data well-formed?)
- Need documentation (what does this metric mean?)

**Recommendation:** Metric registry with schema:

```python
# Metric registration
metric_registry.register(
    name="custom.my_accelerator.queue_depth",
    type="gauge",
    unit="count",
    description="Number of pending operations in accelerator queue",
    labels=["accelerator_id", "priority"],
    valid_range=(0, 1024),  # Validation
    owner="team-accelerator@example.com",
)
```

#### 5. Correlation Across Levels

**Concern:** How to correlate spans across model/SoC/run levels?

**Recommendation:** Use consistent trace context:

```python
# All spans within a frame share trace_id
# Parent-child relationships use span_id
# Cross-frame/cross-model correlation via explicit links

span.add_link(
    trace_id=previous_frame_trace_id,
    span_id=previous_frame_root_span_id,
    attributes={"link.type": "sequential_frame"}
)
```

#### 6. Real-time vs Batch Collection

**Concern:** Do we need real-time streaming or is batch sufficient?

**Analysis:**

| Use Case | Real-time Needed? | Latency Tolerance |
|----------|------------------|-------------------|
| CI regression | No | Minutes |
| Development debugging | Yes | Seconds |
| Production monitoring | Depends | Seconds to minutes |
| Architecture exploration | No | Hours |

**Recommendation:** Support both modes:

```python
# Batch mode (CI benchmarks)
config = ExoConfig(
    export_mode="batch",
    batch_size=100,           # Export every 100 traces
    batch_timeout_s=60,       # Or every 60 seconds
    output="file://traces/",  # Write to files
)

# Streaming mode (development)
config = ExoConfig(
    export_mode="streaming",
    endpoint="otlp://collector:4317",
    flush_interval_ms=100,
)
```

---

## Instrumentation Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Target Device (SoC)                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         AI Firmware                                     │ │
│  │                                                                         │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │                    libexo Instrumentation                         │  │ │
│  │  │                                                                   │  │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │  │ │
│  │  │  │   Tracer    │  │   Meter     │  │  HW Counter │              │  │ │
│  │  │  │             │  │             │  │  Sampler    │              │  │ │
│  │  │  │ - Spans     │  │ - Counters  │  │             │              │  │ │
│  │  │  │ - Events    │  │ - Gauges    │  │ - PMU       │              │  │ │
│  │  │  │ - Context   │  │ - Histograms│  │ - Bus Mon   │              │  │ │
│  │  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │  │ │
│  │  │         │                │                │                      │  │ │
│  │  │         └────────────────┼────────────────┘                      │  │ │
│  │  │                          ▼                                        │  │ │
│  │  │  ┌─────────────────────────────────────────────────────────────┐ │  │ │
│  │  │  │                   Collection Buffer                          │ │  │ │
│  │  │  │                                                              │ │  │ │
│  │  │  │  Ring buffer for spans, events, metrics, counters           │ │  │ │
│  │  │  │  Lockfree, bounded memory, overflow policy                  │ │  │ │
│  │  │  │                                                              │ │  │ │
│  │  │  └──────────────────────────┬──────────────────────────────────┘ │  │ │
│  │  │                             │                                     │  │ │
│  │  │  ┌──────────────────────────▼──────────────────────────────────┐ │  │ │
│  │  │  │                   Exporter                                   │ │  │ │
│  │  │  │                                                              │ │  │ │
│  │  │  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐   │ │  │ │
│  │  │  │  │ Perfetto  │ │   OTLP    │ │   File    │ │   UART    │   │ │  │ │
│  │  │  │  │ (binary)  │ │ (proto)   │ │  (local)  │ │ (stream)  │   │ │  │ │
│  │  │  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘   │ │  │ │
│  │  │  │                                                              │ │  │ │
│  │  │  └──────────────────────────────────────────────────────────────┘ │  │ │
│  │  │                                                                   │  │ │
│  │  └───────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (files, network, shared memory)
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Host / Collection Server                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        Exo Collector                                     ││
│  │                                                                          ││
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────────┐   ││
│  │  │ Receiver      │  │ Processor     │  │ Storage Writer            │   ││
│  │  │               │  │               │  │                           │   ││
│  │  │ - File watch  │  │ - Validate    │  │ - Object store (traces)  │   ││
│  │  │ - OTLP gRPC   │  │ - Enrich      │  │ - TimeSeries DB (metrics)│   ││
│  │  │ - UART        │  │ - Normalize   │  │ - Metadata DB (catalog)  │   ││
│  │  │ - Shared mem  │  │ - Aggregate   │  │                           │   ││
│  │  └───────────────┘  └───────────────┘  └───────────────────────────┘   ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Trace Collection

### Trace Structure

```c
// Core trace structure
typedef struct exo_frame_trace {
    // Identity
    exo_trace_id_t trace_id;           // W3C compatible 128-bit ID
    char hierarchical_id[256];          // "run/soc/model/frame"
    uint32_t frame_index;
    
    // Context
    exo_run_context_t run_ctx;          // Run-level context
    exo_soc_context_t soc_ctx;          // SoC-level context
    exo_model_context_t model_ctx;      // Model-level context
    
    // Timing
    uint64_t start_timestamp_ns;
    uint64_t end_timestamp_ns;
    
    // Content (collected during frame execution)
    exo_span_t* spans;
    size_t span_count;
    
    exo_event_t* events;
    size_t event_count;
    
    exo_metric_sample_t* metrics;
    size_t metric_count;
    
    exo_hw_counter_sample_t* hw_counters;
    size_t hw_counter_count;
    
} exo_frame_trace_t;
```

### Span Definition

```c
typedef struct exo_span {
    // Standard OpenTelemetry fields
    exo_span_id_t span_id;
    exo_span_id_t parent_span_id;      // Zero if root
    
    char name[64];
    exo_span_kind_t kind;               // Internal, Producer, Consumer, etc.
    
    uint64_t start_time_ns;
    uint64_t end_time_ns;
    
    exo_span_status_t status;
    char status_message[128];
    
    // Attributes
    exo_attribute_t attributes[16];
    uint8_t attribute_count;
    
    // Events within span
    exo_span_event_t events[8];
    uint8_t event_count;
    
    // Links to related spans
    exo_span_link_t links[4];
    uint8_t link_count;
    
    // Exo extensions
    exo_track_id_t track_id;            // Which HW track (CPU, NPU, DMA, etc.)
    
} exo_span_t;
```

### Track Assignment

Tracks map to hardware units for timeline visualization:

```c
// Pre-defined tracks
typedef enum {
    EXO_TRACK_MAIN = 0,
    EXO_TRACK_CPU,
    EXO_TRACK_NPU,
    EXO_TRACK_GPU,
    EXO_TRACK_DMA,
    EXO_TRACK_DSP,
    EXO_TRACK_CUSTOM_BASE = 100,
} exo_track_id_t;

// Assign span to track
exo_span_set_track(span, EXO_TRACK_NPU);

// This produces Perfetto visualization with separate tracks:
// ▼ Frame 42
//   ├─ CPU    ─────[preprocess]─────────────────────[postprocess]────
//   ├─ NPU    ─────────────────[conv2d]──[relu]──[pool]──────────────
//   ├─ DMA    ────[in]────────────────────────────────────────[out]──
```

---

## Event Collection

### Event Types

```c
typedef enum {
    // Software events
    EXO_EVENT_GENERIC,
    EXO_EVENT_LOG,
    EXO_EVENT_ERROR,
    EXO_EVENT_CHECKPOINT,
    
    // Hardware events (usually from PMU/interrupts)
    EXO_EVENT_HW_INTERRUPT,
    EXO_EVENT_HW_DMA_COMPLETE,
    EXO_EVENT_HW_CACHE_FLUSH,
    EXO_EVENT_HW_POWER_STATE,
    
    // Custom events
    EXO_EVENT_CUSTOM = 1000,
} exo_event_type_t;

typedef struct exo_event {
    exo_event_type_t type;
    uint64_t timestamp_ns;
    char name[64];
    exo_track_id_t track_id;
    
    // Optional: associated span
    exo_span_id_t span_id;              // Zero if not associated
    
    // Attributes
    exo_attribute_t attributes[8];
    uint8_t attribute_count;
    
} exo_event_t;
```

### Event API

```c
// Record instantaneous event
exo_record_event("dma_transfer_complete", EXO_TRACK_DMA);

// Record event with attributes
exo_record_event_ex("dma_transfer_complete", EXO_TRACK_DMA, 
    (exo_attribute_t[]){
        EXO_ATTR_INT("bytes", 4096),
        EXO_ATTR_INT("channel", 2),
    }, 2);

// Record event within a span
exo_span_add_event(span, "started_execution");

// Record hardware event (from ISR)
void dma_isr(int channel) {
    exo_record_hw_event(EXO_EVENT_HW_DMA_COMPLETE, 
        (exo_attribute_t[]){
            EXO_ATTR_INT("channel", channel),
        }, 1);
    
    // Handle DMA...
}
```

---

## Metric Collection

### Metric Types

```c
typedef enum {
    EXO_METRIC_COUNTER,       // Monotonically increasing (e.g., total_ops)
    EXO_METRIC_GAUGE,         // Point-in-time value (e.g., temperature)
    EXO_METRIC_HISTOGRAM,     // Distribution (e.g., latency distribution)
    EXO_METRIC_SUMMARY,       // Pre-computed quantiles
} exo_metric_type_t;
```

### Standard Metrics (Built-in)

```yaml
# Timing metrics (automatically collected)
timing:
  - name: exo.frame.latency_ms
    type: gauge
    unit: milliseconds
    description: Total frame execution time
    
  - name: exo.span.duration_us
    type: histogram
    unit: microseconds
    description: Span duration distribution
    labels: [span_name, track]

# Resource utilization (require HW support)
utilization:
  - name: exo.npu.utilization
    type: gauge
    unit: percent
    description: NPU compute utilization
    
  - name: exo.memory.bandwidth_used
    type: gauge  
    unit: GB/s
    description: Memory bandwidth utilization
    
  - name: exo.dma.utilization
    type: gauge
    unit: percent
    description: DMA engine utilization

# Throughput metrics
throughput:
  - name: exo.inference.fps
    type: gauge
    unit: frames/second
    description: Inference throughput
    
  - name: exo.ops.per_second
    type: gauge
    unit: ops/second
    description: Operations per second
```

### Hardware Counter Metrics

```c
// Hardware counter configuration
typedef struct exo_hw_counter_config {
    const char* name;           // e.g., "npu.cycles"
    uint32_t hw_counter_id;     // Platform-specific counter ID
    exo_metric_type_t type;     // Usually COUNTER or GAUGE
    const char* unit;
    
    // Sampling configuration
    uint32_t sample_interval_us;
    bool sample_on_span_boundary;
    
} exo_hw_counter_config_t;

// Register hardware counters (platform-specific)
exo_hw_counter_config_t counters[] = {
    {"npu.cycles", HW_COUNTER_NPU_CYCLES, EXO_METRIC_COUNTER, "cycles", 100, true},
    {"npu.stalls", HW_COUNTER_NPU_STALLS, EXO_METRIC_COUNTER, "cycles", 100, true},
    {"dram.reads", HW_COUNTER_DRAM_RD, EXO_METRIC_COUNTER, "bytes", 1000, false},
    {"dram.writes", HW_COUNTER_DRAM_WR, EXO_METRIC_COUNTER, "bytes", 1000, false},
    {"bus.utilization", HW_COUNTER_BUS_UTIL, EXO_METRIC_GAUGE, "percent", 100, false},
};
exo_register_hw_counters(counters, sizeof(counters)/sizeof(counters[0]));

// Counters are automatically sampled based on configuration
```

### Derived Metrics

```c
// Derived metrics computed from raw counters
typedef struct exo_derived_metric {
    const char* name;
    const char* formula;        // Expression using other metric names
    const char* unit;
} exo_derived_metric_t;

exo_derived_metric_t derived[] = {
    {
        "npu.ipc",                              // Instructions per cycle
        "npu.instructions / npu.cycles",
        "instructions/cycle"
    },
    {
        "memory.arithmetic_intensity",
        "npu.flops / (dram.reads + dram.writes)",
        "FLOPS/byte"
    },
    {
        "npu.stall_ratio",
        "npu.stalls / npu.cycles",
        "ratio"
    },
};
exo_register_derived_metrics(derived, 3);
```

---

## User-Defined Instrumentation

### Metric Registry

```c
// User defines custom metrics at initialization
exo_metric_def_t my_metrics[] = {
    {
        .name = "myapp.queue_depth",
        .type = EXO_METRIC_GAUGE,
        .unit = "count",
        .description = "Pending items in processing queue",
        .labels = {"queue_name", "priority"},
        .label_count = 2,
    },
    {
        .name = "myapp.cache_hit_rate",
        .type = EXO_METRIC_GAUGE,
        .unit = "ratio",
        .description = "Weight cache hit rate",
        .labels = {},
        .label_count = 0,
    },
};

exo_status_t status = exo_register_metrics(my_metrics, 2);

// Get handles for recording
exo_gauge_t* queue_depth = exo_get_gauge("myapp.queue_depth");
exo_gauge_t* cache_hit = exo_get_gauge("myapp.cache_hit_rate");

// Record values
exo_gauge_set(queue_depth, 42, 
    (exo_label_t[]){{"queue_name", "inference"}, {"priority", "high"}}, 2);
exo_gauge_set(cache_hit, 0.85, NULL, 0);
```

### Custom Events

```c
// Register custom event types
exo_event_type_t MY_ACCEL_START = exo_register_event_type(
    "myapp.accelerator_start",
    "Custom accelerator operation started"
);

exo_event_type_t MY_ACCEL_DONE = exo_register_event_type(
    "myapp.accelerator_done",
    "Custom accelerator operation completed"
);

// Record custom events
exo_record_event_typed(MY_ACCEL_START, "conv2d_custom", EXO_TRACK_CUSTOM_BASE,
    (exo_attribute_t[]){
        EXO_ATTR_INT("input_channels", 64),
        EXO_ATTR_INT("output_channels", 128),
    }, 2);
```

### Configuration File

```yaml
# exo_instrumentation.yaml - User instrumentation configuration

# Custom metrics
metrics:
  - name: myapp.weight_cache.size
    type: gauge
    unit: bytes
    description: Current size of weight cache
    labels: []
    
  - name: myapp.weight_cache.hits
    type: counter
    unit: count
    description: Weight cache hit count
    labels: [layer_name]
    
  - name: myapp.dma.latency
    type: histogram
    unit: microseconds
    description: DMA transfer latency
    labels: [direction]  # in, out
    buckets: [10, 50, 100, 500, 1000, 5000]

# Custom events
events:
  - name: myapp.layer_start
    description: Neural network layer started execution
    
  - name: myapp.layer_complete
    description: Neural network layer completed
    attributes:
      - name: layer_name
        type: string
      - name: output_size
        type: int

# Custom tracks
tracks:
  - id: 100
    name: "Custom Accelerator"
    color: "#FF5733"
    
  - id: 101
    name: "Weight Prefetcher"
    color: "#33FF57"

# Hardware counters (platform-specific)
hw_counters:
  - name: myplatform.custom_counter_1
    hw_id: 0x100
    type: counter
    unit: count
    sample_interval_us: 100
```

### Loading Configuration

```c
// Load user configuration at init
exo_config_t config = {
    .service_name = "my_inference_engine",
    .instrumentation_config_path = "exo_instrumentation.yaml",
};
exo_init(&config);
```

---

## Collection Profiles

Different scenarios need different collection granularity:

### Profile: Minimal (Production)

```yaml
profile: minimal
description: Lightweight collection for production monitoring

traces:
  enabled: true
  sampling_rate: 0.01          # 1% of frames
  span_filter:
    - include: "inference"      # Only top-level
    - exclude: "*"              # No detailed spans
    
events:
  enabled: true
  filter:
    - include: "error.*"
    - include: "warning.*"
    
metrics:
  enabled: true
  interval_ms: 1000            # 1 second resolution
  filter:
    - include: "exo.frame.*"
    - include: "exo.*.utilization"
    
hw_counters:
  enabled: false               # Too expensive for production
```

### Profile: Standard (CI Benchmarks)

```yaml
profile: standard
description: Balanced collection for CI benchmarks

traces:
  enabled: true
  sampling_rate: 0.1           # 10% of frames with full detail
  warmup_frames: 10            # Skip first 10 frames
  always_sample:
    - first_frame: true
    - last_frame: true
    - outliers: true           # >3σ latency
    
  span_filter:
    - include: "*"             # All spans
    
events:
  enabled: true
  filter:
    - include: "*"             # All events
    
metrics:
  enabled: true
  interval_ms: 100             # 100ms resolution
  filter:
    - include: "*"             # All metrics
    
hw_counters:
  enabled: true
  sample_interval_us: 1000     # 1ms sampling
  filter:
    - include: "*.utilization"
    - include: "*.bandwidth"
```

### Profile: Detailed (Debugging/Profiling)

```yaml
profile: detailed
description: Maximum detail for debugging and profiling

traces:
  enabled: true
  sampling_rate: 1.0           # Every frame
  
  span_filter:
    - include: "*"
    
  # Extra span attributes
  extra_attributes:
    - input_tensors: true      # Log input tensor metadata
    - output_tensors: true
    - memory_allocations: true
    
events:
  enabled: true
  filter:
    - include: "*"
    
metrics:
  enabled: true
  interval_ms: 10              # 10ms resolution (high overhead)
  filter:
    - include: "*"
    
hw_counters:
  enabled: true
  sample_interval_us: 100      # 100µs sampling (high overhead)
  filter:
    - include: "*"             # All available counters
```

### Profile Selection

```c
// Select profile at runtime
exo_set_collection_profile("standard");

// Or via environment variable
// EXO_COLLECTION_PROFILE=detailed ./my_benchmark

// Or programmatically
exo_collection_config_t config;
exo_load_profile("standard", &config);
config.traces.sampling_rate = 0.5;  // Override specific setting
exo_apply_collection_config(&config);
```

---

## Data Normalization

### Canonical Internal Format

All collected data is normalized to an internal canonical format before storage:

```protobuf
// exo_trace.proto - Internal canonical format

message FrameTrace {
    // Identity
    string trace_id = 1;              // W3C trace ID (hex)
    string hierarchical_id = 2;       // "run/soc/model/frame"
    uint32 frame_index = 3;
    
    // Context references
    string run_id = 4;
    string soc_id = 5;
    string model_id = 6;
    
    // Timing
    uint64 start_time_ns = 7;
    uint64 end_time_ns = 8;
    
    // Content
    repeated Span spans = 10;
    repeated Event events = 11;
    repeated MetricSample metrics = 12;
    repeated HWCounterSample hw_counters = 13;
    
    // Aggregated statistics (always computed)
    FrameStats stats = 20;
}

message Span {
    bytes span_id = 1;                // 8 bytes
    bytes parent_span_id = 2;         // 8 bytes, zero if root
    
    string name = 3;
    SpanKind kind = 4;
    
    uint64 start_time_ns = 5;
    uint64 end_time_ns = 6;
    
    SpanStatus status = 7;
    string status_message = 8;
    
    repeated Attribute attributes = 9;
    repeated SpanEvent events = 10;
    repeated SpanLink links = 11;
    
    uint32 track_id = 12;
}

message Event {
    uint64 timestamp_ns = 1;
    string name = 2;
    EventType type = 3;
    uint32 track_id = 4;
    bytes span_id = 5;                // Associated span, if any
    repeated Attribute attributes = 6;
}

message MetricSample {
    uint64 timestamp_ns = 1;
    string name = 2;
    MetricType type = 3;
    
    oneof value {
        int64 counter_value = 4;
        double gauge_value = 5;
        HistogramValue histogram = 6;
    }
    
    repeated Label labels = 7;
}

message HWCounterSample {
    uint64 timestamp_ns = 1;
    string name = 2;
    int64 value = 3;
    uint64 delta_ns = 4;              // Time since last sample
}

message FrameStats {
    // Timing
    double latency_ms = 1;
    double preprocess_ms = 2;
    double inference_ms = 3;
    double postprocess_ms = 4;
    
    // Operator breakdown
    repeated OperatorStat operator_stats = 5;
    
    // Resource utilization (averages)
    double avg_npu_utilization = 10;
    double avg_memory_bandwidth = 11;
    double avg_dma_utilization = 12;
}

message OperatorStat {
    string op_name = 1;
    string op_type = 2;
    double duration_us = 3;
    double percentage = 4;            // % of total time
}
```

### Conversion to Standard Formats

```python
class TraceNormalizer:
    """Convert internal format to standard formats."""
    
    def to_perfetto(self, frame_trace: FrameTrace) -> bytes:
        """Convert to Perfetto protobuf format."""
        trace = perfetto_trace_pb2.Trace()
        
        # Add track descriptors
        for track_id, track_name in self.tracks.items():
            packet = trace.packet.add()
            track_desc = packet.track_descriptor
            track_desc.uuid = track_id
            track_desc.name = track_name
        
        # Add spans as slice events
        for span in frame_trace.spans:
            # Begin event
            packet = trace.packet.add()
            packet.timestamp = span.start_time_ns
            event = packet.track_event
            event.type = perfetto_trace_pb2.TrackEvent.TYPE_SLICE_BEGIN
            event.track_uuid = span.track_id
            event.name = span.name
            
            # Add attributes as debug annotations
            for attr in span.attributes:
                ann = event.debug_annotation.add()
                ann.name = attr.key
                self._set_annotation_value(ann, attr)
            
            # End event
            packet = trace.packet.add()
            packet.timestamp = span.end_time_ns
            event = packet.track_event
            event.type = perfetto_trace_pb2.TrackEvent.TYPE_SLICE_END
            event.track_uuid = span.track_id
        
        # Add counter tracks for HW counters
        for sample in frame_trace.hw_counters:
            packet = trace.packet.add()
            packet.timestamp = sample.timestamp_ns
            counter = packet.track_event
            counter.type = perfetto_trace_pb2.TrackEvent.TYPE_COUNTER
            counter.counter_value = sample.value
            counter.track_uuid = self._get_counter_track(sample.name)
        
        return trace.SerializeToString()
    
    def to_otlp(self, frame_trace: FrameTrace) -> ExportTraceServiceRequest:
        """Convert to OTLP format."""
        request = ExportTraceServiceRequest()
        
        resource_spans = request.resource_spans.add()
        self._set_resource(resource_spans.resource, frame_trace)
        
        scope_spans = resource_spans.scope_spans.add()
        scope_spans.scope.name = "exo"
        scope_spans.scope.version = EXO_VERSION
        
        for span in frame_trace.spans:
            otlp_span = scope_spans.spans.add()
            otlp_span.trace_id = bytes.fromhex(frame_trace.trace_id)
            otlp_span.span_id = span.span_id
            otlp_span.parent_span_id = span.parent_span_id
            otlp_span.name = span.name
            otlp_span.kind = self._convert_span_kind(span.kind)
            otlp_span.start_time_unix_nano = span.start_time_ns
            otlp_span.end_time_unix_nano = span.end_time_ns
            
            for attr in span.attributes:
                kv = otlp_span.attributes.add()
                kv.key = attr.key
                self._set_any_value(kv.value, attr)
        
        return request
    
    def to_chrome_trace(self, frame_trace: FrameTrace) -> dict:
        """Convert to Chrome Trace Event format."""
        events = []
        
        for span in frame_trace.spans:
            # Complete event (X)
            events.append({
                "name": span.name,
                "cat": self._get_category(span),
                "ph": "X",
                "ts": span.start_time_ns / 1000,  # microseconds
                "dur": (span.end_time_ns - span.start_time_ns) / 1000,
                "pid": 1,
                "tid": span.track_id,
                "args": {attr.key: self._attr_value(attr) for attr in span.attributes}
            })
        
        for event in frame_trace.events:
            events.append({
                "name": event.name,
                "cat": event.type,
                "ph": "i",  # instant
                "ts": event.timestamp_ns / 1000,
                "pid": 1,
                "tid": event.track_id,
                "s": "t",  # thread scope
            })
        
        return {
            "traceEvents": events,
            "metadata": {
                "trace_id": frame_trace.trace_id,
                "frame_index": frame_trace.frame_index,
            }
        }
```

---

## Implementation

### API Summary

```c
//=============================================================================
// Initialization
//=============================================================================

exo_status_t exo_init(const exo_config_t* config);
exo_status_t exo_shutdown(void);

// Load user instrumentation config
exo_status_t exo_load_instrumentation_config(const char* path);

// Set collection profile
exo_status_t exo_set_collection_profile(const char* profile_name);

//=============================================================================
// Context Management
//=============================================================================

// Set run-level context (once per benchmark run)
exo_status_t exo_set_run_context(const exo_run_context_t* ctx);

// Set SoC context (once per SoC)
exo_status_t exo_set_soc_context(const exo_soc_context_t* ctx);

// Set model context (once per model)
exo_status_t exo_set_model_context(const exo_model_context_t* ctx);

//=============================================================================
// Frame Tracing
//=============================================================================

// Begin a new frame trace
exo_frame_t* exo_frame_begin(uint32_t frame_index);

// End frame trace and export
exo_status_t exo_frame_end(exo_frame_t* frame);

// Within a frame: create spans
exo_span_t* exo_span_start(const char* name, exo_track_id_t track);
exo_status_t exo_span_end(exo_span_t* span);

// Record events
exo_status_t exo_record_event(const char* name, exo_track_id_t track);

//=============================================================================
// Metrics
//=============================================================================

// Register custom metrics
exo_status_t exo_register_metrics(const exo_metric_def_t* defs, size_t count);

// Get metric handles
exo_counter_t* exo_get_counter(const char* name);
exo_gauge_t* exo_get_gauge(const char* name);
exo_histogram_t* exo_get_histogram(const char* name);

// Record metric values
exo_status_t exo_counter_add(exo_counter_t* counter, int64_t delta, 
                              const exo_label_t* labels, size_t label_count);
exo_status_t exo_gauge_set(exo_gauge_t* gauge, double value,
                            const exo_label_t* labels, size_t label_count);
exo_status_t exo_histogram_record(exo_histogram_t* hist, double value,
                                   const exo_label_t* labels, size_t label_count);

//=============================================================================
// Hardware Counters
//=============================================================================

// Register HW counters (platform-specific)
exo_status_t exo_register_hw_counters(const exo_hw_counter_config_t* configs, 
                                       size_t count);

// Manual sampling (if not using automatic sampling)
exo_status_t exo_sample_hw_counters(void);

//=============================================================================
// Export Control
//=============================================================================

// Force flush all buffered data
exo_status_t exo_flush(void);

// Get collection statistics
exo_status_t exo_get_stats(exo_collection_stats_t* stats);
```

### Usage Example

```c
#include <exo/exo.h>

int main() {
    // Initialize with config file
    exo_config_t config = {
        .service_name = "ai_benchmark",
        .instrumentation_config_path = "exo_config.yaml",
    };
    exo_init(&config);
    
    // Set run context
    exo_set_run_context(&(exo_run_context_t){
        .run_id = "ci-2026-01-13-abc123",
        .git_commit = "abc123",
        .timestamp = time(NULL),
    });
    
    // For each SoC...
    for (int s = 0; s < num_socs; s++) {
        exo_set_soc_context(&(exo_soc_context_t){
            .soc_id = socs[s].id,
            .soc_name = socs[s].name,
            .firmware_version = socs[s].fw_version,
        });
        
        // For each model...
        for (int m = 0; m < num_models; m++) {
            exo_set_model_context(&(exo_model_context_t){
                .model_id = models[m].id,
                .model_name = models[m].name,
                .model_format = models[m].format,
            });
            
            // For each frame...
            for (int f = 0; f < num_frames; f++) {
                exo_frame_t* frame = exo_frame_begin(f);
                
                // Preprocessing
                exo_span_t* preprocess = exo_span_start("preprocess", EXO_TRACK_CPU);
                do_preprocess(&input);
                exo_span_end(preprocess);
                
                // Inference
                exo_span_t* inference = exo_span_start("inference", EXO_TRACK_NPU);
                run_inference(model, &input, &output);
                exo_span_end(inference);
                
                // Postprocessing
                exo_span_t* postprocess = exo_span_start("postprocess", EXO_TRACK_CPU);
                do_postprocess(&output);
                exo_span_end(postprocess);
                
                // Record custom metrics
                exo_gauge_set(queue_depth_gauge, get_queue_depth(), NULL, 0);
                
                exo_frame_end(frame);
            }
        }
    }
    
    exo_flush();
    exo_shutdown();
    return 0;
}
```

---

## Summary

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Two-layer format** | Internal canonical model + export to Perfetto/OTLP/Chrome |
| **Hierarchical IDs** | Natural mapping to CI structure, easy querying |
| **Sampling support** | Essential for managing data volume |
| **User-defined metrics** | Extensibility without core changes |
| **Collection profiles** | Different detail levels for different use cases |
| **Track-based model** | Excellent timeline visualization in Perfetto |
| **Hardware abstraction** | Graceful handling of different SoC capabilities |

### Trade-offs

| Trade-off | Choice | Alternative |
|-----------|--------|-------------|
| Format | Perfetto + OTLP | Single format (less compatible) |
| Sampling | Configurable | Always-on (too expensive) |
| Metrics | Schema-based registry | Free-form (chaos) |
| HW counters | Best-effort with caps | Require all or nothing |

### Next Steps

1. Define detailed storage schema (next document)
2. Implement libexo core with this design
3. Build collector service
4. Create validation suite
