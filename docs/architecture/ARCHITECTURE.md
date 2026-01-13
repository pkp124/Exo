# Exo Architecture Overview

**EXO - Execution Observability for AI Firmware**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Design Philosophy](#design-philosophy)
4. [Open Standards Foundation](#open-standards-foundation)
5. [System Architecture](#system-architecture)
6. [Pluggable Component Model](#pluggable-component-model)
7. [Data Flow](#data-flow)
8. [Deployment Models](#deployment-models)
9. [Technology Choices](#technology-choices)

---

## Executive Summary

Exo is a comprehensive observability framework designed specifically for AI inference firmware running on specialized SoC hardware. It bridges the gap between traditional application performance monitoring (APM) and the unique requirements of embedded AI systems.

**Core Principle: Open Standards First**

Exo is built on open standards, ensuring interoperability and allowing users to replace any component with alternatives from the ecosystem:

- **OpenTelemetry** - Primary API and data model standard
- **W3C Trace Context** - Distributed trace propagation
- **OTLP** - Wire protocol for telemetry export
- **Prometheus** - Metrics exposition format
- **OpenMetrics** - Metrics semantics

The framework provides:
- **Lightweight tracing library** (libexo) - OpenTelemetry-compatible instrumentation for firmware
- **Metrics collection** - Prometheus/OpenMetrics compatible metrics
- **Structured logging** - Correlated log events with trace context
- **Pluggable backends** - Use Jaeger, Zipkin, Prometheus, Grafana, or any OTel-compatible backend
- **AI-specific extensions** - Optional extensions for neural network profiling
- **Performance modeling** - Building analytical models from trace data

---

## Problem Statement

### Current Challenges in AI Firmware Development

1. **Lack of Visibility**: AI inference stacks on embedded SoCs operate as black boxes. Understanding where time is spent, where bottlenecks occur, and how hardware resources are utilized is extremely difficult.

2. **No Standard Observability**: Unlike cloud applications with mature APM solutions, firmware lacks equivalent tooling. Proprietary solutions create vendor lock-in.

3. **Fragmented Tooling**: Different vendors provide incompatible profiling tools, making it hard to correlate data across the stack.

4. **Hardware-Software Co-design Gap**: When exploring new AI accelerator architectures, there's no systematic way to collect standardized execution data.

5. **Benchmark Reproducibility**: AI benchmark results are often reported without context, making comparison difficult.

### Why Open Standards Matter

| Challenge | Open Standards Solution |
|-----------|------------------------|
| Vendor lock-in | OpenTelemetry provides vendor-neutral APIs |
| Tool fragmentation | Standard formats enable tool interoperability |
| Learning curve | Familiar APIs for developers from cloud/backend |
| Future-proofing | Active community ensures continued evolution |
| Ecosystem leverage | Thousands of existing integrations and tools |

---

## Design Philosophy

### 1. Standards-First, Extensions Second

```
┌─────────────────────────────────────────────────────────────┐
│                    User Application                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │        Exo AI Extensions (Optional Layer)           │   │
│  │   • AI operator semantics                           │   │
│  │   • Tensor tracking                                 │   │
│  │   • Hardware counter correlation                    │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │     OpenTelemetry-Compatible Core (Required)        │   │
│  │   • Standard Trace API                              │   │
│  │   • Standard Metrics API                            │   │
│  │   • Standard Logging API                            │   │
│  │   • W3C Trace Context propagation                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

The core APIs follow OpenTelemetry semantics exactly. Users familiar with OTel in other languages will feel at home. AI-specific extensions are optional layers that build on the standard APIs.

### 2. Every Component is Replaceable

```
┌──────────────────────────────────────────────────────────────┐
│                      Instrumentation                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │
│  │ libexo     │  │ OTel C++   │  │ Your Custom Tracer     │ │
│  │ (default)  │  │ SDK        │  │                        │ │
│  └─────┬──────┘  └─────┬──────┘  └───────────┬────────────┘ │
│        │               │                      │              │
│        └───────────────┼──────────────────────┘              │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Standard Interface (OTLP)                  ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                       Collection                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │
│  │ Exo        │  │ OTel       │  │ Your Custom Collector  │ │
│  │ Collector  │  │ Collector  │  │                        │ │
│  └─────┬──────┘  └─────┬──────┘  └───────────┬────────────┘ │
│        │               │                      │              │
│        └───────────────┼──────────────────────┘              │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │          Standard Formats (OTLP, Prometheus)            ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                        Backend                               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ │
│  │ Jaeger  │ │ Zipkin  │ │ Tempo   │ │ Custom  │ │ Files  │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 3. Graceful Degradation

When resources are constrained or components unavailable:

| Scenario | Behavior |
|----------|----------|
| No backend available | Buffer locally, export later |
| Buffer full | Drop oldest data, count drops |
| Tracing disabled | Zero-cost no-ops |
| No OTLP support | Fall back to simpler formats |

### 4. Firmware-Appropriate Constraints

While following standards, we respect firmware constraints:

| Standard Feature | Exo Adaptation |
|-----------------|----------------|
| Dynamic attributes | Fixed-size attribute arrays |
| Unbounded events | Limited events per span |
| Heap allocation | Pre-allocated pools |
| Complex exporters | Simple binary/UART options |

---

## Open Standards Foundation

### OpenTelemetry Compatibility

Exo implements the OpenTelemetry specification adapted for C/embedded environments:

#### Tracing API Mapping

| OpenTelemetry Concept | Exo Implementation |
|----------------------|-------------------|
| `Tracer` | `exo_tracer_t` |
| `Span` | `exo_span_t` |
| `SpanContext` | `exo_span_context_t` |
| `TraceId` (16 bytes) | `exo_trace_id_t` |
| `SpanId` (8 bytes) | `exo_span_id_t` |
| `TraceFlags` | `exo_trace_flags_t` |
| `SpanKind` | `exo_span_kind_t` |
| `Attributes` | `exo_attribute_t[]` |
| `Events` | `exo_event_t[]` |
| `Links` | `exo_link_t[]` |

#### Metrics API Mapping

| OpenTelemetry Concept | Exo Implementation |
|----------------------|-------------------|
| `Meter` | `exo_meter_t` |
| `Counter` | `exo_counter_t` |
| `UpDownCounter` | `exo_updown_counter_t` |
| `Gauge` | `exo_gauge_t` |
| `Histogram` | `exo_histogram_t` |

#### Logging API Mapping

| OpenTelemetry Concept | Exo Implementation |
|----------------------|-------------------|
| `Logger` | `exo_logger_t` |
| `LogRecord` | `exo_log_record_t` |
| `SeverityNumber` | `exo_log_level_t` |

### W3C Trace Context

Full support for W3C Trace Context propagation:

```c
// Trace context format: traceparent header
// 00-<trace-id>-<span-id>-<trace-flags>
// 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

// Parse incoming context
exo_span_context_t ctx;
exo_parse_traceparent("00-4bf92f...-00f067...-01", &ctx);

// Generate outgoing context
char traceparent[64];
exo_format_traceparent(&ctx, traceparent, sizeof(traceparent));
```

### OTLP (OpenTelemetry Protocol)

Native support for OTLP export:

```c
// Configure OTLP exporter
exo_otlp_exporter_config_t config = {
    .endpoint = "http://collector:4317",
    .protocol = EXO_OTLP_GRPC,  // or EXO_OTLP_HTTP_PROTOBUF, EXO_OTLP_HTTP_JSON
    .headers = {{"Authorization", "Bearer token"}},
};
exo_exporter_t* exporter = exo_otlp_exporter_create(&config);
```

For embedded targets without network stack:
```c
// Binary OTLP to file/UART (can be forwarded by host)
exo_exporter_t* exporter = exo_otlp_file_exporter_create("/traces/run001.otlp");
```

### Prometheus / OpenMetrics

Metrics can be exposed in Prometheus format:

```c
// Prometheus exposition format
exo_prometheus_config_t config = {
    .path = "/metrics",
    .port = 9090,
};
exo_prometheus_exporter_create(&config);

// Or write to file for scraping
exo_prometheus_file_exporter_create("/var/metrics/exo.prom");
```

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AI Firmware Application                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Instrumentation API Layer                         │   │
│   │                                                                      │   │
│   │  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────────┐ │   │
│   │  │   Tracing API    │ │   Metrics API    │ │    Logging API       │ │   │
│   │  │ (OTel-compatible)│ │ (OTel-compatible)│ │  (OTel-compatible)   │ │   │
│   │  └────────┬─────────┘ └────────┬─────────┘ └──────────┬───────────┘ │   │
│   │           │                    │                      │             │   │
│   │  ┌────────▼────────────────────▼──────────────────────▼───────────┐ │   │
│   │  │                  AI Extensions (Optional)                       │ │   │
│   │  │  • exo_ai_*  - AI operator tracing                             │ │   │
│   │  │  • exo_tensor_* - Tensor lifecycle tracking                    │ │   │
│   │  │  • exo_hw_* - Hardware counter correlation                     │ │   │
│   │  └────────────────────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│   ┌──────────────────────────────────▼──────────────────────────────────┐   │
│   │                         SDK Layer                                    │   │
│   │  ┌────────────────┐ ┌────────────────┐ ┌────────────────────────┐   │   │
│   │  │ Span Processor │ │ Metric Reader  │ │ Log Record Processor   │   │   │
│   │  └───────┬────────┘ └───────┬────────┘ └───────────┬────────────┘   │   │
│   │          └──────────────────┼──────────────────────┘                │   │
│   │                             ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                    Export Interface                          │    │   │
│   │  │  (Pluggable: OTLP, File, UART, Memory, Custom)              │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   OTLP/gRPC     │         │   OTLP/File     │         │  UART/Serial    │
│   (to OTel      │         │   (offline      │         │  (streaming)    │
│    Collector)   │         │    analysis)    │         │                 │
└────────┬────────┘         └────────┬────────┘         └────────┬────────┘
         │                           │                           │
         ▼                           ▼                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              Collection Layer (Choose Your Backend)                          │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ OpenTelemetry   │  │ Exo Collector   │  │    Direct to Backend        │  │
│  │ Collector       │  │ (lightweight)   │  │    (Jaeger, Tempo, etc.)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Backend Layer (Choose Your Tools)                         │
│                                                                              │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌────────────┐ ┌───────────────────┐  │
│  │ Jaeger  │ │ Zipkin  │ │ Grafana  │ │ Prometheus │ │ Exo Analysis      │  │
│  │         │ │         │ │ Tempo    │ │            │ │ (Python tools)    │  │
│  └─────────┘ └─────────┘ └──────────┘ └────────────┘ └───────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Component Interfaces

Each component has a well-defined interface that can be replaced:

```c
//=============================================================================
// Tracer Provider Interface
//=============================================================================
typedef struct exo_tracer_provider {
    // Get or create a tracer
    exo_tracer_t* (*get_tracer)(const char* name, const char* version);
    
    // Force flush all spans
    exo_status_t (*force_flush)(uint32_t timeout_ms);
    
    // Shutdown
    exo_status_t (*shutdown)(void);
} exo_tracer_provider_t;

// Users can provide their own implementation
exo_status_t exo_set_tracer_provider(exo_tracer_provider_t* provider);

//=============================================================================
// Span Exporter Interface
//=============================================================================
typedef struct exo_span_exporter {
    // Export batch of spans
    exo_status_t (*export)(const exo_span_data_t* spans, size_t count);
    
    // Force flush
    exo_status_t (*flush)(uint32_t timeout_ms);
    
    // Shutdown
    exo_status_t (*shutdown)(void);
} exo_span_exporter_t;

// Register custom exporter
exo_status_t exo_register_span_exporter(exo_span_exporter_t* exporter);

//=============================================================================
// Sampler Interface
//=============================================================================
typedef struct exo_sampler {
    // Decide whether to sample a span
    exo_sampling_result_t (*should_sample)(
        const exo_span_context_t* parent_ctx,
        const char* name,
        exo_span_kind_t kind,
        const exo_attribute_t* attrs,
        size_t attr_count
    );
} exo_sampler_t;

// Register custom sampler
exo_status_t exo_set_sampler(exo_sampler_t* sampler);

//=============================================================================
// ID Generator Interface
//=============================================================================
typedef struct exo_id_generator {
    exo_trace_id_t (*generate_trace_id)(void);
    exo_span_id_t (*generate_span_id)(void);
} exo_id_generator_t;

// Register custom ID generator (e.g., for deterministic testing)
exo_status_t exo_set_id_generator(exo_id_generator_t* generator);

//=============================================================================
// Clock Interface
//=============================================================================
typedef struct exo_clock {
    uint64_t (*now_ns)(void);
} exo_clock_t;

// Register custom clock (e.g., hardware timer)
exo_status_t exo_set_clock(exo_clock_t* clock);
```

---

## Pluggable Component Model

### Replacement Scenarios

| Component | Default | Alternatives |
|-----------|---------|--------------|
| **Tracer Provider** | libexo built-in | OpenTelemetry C++ SDK, custom |
| **Span Exporter** | OTLP | Jaeger, Zipkin, file, UART, custom |
| **Metric Exporter** | OTLP | Prometheus, StatsD, custom |
| **Log Exporter** | OTLP | Loki, file, syslog, custom |
| **Sampler** | Probability | Parent-based, rate-limiting, custom |
| **ID Generator** | Random | Sequential (testing), UUID-based |
| **Clock** | System clock | Hardware timer, TSC, custom |
| **Propagator** | W3C TraceContext | B3, Jaeger, custom |

### Example: Replace Tracing with OpenTelemetry C++ SDK

```cpp
// Instead of using libexo, use OpenTelemetry C++ SDK
#include <opentelemetry/trace/provider.h>
#include <opentelemetry/exporters/otlp/otlp_grpc_exporter.h>

// The rest of your code uses standard OTel APIs
auto provider = opentelemetry::trace::Provider::GetTracerProvider();
auto tracer = provider->GetTracer("my-firmware");
auto span = tracer->StartSpan("inference");
```

### Example: Use libexo with Jaeger Backend

```c
#include <exo/exo.h>
#include <exo/exporters/jaeger.h>

// Configure Jaeger exporter
exo_jaeger_exporter_config_t jaeger_config = {
    .agent_host = "localhost",
    .agent_port = 6831,
};
exo_span_exporter_t* jaeger = exo_jaeger_exporter_create(&jaeger_config);
exo_register_span_exporter(jaeger);

// Use standard tracing APIs
exo_span_t* span = exo_trace_start_span("my-operation");
// ... work ...
exo_trace_end_span(span);
// Spans are exported to Jaeger
```

### Example: Custom Exporter for Proprietary System

```c
// Implement the exporter interface
static exo_status_t my_export(const exo_span_data_t* spans, size_t count) {
    for (size_t i = 0; i < count; i++) {
        my_proprietary_send_trace(&spans[i]);
    }
    return EXO_OK;
}

static exo_status_t my_flush(uint32_t timeout_ms) {
    return my_proprietary_flush();
}

static exo_status_t my_shutdown(void) {
    return my_proprietary_close();
}

exo_span_exporter_t my_exporter = {
    .export = my_export,
    .flush = my_flush,
    .shutdown = my_shutdown,
};

// Register it
exo_register_span_exporter(&my_exporter);
```

---

## Data Flow

### Standard Data Path

```
Application Code
       │
       ▼
┌──────────────────┐
│ Instrumentation  │  exo_trace_start_span(), etc.
│ API              │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Span Processor   │  Batching, filtering
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Exporter         │  Serialize to OTLP/Jaeger/etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Transport        │  gRPC, HTTP, file, UART
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Collector        │  OTel Collector, Jaeger, etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Backend          │  Storage, visualization
└──────────────────┘
```

### Context Propagation (W3C Trace Context)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Request Flow                                 │
│                                                                  │
│  Host Process                    Device Firmware                 │
│  ┌────────────────┐             ┌────────────────────────────┐  │
│  │ Start Span     │             │                            │  │
│  │ trace_id: A    │ ──────────▶ │  Parse traceparent        │  │
│  │ span_id: 1     │  (header)   │  Start child span          │  │
│  └────────────────┘             │  trace_id: A (inherited)   │  │
│                                 │  span_id: 2                │  │
│                                 │  parent_span_id: 1         │  │
│                                 └────────────────────────────┘  │
│                                                                  │
│  traceparent: 00-A-1-01                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Deployment Models

### Model 1: Full OpenTelemetry Stack

```
┌──────────────┐    ┌────────────────┐    ┌─────────────────┐    ┌──────────┐
│  SoC Device  │───▶│  OTel         │───▶│  Grafana Tempo  │───▶│ Grafana  │
│  (libexo)    │OTLP│  Collector    │    │  (traces)       │    │ UI       │
└──────────────┘    └────────────────┘    └─────────────────┘    └──────────┘
                           │
                           ▼
                    ┌─────────────────┐
                    │  Prometheus     │
                    │  (metrics)      │
                    └─────────────────┘
```

### Model 2: Lightweight / Offline

```
┌──────────────┐    ┌────────────────┐    ┌─────────────────┐
│  SoC Device  │───▶│  OTLP Files    │───▶│ Exo Analysis    │
│  (libexo)    │    │  (.otlp.json)  │    │ Tools (Python)  │
└──────────────┘    └────────────────┘    └─────────────────┘
```

### Model 3: Hybrid (Development)

```
┌──────────────┐    ┌────────────────┐
│  SoC Device  │───▶│  Jaeger        │  ◀── View traces live
│  (libexo)    │    │  (all-in-one)  │
└──────────────┘    └───────┬────────┘
                            │
                            ▼
                    ┌────────────────┐
                    │  Exo Analysis  │  ◀── Deep analysis later
                    │  Tools         │
                    └────────────────┘
```

---

## Technology Choices

### Standards Adopted

| Standard | Version | Usage |
|----------|---------|-------|
| OpenTelemetry | 1.x | API semantics, data model |
| W3C Trace Context | Level 2 | Distributed context propagation |
| OTLP | 1.0 | Wire protocol |
| Prometheus | 2.x | Metrics exposition |
| OpenMetrics | 1.0 | Metrics semantics |

### Core Library (libexo)

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | C11 with C++17 wrapper | Maximum portability |
| Dependencies | None required | Works on bare-metal |
| API Style | OpenTelemetry-compatible | Familiar, standard |
| Wire Format | OTLP (protobuf or JSON) | Industry standard |

### Analysis Tools

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python | Data science ecosystem |
| Format Support | OTLP, Jaeger, Zipkin | Standard formats |
| Visualization | Perfetto, Grafana | Standard tools |

---

## Comparison: Exo vs. Direct OpenTelemetry

| Aspect | OpenTelemetry C++ SDK | libexo |
|--------|----------------------|--------|
| Binary size | ~2MB | ~50KB |
| RAM usage | ~100KB+ | ~16KB (configurable) |
| Dependencies | abseil, protobuf, gRPC | None (optional libc) |
| Thread model | Multi-threaded | Single or multi |
| Dynamic allocation | Yes | Optional (pool-based) |
| Bare-metal support | Limited | Yes |
| AI extensions | No | Yes (optional) |
| Standard compliant | Yes | Yes |

**Recommendation**: Use libexo for embedded/firmware targets. Use OpenTelemetry C++ SDK for Linux applications with more resources. Both produce compatible data.

---

## Next Steps

1. **Detailed Specs**: Create specs for metrics, logging, and data formats
2. **Prototype**: Build minimal prototype to validate compatibility
3. **Interoperability Testing**: Verify export to Jaeger, Zipkin, Tempo
4. **Performance Modeling Design**: Detail the analysis approach

---

## References

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/)
- [OpenTelemetry C++ SDK](https://github.com/open-telemetry/opentelemetry-cpp)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)
- [Prometheus Exposition Format](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [OpenMetrics Specification](https://openmetrics.io/)
- [Jaeger](https://www.jaegertracing.io/)
- [Grafana Tempo](https://grafana.com/oss/tempo/)
