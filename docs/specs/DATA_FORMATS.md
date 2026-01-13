# Data Formats and Protocols Specification

**Wire Formats, Storage Formats, and Protocol Support**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Supported Formats](#supported-formats)
3. [Perfetto Format](#perfetto-format)
4. [OpenTelemetry Protocol (OTLP)](#opentelemetry-protocol-otlp)
5. [Chrome Trace Event Format](#chrome-trace-event-format)
6. [Exo Binary Format](#exo-binary-format)
7. [Format Selection Guide](#format-selection-guide)
8. [Conversion and Interoperability](#conversion-and-interoperability)

---

## Overview

Exo supports multiple data formats to ensure interoperability with existing tools and flexibility for different use cases:

| Format | Use Case | Visualization | Standard |
|--------|----------|---------------|----------|
| **Perfetto** | System tracing, timeline visualization | Perfetto UI | Google Open Source |
| **OTLP** | Distributed tracing backends | Jaeger, Tempo, Zipkin | OpenTelemetry |
| **Chrome Trace** | Simple timeline visualization | Chrome, Perfetto | Google |
| **Exo Binary** | Minimal overhead, embedded | Exo tools, converts to others | Exo-native |

### Design Principle: Format Agnostic Core

```
┌─────────────────────────────────────────────────────────────────┐
│                     Application Code                             │
│                                                                  │
│  exo_span_t* span = exo_start_span("my_operation");             │
│  // ... work ...                                                 │
│  exo_span_end(span);                                            │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Internal Representation                        │
│                    (Format-agnostic spans)                        │
└──────────────────────────────┬───────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Perfetto        │ │ OTLP            │ │ Chrome Trace    │
│ Exporter        │ │ Exporter        │ │ Exporter        │
│                 │ │                 │ │                 │
│ → .perfetto     │ │ → OTLP/gRPC     │ │ → .json         │
│ → .pftrace      │ │ → OTLP/HTTP     │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## Supported Formats

### Primary Formats

| Format | Export | Import | Visualization |
|--------|--------|--------|---------------|
| Perfetto Protobuf | ✅ | ✅ | [Perfetto UI](https://ui.perfetto.dev) |
| OTLP Protobuf | ✅ | ✅ | Jaeger, Tempo, etc. |
| OTLP JSON | ✅ | ✅ | Any OTLP backend |
| Chrome Trace JSON | ✅ | ✅ | Chrome, Perfetto UI |

### Secondary Formats

| Format | Export | Import | Use Case |
|--------|--------|--------|----------|
| Exo Binary | ✅ | ✅ | Minimal size, embedded |
| Jaeger Thrift | ✅ | ❌ | Legacy Jaeger |
| Zipkin JSON | ✅ | ❌ | Zipkin backend |
| Prometheus | ✅ (metrics) | ❌ | Prometheus/Grafana |

---

## Perfetto Format

### Why Perfetto?

Perfetto is an ideal format for firmware/SoC observability:

1. **System-level design** - Built for OS/hardware tracing
2. **Efficient binary format** - Protobuf-based, compact
3. **Excellent visualization** - Free, powerful web UI
4. **Track-based model** - Maps well to hardware units (CPU, NPU, DMA)
5. **Counter support** - Native support for performance counters
6. **Flow events** - Show data flow across components
7. **Widely adopted** - Android, Chrome, Linux perf

### Perfetto Concepts Mapping

| Perfetto Concept | Exo Concept | Notes |
|------------------|-------------|-------|
| Trace | Session | Collection of all data |
| Track | Hardware unit / Thread | Parallel execution lanes |
| TrackEvent (slice) | Span | Timed operation |
| TrackEvent (instant) | Event | Point-in-time annotation |
| Counter | Metric (Gauge) | Time-series values |
| Flow | Span Link | Causal connection |
| ProcessDescriptor | Resource | Service identification |
| ThreadDescriptor | Context | Execution context |

### Track Organization for AI Firmware

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Perfetto UI Visualization                                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│ ▼ inference_engine (Process)                                            │
│   │                                                                      │
│   ├─ ▼ CPU Thread 0                                                     │
│   │    ├──[model_load]──┤                                               │
│   │    ├──[preprocess]──┼──────────────────[postprocess]──┤             │
│   │                                                                      │
│   ├─ ▼ NPU                                                              │
│   │    │                ├──[conv2d]──┼──[relu]──┼──[pool]──┤            │
│   │                                                                      │
│   ├─ ▼ DMA                                                              │
│   │    │         ├─[in]─┤                              ├─[out]─┤        │
│   │                                                                      │
│   ├─ ▼ Counters                                                         │
│   │    ├─ NPU Utilization: ▁▂▅▇█▇▅▂▁                                    │
│   │    ├─ Memory Bandwidth: ▂▃▅▆▇▆▅▃▂                                   │
│   │    └─ Power (mW): ▁▂▄▆█▆▄▂▁                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Perfetto Protobuf Schema

Exo generates traces compatible with Perfetto's protobuf schema:

```protobuf
// Simplified Perfetto trace structure (actual schema is more complex)

message Trace {
  repeated TracePacket packet = 1;
}

message TracePacket {
  optional uint64 timestamp = 8;  // Nanoseconds
  
  oneof data {
    TrackEvent track_event = 11;
    TrackDescriptor track_descriptor = 60;
    ProcessDescriptor process_descriptor = 44;
    ThreadDescriptor thread_descriptor = 44;
    CounterValue counter_value = 62;
  }
  
  optional uint64 trusted_packet_sequence_id = 10;
}

message TrackEvent {
  enum Type {
    TYPE_SLICE_BEGIN = 1;
    TYPE_SLICE_END = 2;
    TYPE_INSTANT = 3;
    TYPE_COUNTER = 4;
  }
  
  optional Type type = 9;
  optional uint64 track_uuid = 11;
  optional string name = 23;
  repeated DebugAnnotation debug_annotation = 4;  // Attributes
  
  // For flow events (span links)
  optional uint64 flow_id = 36;
}

message TrackDescriptor {
  optional uint64 uuid = 1;
  optional string name = 2;
  optional ProcessDescriptor process = 3;
  optional ThreadDescriptor thread = 4;
  optional CounterDescriptor counter = 8;
}
```

### Exo Perfetto API

```c
#include <exo/exporters/perfetto.h>

//=============================================================================
// Perfetto Exporter Configuration
//=============================================================================

typedef struct exo_perfetto_config {
    // Output destination
    const char* output_path;          // File path (e.g., "trace.perfetto")
    int output_fd;                    // Or file descriptor
    void* output_buffer;              // Or memory buffer
    size_t output_buffer_size;
    
    // Track configuration
    bool auto_create_tracks;          // Create tracks for threads/HW units
    bool include_counters;            // Include counter tracks
    bool include_flow_events;         // Include flow events for links
    
    // Compression
    bool enable_compression;          // zlib compression
    
    // Batching
    size_t packet_buffer_size;        // Default: 64KB
    uint32_t flush_interval_ms;       // Default: 1000ms
    
} exo_perfetto_config_t;

//=============================================================================
// Perfetto Exporter API
//=============================================================================

// Create Perfetto exporter
exo_span_exporter_t* exo_perfetto_exporter_create(
    const exo_perfetto_config_t* config
);

// Create Perfetto file exporter (convenience)
exo_span_exporter_t* exo_perfetto_file_exporter_create(
    const char* path
);

// Create Perfetto memory exporter (for embedded)
exo_span_exporter_t* exo_perfetto_memory_exporter_create(
    void* buffer,
    size_t size,
    size_t* bytes_written  // Output: actual size
);

//=============================================================================
// Perfetto Tracks API
//=============================================================================

// Explicitly define custom tracks for hardware units
typedef struct exo_perfetto_track {
    uint64_t uuid;
    const char* name;
    exo_perfetto_track_type_t type;  // THREAD, COUNTER, ASYNC
} exo_perfetto_track_t;

exo_status_t exo_perfetto_register_track(
    const exo_perfetto_track_t* track
);

// Pre-defined tracks for AI hardware
exo_status_t exo_perfetto_register_hw_tracks(void);
// Creates tracks: CPU, NPU, GPU, DMA, etc.

//=============================================================================
// Counter Tracks (for metrics)
//=============================================================================

// Register a counter track
exo_perfetto_counter_t* exo_perfetto_counter_create(
    const char* name,
    const char* unit,
    exo_perfetto_counter_type_t type  // INT64, DOUBLE
);

// Record counter value (appears as time-series in Perfetto)
void exo_perfetto_counter_set(
    exo_perfetto_counter_t* counter,
    int64_t value
);

void exo_perfetto_counter_set_double(
    exo_perfetto_counter_t* counter,
    double value
);
```

### Perfetto Usage Example

```c
#include <exo/exo.h>
#include <exo/exporters/perfetto.h>

int main() {
    // Initialize Exo with Perfetto exporter
    exo_span_exporter_t* perfetto = exo_perfetto_file_exporter_create(
        "inference_trace.perfetto"
    );
    
    exo_config_t config = {
        .service_name = "ai_inference",
        .tracing = {
            .exporter = perfetto,
        },
    };
    exo_init(&config);
    
    // Register hardware tracks
    exo_perfetto_register_hw_tracks();
    
    // Create counter tracks for hardware metrics
    exo_perfetto_counter_t* npu_util = exo_perfetto_counter_create(
        "NPU Utilization", "%", EXO_PERFETTO_COUNTER_DOUBLE
    );
    exo_perfetto_counter_t* mem_bw = exo_perfetto_counter_create(
        "Memory Bandwidth", "GB/s", EXO_PERFETTO_COUNTER_DOUBLE
    );
    
    // Run inference with tracing
    exo_span_t* inference = exo_start_span("inference");
    exo_span_set_attribute_string(inference, "model", "resnet50");
    
    // Track DMA transfer
    exo_span_t* dma_in = exo_start_span("dma_input");
    exo_span_set_attribute_string(dma_in, "perfetto.track", "DMA");
    // ... DMA transfer ...
    exo_span_end(dma_in);
    
    // Track NPU execution
    exo_span_t* npu = exo_start_span("npu_execute");
    exo_span_set_attribute_string(npu, "perfetto.track", "NPU");
    
    // Record hardware counters
    exo_perfetto_counter_set_double(npu_util, 0.85);
    exo_perfetto_counter_set_double(mem_bw, 25.6);
    
    // ... NPU execution ...
    exo_span_end(npu);
    
    exo_span_end(inference);
    
    // Shutdown flushes trace
    exo_shutdown();
    
    // Open trace.perfetto in https://ui.perfetto.dev
    return 0;
}
```

### Perfetto Trace Analysis

```python
# Python script to analyze Perfetto traces
from exo_analysis import PerfettoTrace

# Load trace
trace = PerfettoTrace.load("inference_trace.perfetto")

# Query slices (spans)
inference_spans = trace.query_slices("name == 'inference'")
for span in inference_spans:
    print(f"Inference: {span.duration_ms}ms")

# Get counter data
npu_util = trace.get_counter("NPU Utilization")
print(f"Average NPU utilization: {npu_util.mean():.1%}")

# Export to pandas for analysis
df = trace.to_dataframe()
```

### Perfetto System Integration

For deep system integration, Exo can write to Perfetto's traced daemon:

```c
// On Linux with Perfetto traced running
exo_span_exporter_t* perfetto = exo_perfetto_traced_exporter_create(
    "exo_session"  // Session name
);

// Traces go to system-wide Perfetto trace
// Combine with kernel traces, other processes
```

---

## OpenTelemetry Protocol (OTLP)

### OTLP Overview

OTLP is the standard protocol for OpenTelemetry, supporting:
- gRPC transport
- HTTP/protobuf transport
- HTTP/JSON transport

### OTLP Exporter Configuration

```c
#include <exo/exporters/otlp.h>

typedef struct exo_otlp_config {
    // Endpoint
    const char* endpoint;             // e.g., "http://localhost:4317"
    
    // Protocol
    exo_otlp_protocol_t protocol;     // GRPC, HTTP_PROTOBUF, HTTP_JSON
    
    // Headers (for auth, etc.)
    exo_string_pair_t headers[8];
    size_t header_count;
    
    // Compression
    exo_compression_t compression;    // NONE, GZIP
    
    // Timeouts
    uint32_t timeout_ms;
    
    // TLS (optional)
    const char* certificate_path;
    
} exo_otlp_config_t;

// Create exporters
exo_span_exporter_t* exo_otlp_span_exporter_create(const exo_otlp_config_t* config);
exo_metric_exporter_t* exo_otlp_metric_exporter_create(const exo_otlp_config_t* config);
exo_log_exporter_t* exo_otlp_log_exporter_create(const exo_otlp_config_t* config);

// File-based OTLP (for offline/embedded)
exo_span_exporter_t* exo_otlp_file_exporter_create(const char* path);
// Writes OTLP protobuf to file, can be sent later
```

### OTLP JSON Format

For debugging or simple tools, OTLP JSON is human-readable:

```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [
        {"key": "service.name", "value": {"stringValue": "inference-engine"}},
        {"key": "service.version", "value": {"stringValue": "1.0.0"}}
      ]
    },
    "scopeSpans": [{
      "scope": {
        "name": "exo",
        "version": "0.1.0"
      },
      "spans": [{
        "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
        "spanId": "00f067aa0ba902b7",
        "parentSpanId": "",
        "name": "inference",
        "kind": 1,
        "startTimeUnixNano": "1704067200000000000",
        "endTimeUnixNano": "1704067200045200000",
        "attributes": [
          {"key": "model.name", "value": {"stringValue": "resnet50"}},
          {"key": "batch_size", "value": {"intValue": "32"}}
        ],
        "status": {"code": 1}
      }]
    }]
  }]
}
```

---

## Chrome Trace Event Format

### Overview

Chrome Trace Event format is a simple JSON format supported by:
- Chrome DevTools (`chrome://tracing`)
- Perfetto UI
- Many other tools

It's less feature-rich than Perfetto or OTLP but very portable.

### Format Specification

```json
{
  "traceEvents": [
    {
      "name": "inference",
      "cat": "ai",
      "ph": "B",
      "ts": 1000000,
      "pid": 1,
      "tid": 1
    },
    {
      "name": "inference",
      "cat": "ai", 
      "ph": "E",
      "ts": 1045200,
      "pid": 1,
      "tid": 1,
      "args": {
        "model": "resnet50",
        "batch_size": 32
      }
    }
  ],
  "metadata": {
    "exo_version": "0.1.0",
    "service_name": "inference-engine"
  }
}
```

### Event Types

| Phase (ph) | Meaning | Exo Mapping |
|------------|---------|-------------|
| `B` | Begin | Span start |
| `E` | End | Span end |
| `X` | Complete | Span (single event) |
| `i` | Instant | Event |
| `C` | Counter | Gauge metric |
| `s` | Flow start | Span link start |
| `f` | Flow end | Span link end |
| `M` | Metadata | Resource attributes |

### Chrome Trace Exporter

```c
#include <exo/exporters/chrome_trace.h>

// Create Chrome trace exporter
exo_span_exporter_t* exo_chrome_trace_exporter_create(
    const char* path
);

// With options
typedef struct exo_chrome_trace_config {
    const char* output_path;
    bool pretty_print;              // Indented JSON
    bool include_metadata;          // Include header metadata
    size_t max_file_size;          // Rotate at this size
} exo_chrome_trace_config_t;

exo_span_exporter_t* exo_chrome_trace_exporter_create_with_config(
    const exo_chrome_trace_config_t* config
);
```

---

## Exo Binary Format

### Design Goals

For embedded systems with minimal resources:
- Minimal overhead (< 10 bytes per span for wire format)
- No text parsing needed
- Streamable (no random access required)
- Self-describing (version, schema)

### Binary Format Specification

```
┌─────────────────────────────────────────────────────────────┐
│ Exo Binary Trace Format                                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Header (16 bytes)                                            │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Magic: "EXO\0" (4 bytes)                               │  │
│ │ Version: uint16 (2 bytes)                              │  │
│ │ Flags: uint16 (2 bytes)                                │  │
│ │ Timestamp epoch: uint64 (8 bytes)                      │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ Records (variable length, repeated)                          │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Record type: uint8                                     │  │
│ │ Record length: uint16 (varint for efficiency)          │  │
│ │ Record data: [length] bytes                            │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Record Types

```c
typedef enum {
    EXO_RECORD_SPAN_START = 0x01,
    EXO_RECORD_SPAN_END = 0x02,
    EXO_RECORD_SPAN_COMPLETE = 0x03,  // Combined start+end
    EXO_RECORD_EVENT = 0x04,
    EXO_RECORD_ATTRIBUTE = 0x05,
    EXO_RECORD_COUNTER = 0x10,
    EXO_RECORD_LOG = 0x20,
    EXO_RECORD_RESOURCE = 0x80,
    EXO_RECORD_STRING_TABLE = 0x81,
} exo_record_type_t;
```

### Span Record Format

```
Span Complete Record:
┌─────────────────────────────────────────────────────────────┐
│ Type: 0x03 (1 byte)                                          │
│ Length: varint                                               │
│ Trace ID: 16 bytes                                           │
│ Span ID: 8 bytes                                             │
│ Parent Span ID: 8 bytes (0 if root)                          │
│ Name index: varint (into string table)                       │
│ Start time delta: varint (from epoch, in 100ns units)        │
│ Duration: varint (in 100ns units)                            │
│ Kind: uint8                                                  │
│ Status: uint8                                                │
│ Attribute count: uint8                                       │
│ Attributes: [see attribute encoding]                         │
└─────────────────────────────────────────────────────────────┘
```

### Binary Exporter

```c
#include <exo/exporters/binary.h>

// Create binary file exporter
exo_span_exporter_t* exo_binary_exporter_create(
    const char* path
);

// Create binary UART exporter
exo_span_exporter_t* exo_binary_uart_exporter_create(
    int fd,
    uint32_t baud_rate
);

// Create binary memory exporter (ring buffer)
exo_span_exporter_t* exo_binary_memory_exporter_create(
    void* buffer,
    size_t size
);
```

### Conversion Tools

```bash
# Convert binary to Perfetto
exo-convert input.exo --format perfetto -o output.perfetto

# Convert binary to OTLP JSON
exo-convert input.exo --format otlp-json -o output.json

# Convert binary to Chrome trace
exo-convert input.exo --format chrome -o output.json

# Stream conversion (for UART)
exo-convert --stream /dev/ttyUSB0 --format perfetto -o trace.perfetto
```

---

## Format Selection Guide

### Decision Matrix

| Scenario | Recommended Format | Reason |
|----------|-------------------|--------|
| Timeline visualization | **Perfetto** | Best UI, track support |
| Distributed tracing backend | **OTLP** | Standard, all backends |
| Simple debugging | **Chrome Trace** | Easy, universal |
| Minimal embedded | **Exo Binary** | Smallest overhead |
| Offline analysis | **Perfetto** or **OTLP** | Rich data model |
| Real-time streaming | **OTLP gRPC** | Efficient, streaming |
| Hardware counter correlation | **Perfetto** | Counter tracks |
| Cross-process tracing | **OTLP** | Distributed traces |

### Format Comparison

| Feature | Perfetto | OTLP | Chrome Trace | Exo Binary |
|---------|----------|------|--------------|------------|
| Binary format | ✅ Protobuf | ✅ Protobuf | ❌ JSON | ✅ Custom |
| Human readable | ❌ | ⚠️ JSON option | ✅ | ❌ |
| Tracks/threads | ✅ Native | ⚠️ Via attributes | ⚠️ pid/tid | ⚠️ Explicit |
| Counter support | ✅ Native | ✅ Metrics | ⚠️ C events | ✅ |
| Flow events | ✅ Native | ✅ Links | ✅ s/f events | ✅ |
| Compression | ✅ | ✅ gzip | ❌ | Optional |
| Size efficiency | High | Medium | Low | Highest |
| Tool support | Perfetto UI | Many backends | Chrome, Perfetto | Exo tools |

---

## Conversion and Interoperability

### Conversion Paths

```
                    ┌─────────────────┐
                    │   Exo Binary    │
                    │   (native)      │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Perfetto     │ │     OTLP        │ │  Chrome Trace   │
│                 │ │                 │ │                 │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         │                   │                   │
         ▼                   ▼                   │
┌─────────────────┐ ┌─────────────────┐          │
│  Perfetto UI    │ │  Jaeger/Tempo   │          │
│                 │ │  Zipkin/etc     │          │
└─────────────────┘ └─────────────────┘          │
                                                 │
                    ┌─────────────────┐          │
                    │  Chrome/        │◀─────────┘
                    │  Perfetto UI    │
                    └─────────────────┘
```

### Python Conversion Library

```python
from exo_analysis import TraceConverter

# Load any format
trace = TraceConverter.load("trace.exo")  # or .perfetto, .json, .otlp

# Convert to different format
trace.save("trace.perfetto", format="perfetto")
trace.save("trace.json", format="chrome")
trace.save("trace.otlp", format="otlp")

# Get pandas DataFrame
df = trace.to_dataframe()

# Get spans
for span in trace.spans:
    print(f"{span.name}: {span.duration_ms}ms")
```

### CLI Conversion Tool

```bash
# Convert between formats
exo convert input.exo output.perfetto
exo convert input.perfetto output.json --format chrome
exo convert input.json output.otlp --format otlp

# Merge multiple traces
exo merge trace1.perfetto trace2.perfetto -o combined.perfetto

# Filter traces
exo filter input.perfetto -o output.perfetto \
    --name "inference*" \
    --min-duration 10ms

# Stream processing
cat /dev/ttyUSB0 | exo convert --stdin --format exo-binary \
    --stdout --format perfetto > trace.perfetto
```

---

## Appendix: Format Details

### Perfetto Track UUIDs

For consistent track identification:

```c
// Reserved UUIDs for standard tracks
#define EXO_TRACK_UUID_MAIN_THREAD  0x0001
#define EXO_TRACK_UUID_CPU          0x0100
#define EXO_TRACK_UUID_NPU          0x0200
#define EXO_TRACK_UUID_GPU          0x0300
#define EXO_TRACK_UUID_DMA          0x0400
#define EXO_TRACK_UUID_DSP          0x0500
#define EXO_TRACK_UUID_COUNTERS     0x1000
```

### OTLP Resource Attributes

Standard resource attributes included:

```
service.name: "inference-engine"
service.version: "1.0.0"
service.instance.id: "device-001"
telemetry.sdk.name: "exo"
telemetry.sdk.version: "0.1.0"
telemetry.sdk.language: "c"
device.id: "soc-abc123"
device.model.name: "AI Accelerator v1"
host.arch: "arm64"
```

### Chrome Trace Metadata

```json
{
  "displayTimeUnit": "ns",
  "systemTraceEvents": "...",
  "metadata": {
    "clock-offset-since-epoch": 1704067200000000000,
    "command_line": "inference_engine --model resnet50",
    "exo-version": "0.1.0"
  }
}
```
