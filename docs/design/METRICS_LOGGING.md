# Metrics and Logging Design

**OpenTelemetry-Compatible Metrics and Logs for Firmware**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Metrics API](#metrics-api)
3. [Logging API](#logging-api)
4. [Correlation](#correlation)
5. [Export Formats](#export-formats)
6. [AI-Specific Metrics](#ai-specific-metrics)

---

## Overview

Exo provides OpenTelemetry-compatible metrics and logging APIs alongside tracing. All three signals share:

- **Common resource attributes** - Service name, version, device ID
- **Trace correlation** - Logs and metrics linked to trace context
- **Pluggable exporters** - OTLP, Prometheus, file, custom
- **Embedded-friendly implementation** - Fixed memory, no malloc

### Signal Correlation

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Request Processing                            │
│                                                                      │
│  Trace ─────────────────────────────────────────────────────────    │
│  │                                                                   │
│  ├── Span: inference ─────────────────────────────────────────      │
│  │   trace_id: abc123                                                │
│  │   │                                                               │
│  │   ├── Metric: inference_latency_ms = 45.2                        │
│  │   │   exemplar.trace_id: abc123  ◄── Linked to trace             │
│  │   │                                                               │
│  │   ├── Log: "Model loaded successfully"                           │
│  │   │   trace_id: abc123  ◄── Linked to trace                      │
│  │   │   span_id: def456                                            │
│  │   │                                                               │
│  │   └── Metric: memory_usage_bytes = 1048576                       │
│  │       exemplar.trace_id: abc123                                  │
│  │                                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Metrics API

### OpenTelemetry Metrics Mapping

| OTel Concept | Exo Implementation | Notes |
|--------------|-------------------|-------|
| MeterProvider | `exo_meter_provider_t` | Pluggable |
| Meter | `exo_meter_t` | Named, versioned |
| Counter | `exo_counter_t` | Monotonic, additive |
| UpDownCounter | `exo_updown_counter_t` | Non-monotonic |
| Gauge | `exo_gauge_t` | Point-in-time value |
| Histogram | `exo_histogram_t` | Distribution |
| Attributes | Labels on data points | |
| Exemplars | Trace context links | Optional |

### Meter Provider API

```c
//=============================================================================
// MeterProvider - manages Meter instances
//=============================================================================

// Get the global meter provider
exo_meter_provider_t* exo_get_meter_provider(void);

// Set a custom meter provider
exo_status_t exo_set_meter_provider(exo_meter_provider_t* provider);

// Get a meter from the provider
exo_meter_t* exo_get_meter(const char* name, const char* version);
```

### Counter API

```c
//=============================================================================
// Counter - monotonically increasing value
//=============================================================================

// Create a counter
exo_counter_t* exo_meter_create_counter(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit  // e.g., "requests", "bytes", "1" for dimensionless
);

// Add to counter (always positive)
void exo_counter_add(
    exo_counter_t* counter,
    int64_t value,
    const exo_attribute_t* attributes,
    size_t attribute_count
);

// Convenience: add without attributes
void exo_counter_inc(exo_counter_t* counter);
void exo_counter_add_simple(exo_counter_t* counter, int64_t value);
```

### UpDownCounter API

```c
//=============================================================================
// UpDownCounter - can increase or decrease
//=============================================================================

exo_updown_counter_t* exo_meter_create_updown_counter(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit
);

// Add (can be positive or negative)
void exo_updown_counter_add(
    exo_updown_counter_t* counter,
    int64_t value,
    const exo_attribute_t* attributes,
    size_t attribute_count
);
```

### Gauge API

```c
//=============================================================================
// Gauge - point-in-time value
//=============================================================================

// Synchronous gauge (set value directly)
exo_gauge_t* exo_meter_create_gauge(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit
);

void exo_gauge_set(
    exo_gauge_t* gauge,
    double value,
    const exo_attribute_t* attributes,
    size_t attribute_count
);

// Asynchronous gauge (callback-based)
typedef double (*exo_gauge_callback_t)(void* user_data);

exo_gauge_t* exo_meter_create_observable_gauge(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit,
    exo_gauge_callback_t callback,
    void* user_data
);
```

### Histogram API

```c
//=============================================================================
// Histogram - value distribution
//=============================================================================

// Create histogram with default bucket boundaries
exo_histogram_t* exo_meter_create_histogram(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit
);

// Create histogram with explicit bucket boundaries
exo_histogram_t* exo_meter_create_histogram_with_buckets(
    exo_meter_t* meter,
    const char* name,
    const char* description,
    const char* unit,
    const double* bucket_boundaries,
    size_t bucket_count
);

// Record a value
void exo_histogram_record(
    exo_histogram_t* histogram,
    double value,
    const exo_attribute_t* attributes,
    size_t attribute_count
);
```

### Usage Examples

```c
// Initialize metrics
exo_meter_t* meter = exo_get_meter("inference-engine", "1.0.0");

// Counter example: track inference count
exo_counter_t* inference_counter = exo_meter_create_counter(
    meter,
    "inference.count",
    "Number of inference requests",
    "requests"
);

// Histogram example: track inference latency
exo_histogram_t* latency_hist = exo_meter_create_histogram(
    meter,
    "inference.latency",
    "Inference latency distribution",
    "ms"
);

// Gauge example: track memory usage
exo_gauge_t* memory_gauge = exo_meter_create_gauge(
    meter,
    "memory.used",
    "Current memory usage",
    "bytes"
);

// Record metrics during inference
void run_inference(model_t* model, input_t* input) {
    exo_attribute_t attrs[] = {
        EXO_ATTR_STRING("model.name", model->name),
    };
    
    uint64_t start = exo_clock_now_ns();
    
    // ... run inference ...
    
    uint64_t elapsed_ms = (exo_clock_now_ns() - start) / 1000000;
    
    exo_counter_add(inference_counter, 1, attrs, 1);
    exo_histogram_record(latency_hist, elapsed_ms, attrs, 1);
    exo_gauge_set(memory_gauge, get_memory_usage(), attrs, 1);
}
```

---

## Logging API

### OpenTelemetry Logging Mapping

| OTel Concept | Exo Implementation | Notes |
|--------------|-------------------|-------|
| LoggerProvider | `exo_logger_provider_t` | Pluggable |
| Logger | `exo_logger_t` | Named |
| LogRecord | `exo_log_record_t` | Individual log entry |
| SeverityNumber | `exo_log_level_t` | TRACE to FATAL |
| Body | String message | |
| Attributes | Key-value pairs | |
| TraceContext | Automatic correlation | |

### Severity Levels

```c
typedef enum {
    EXO_LOG_TRACE = 1,   // OTel: TRACE (1-4)
    EXO_LOG_DEBUG = 5,   // OTel: DEBUG (5-8)
    EXO_LOG_INFO = 9,    // OTel: INFO (9-12)
    EXO_LOG_WARN = 13,   // OTel: WARN (13-16)
    EXO_LOG_ERROR = 17,  // OTel: ERROR (17-20)
    EXO_LOG_FATAL = 21,  // OTel: FATAL (21-24)
} exo_log_level_t;
```

### Logger API

```c
//=============================================================================
// LoggerProvider and Logger
//=============================================================================

// Get global logger provider
exo_logger_provider_t* exo_get_logger_provider(void);

// Set custom logger provider
exo_status_t exo_set_logger_provider(exo_logger_provider_t* provider);

// Get a logger
exo_logger_t* exo_get_logger(const char* name);

//=============================================================================
// Logging Functions
//=============================================================================

// Log with level
void exo_log(
    exo_logger_t* logger,
    exo_log_level_t level,
    const char* message
);

// Log with attributes
void exo_log_with_attrs(
    exo_logger_t* logger,
    exo_log_level_t level,
    const char* message,
    const exo_attribute_t* attributes,
    size_t attribute_count
);

// Convenience macros (use default logger)
#define EXO_LOG_TRACE(msg) exo_log(exo_get_default_logger(), EXO_LOG_TRACE, msg)
#define EXO_LOG_DEBUG(msg) exo_log(exo_get_default_logger(), EXO_LOG_DEBUG, msg)
#define EXO_LOG_INFO(msg)  exo_log(exo_get_default_logger(), EXO_LOG_INFO, msg)
#define EXO_LOG_WARN(msg)  exo_log(exo_get_default_logger(), EXO_LOG_WARN, msg)
#define EXO_LOG_ERROR(msg) exo_log(exo_get_default_logger(), EXO_LOG_ERROR, msg)
#define EXO_LOG_FATAL(msg) exo_log(exo_get_default_logger(), EXO_LOG_FATAL, msg)

// Printf-style logging (allocates temporarily)
void exo_logf(
    exo_logger_t* logger,
    exo_log_level_t level,
    const char* format,
    ...
);

#define EXO_LOG_INFOF(fmt, ...) \
    exo_logf(exo_get_default_logger(), EXO_LOG_INFO, fmt, __VA_ARGS__)
```

### Structured Logging

```c
// Structured log with attributes
exo_attribute_t log_attrs[] = {
    EXO_ATTR_STRING("model.name", "resnet50"),
    EXO_ATTR_INT("batch_size", 32),
    EXO_ATTR_DOUBLE("accuracy", 0.95),
};

exo_log_with_attrs(
    logger,
    EXO_LOG_INFO,
    "Inference completed",
    log_attrs,
    3
);

// Output (JSON format):
// {
//   "timestamp": "2026-01-13T12:00:00Z",
//   "severity": "INFO",
//   "body": "Inference completed",
//   "attributes": {
//     "model.name": "resnet50",
//     "batch_size": 32,
//     "accuracy": 0.95
//   },
//   "trace_id": "abc123...",  // Auto-attached from context
//   "span_id": "def456..."
// }
```

### Trace-Log Correlation

```c
// Logs automatically include trace context from active span
exo_span_t* span = exo_start_span("process_request");

// This log will include trace_id and span_id from the active span
EXO_LOG_INFO("Starting request processing");

// ... processing ...

exo_span_end(span);
```

---

## Correlation

### Exemplars (Metric-Trace Links)

```c
// Enable exemplars on histograms
exo_histogram_config_t config = {
    .exemplars_enabled = true,
    .max_exemplars_per_bucket = 1,
};
exo_histogram_t* hist = exo_meter_create_histogram_with_config(
    meter, "latency", "Latency", "ms", &config
);

// When recording, current trace context is captured as exemplar
exo_span_t* span = exo_start_span("operation");
exo_histogram_record(hist, 45.2, NULL, 0);
// Exemplar includes: trace_id from span, recorded value, timestamp
exo_span_end(span);
```

### Cross-Signal Queries

With correlated telemetry, you can:

```
1. Find slow traces: Query histogram for high-latency exemplars
   → Jump to trace using trace_id

2. Debug errors: Find error logs
   → Jump to trace using trace_id
   → See full request flow

3. Correlate metrics spikes: Find metric anomaly
   → Query exemplars at that time
   → Investigate traces
```

---

## Export Formats

### OTLP (Default)

```c
// OTLP metric exporter
exo_metric_exporter_t* exporter = exo_otlp_metric_exporter_create(&(exo_otlp_exporter_options_t){
    .endpoint = "http://collector:4317",
});

// OTLP log exporter
exo_log_exporter_t* log_exporter = exo_otlp_log_exporter_create(&(exo_otlp_exporter_options_t){
    .endpoint = "http://collector:4317",
});
```

### Prometheus (Metrics)

```c
// Prometheus exposition format
exo_metric_exporter_t* prom = exo_prometheus_exporter_create(&(exo_prometheus_options_t){
    .port = 9090,
    .path = "/metrics",
});

// Output example:
// # HELP inference_count Number of inference requests
// # TYPE inference_count counter
// inference_count{model="resnet50"} 1234
//
// # HELP inference_latency Inference latency distribution
// # TYPE inference_latency histogram
// inference_latency_bucket{le="10"} 100
// inference_latency_bucket{le="50"} 500
// inference_latency_bucket{le="100"} 950
// inference_latency_bucket{le="+Inf"} 1000
// inference_latency_sum 45678
// inference_latency_count 1000
```

### File Export (Offline)

```c
// Export to files for offline analysis
exo_metric_exporter_t* file = exo_file_metric_exporter_create(
    "/data/metrics.jsonl"
);

exo_log_exporter_t* log_file = exo_file_log_exporter_create(
    "/data/logs.jsonl"
);
```

---

## AI-Specific Metrics

### Pre-defined AI Metrics

```c
#include <exo/ai_metrics.h>

// Initialize AI metrics suite
exo_ai_metrics_t* ai_metrics = exo_ai_metrics_create(meter);

// Inference metrics
exo_ai_metrics_record_inference(ai_metrics, &(exo_ai_inference_record_t){
    .model_name = "resnet50",
    .batch_size = 32,
    .latency_ms = 45.2,
    .success = true,
});

// Operator-level metrics
exo_ai_metrics_record_op(ai_metrics, &(exo_ai_op_record_t){
    .op_type = EXO_AI_OP_CONV2D,
    .duration_us = 1234,
    .flops = 1000000,
    .memory_bytes = 4096,
});

// Hardware utilization
exo_ai_metrics_record_hw(ai_metrics, &(exo_ai_hw_record_t){
    .device = "npu",
    .utilization = 0.85,
    .power_mw = 500,
    .temperature_c = 45.5,
});
```

### AI Metrics Semantic Conventions

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `ai.inference.count` | Counter | requests | Total inference count |
| `ai.inference.latency` | Histogram | ms | Inference latency |
| `ai.inference.errors` | Counter | errors | Failed inferences |
| `ai.op.duration` | Histogram | us | Per-op duration |
| `ai.op.flops` | Counter | flops | Floating point ops |
| `ai.memory.allocated` | Gauge | bytes | Currently allocated |
| `ai.memory.peak` | Gauge | bytes | Peak allocation |
| `ai.hw.utilization` | Gauge | ratio | HW utilization (0-1) |
| `ai.hw.power` | Gauge | mW | Power consumption |
| `ai.hw.temperature` | Gauge | Celsius | Temperature |
| `ai.throughput.tokens` | Counter | tokens | Tokens processed (LLM) |
| `ai.throughput.images` | Counter | images | Images processed |

---

## Configuration

```c
// Metrics configuration
typedef struct exo_metrics_config {
    bool enabled;                           // Default: true
    size_t max_instruments;                 // Default: 64
    size_t max_attributes_per_point;        // Default: 8
    uint32_t export_interval_ms;            // Default: 60000
    exo_metric_exporter_t* exporter;        // Default: OTLP
} exo_metrics_config_t;

// Logging configuration
typedef struct exo_logging_config {
    bool enabled;                           // Default: true
    exo_log_level_t min_level;             // Default: INFO
    size_t max_log_buffer;                 // Default: 256 records
    exo_log_exporter_t* exporter;          // Default: OTLP
} exo_logging_config_t;

// Include in main config
exo_config_t config = {
    .service_name = "inference-engine",
    .metrics = {
        .enabled = true,
        .export_interval_ms = 30000,
    },
    .logging = {
        .enabled = true,
        .min_level = EXO_LOG_DEBUG,
    },
};
```

---

## Pluggable Interfaces

### Metric Exporter Interface

```c
typedef struct exo_metric_exporter_vtable {
    exo_status_t (*export)(
        void* impl,
        const exo_metric_data_t* metrics,
        size_t count
    );
    exo_status_t (*force_flush)(void* impl, uint32_t timeout_ms);
    exo_status_t (*shutdown)(void* impl);
} exo_metric_exporter_vtable_t;

typedef struct exo_metric_exporter {
    const exo_metric_exporter_vtable_t* vtable;
    void* impl;
} exo_metric_exporter_t;
```

### Log Exporter Interface

```c
typedef struct exo_log_exporter_vtable {
    exo_status_t (*export)(
        void* impl,
        const exo_log_record_t* logs,
        size_t count
    );
    exo_status_t (*force_flush)(void* impl, uint32_t timeout_ms);
    exo_status_t (*shutdown)(void* impl);
} exo_log_exporter_vtable_t;

typedef struct exo_log_exporter {
    const exo_log_exporter_vtable_t* vtable;
    void* impl;
} exo_log_exporter_t;
```

---

## Appendix: Standard Backend Compatibility

| Backend | Metrics | Logs |
|---------|---------|------|
| OpenTelemetry Collector | ✅ OTLP | ✅ OTLP |
| Prometheus | ✅ Prometheus format | N/A |
| Grafana | ✅ Via Prometheus/OTLP | ✅ Via Loki/OTLP |
| Loki | N/A | ✅ Via OTLP |
| Datadog | ✅ Via OTel Collector | ✅ Via OTel Collector |
| Elastic | ✅ Via OTel Collector | ✅ Via OTel Collector |
| Custom | ✅ Exporter interface | ✅ Exporter interface |
