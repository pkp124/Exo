# libexo Tracing Library Design

**Detailed Design Specification**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [OpenTelemetry Compatibility](#opentelemetry-compatibility)
3. [API Design](#api-design)
4. [Pluggable Architecture](#pluggable-architecture)
5. [Memory Management](#memory-management)
6. [Thread Safety](#thread-safety)
7. [Context Propagation](#context-propagation)
8. [Sampling](#sampling)
9. [Export Mechanism](#export-mechanism)
10. [AI-Specific Extensions](#ai-specific-extensions)

---

## Overview

libexo is a lightweight, OpenTelemetry-compatible tracing library designed for embedded systems and AI firmware. It provides:

- **Full OpenTelemetry API compatibility** - Same concepts, similar API surface
- **Pluggable components** - Replace any part with standard alternatives
- **Firmware-optimized implementation** - Fixed memory, deterministic latency
- **Optional AI extensions** - First-class support for neural network profiling

### Design Philosophy

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User's Choice                                 │
│                                                                      │
│  "I want to use libexo for tracing"                                 │
│        │                                                             │
│        ├──▶ Use libexo API + libexo SDK                             │
│        │    (Full embedded optimization)                             │
│        │                                                             │
│        ├──▶ Use libexo API + OTLP export to any backend             │
│        │    (libexo instrumentation → Jaeger/Tempo/Zipkin)          │
│        │                                                             │
│        └──▶ Use OpenTelemetry C++ SDK + Exo AI extensions           │
│             (Standard SDK + AI-specific semantic conventions)        │
│                                                                      │
│  All paths produce OpenTelemetry-compatible data!                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## OpenTelemetry Compatibility

### Specification Compliance

libexo implements the OpenTelemetry Tracing specification with adaptations for embedded environments:

| OTel Spec Component | libexo Support | Notes |
|---------------------|----------------|-------|
| TracerProvider | ✅ Full | Pluggable implementation |
| Tracer | ✅ Full | Named, versioned tracers |
| Span | ✅ Full | All span operations |
| SpanContext | ✅ Full | W3C Trace Context compatible |
| SpanKind | ✅ Full | All 5 kinds supported |
| Span Attributes | ✅ Limited | Fixed array size (configurable) |
| Span Events | ✅ Limited | Fixed array size (configurable) |
| Span Links | ✅ Limited | Fixed array size (configurable) |
| Span Status | ✅ Full | Ok, Error, Unset |
| Baggage | ⚠️ Optional | Compile-time option |
| Sampling | ✅ Full | Pluggable sampler interface |
| SpanProcessor | ✅ Full | Pluggable processor interface |
| SpanExporter | ✅ Full | Pluggable exporter interface |
| Resource | ✅ Full | Service name, version, etc. |

### Semantic Conventions

libexo follows OpenTelemetry semantic conventions where applicable:

```c
// Standard semantic convention attributes
exo_span_set_attr_string(span, "service.name", "inference-engine");
exo_span_set_attr_string(span, "service.version", "1.2.3");
exo_span_set_attr_string(span, "deployment.environment", "production");

// Standard HTTP attributes (if applicable)
exo_span_set_attr_string(span, "http.method", "POST");
exo_span_set_attr_int(span, "http.status_code", 200);

// Standard RPC attributes
exo_span_set_attr_string(span, "rpc.system", "grpc");
exo_span_set_attr_string(span, "rpc.method", "Inference");
```

### AI/ML Semantic Conventions (Proposed Extension)

Building on OpenTelemetry's emerging ML semantic conventions:

```c
// Proposed AI semantic conventions (exo extension)
exo_span_set_attr_string(span, "ai.model.name", "resnet50");
exo_span_set_attr_string(span, "ai.model.version", "1.0");
exo_span_set_attr_string(span, "ai.framework", "tflite");
exo_span_set_attr_string(span, "ai.operation.type", "conv2d");
exo_span_set_attr_int(span, "ai.batch_size", 1);
exo_span_set_attr_string(span, "ai.dtype", "float16");
exo_span_set_attr_string(span, "ai.device", "npu");
```

---

## API Design

### Tracer Provider API

```c
//=============================================================================
// TracerProvider - manages Tracer instances
//=============================================================================

// Get the global tracer provider
exo_tracer_provider_t* exo_get_tracer_provider(void);

// Set a custom tracer provider (for replacing implementation)
exo_status_t exo_set_tracer_provider(exo_tracer_provider_t* provider);

// Get a tracer from the provider
exo_tracer_t* exo_get_tracer(const char* name, const char* version);

// Convenience: get tracer with just name
exo_tracer_t* exo_get_tracer_simple(const char* name);
```

### Tracer API

```c
//=============================================================================
// Tracer - creates spans
//=============================================================================

// Start a new span
exo_span_t* exo_tracer_start_span(
    exo_tracer_t* tracer,
    const char* name
);

// Start span with options
exo_span_t* exo_tracer_start_span_with_options(
    exo_tracer_t* tracer,
    const char* name,
    const exo_span_options_t* options
);

// Span options structure
typedef struct {
    exo_span_kind_t kind;                    // Default: INTERNAL
    const exo_span_context_t* parent;        // NULL for root or use context
    const exo_attribute_t* attributes;       // Initial attributes
    size_t attribute_count;
    const exo_link_t* links;                 // Span links
    size_t link_count;
    exo_timestamp_t start_time;              // 0 for now
} exo_span_options_t;
```

### Span API

```c
//=============================================================================
// Span - represents a unit of work
//=============================================================================

// End the span (required to complete the span)
void exo_span_end(exo_span_t* span);

// End with specific timestamp
void exo_span_end_with_timestamp(exo_span_t* span, exo_timestamp_t timestamp);

// Check if span is recording (not sampled out)
bool exo_span_is_recording(exo_span_t* span);

// Get span context (for propagation)
exo_span_context_t exo_span_get_context(exo_span_t* span);

//-----------------------------------------------------------------------------
// Attributes (key-value pairs)
//-----------------------------------------------------------------------------

exo_status_t exo_span_set_attribute_string(exo_span_t* span, const char* key, const char* value);
exo_status_t exo_span_set_attribute_int(exo_span_t* span, const char* key, int64_t value);
exo_status_t exo_span_set_attribute_double(exo_span_t* span, const char* key, double value);
exo_status_t exo_span_set_attribute_bool(exo_span_t* span, const char* key, bool value);

// Set multiple attributes at once (more efficient)
exo_status_t exo_span_set_attributes(
    exo_span_t* span,
    const exo_attribute_t* attributes,
    size_t count
);

//-----------------------------------------------------------------------------
// Events (timestamped annotations)
//-----------------------------------------------------------------------------

exo_status_t exo_span_add_event(exo_span_t* span, const char* name);

exo_status_t exo_span_add_event_with_attributes(
    exo_span_t* span,
    const char* name,
    const exo_attribute_t* attributes,
    size_t count
);

exo_status_t exo_span_add_event_with_timestamp(
    exo_span_t* span,
    const char* name,
    exo_timestamp_t timestamp,
    const exo_attribute_t* attributes,
    size_t count
);

//-----------------------------------------------------------------------------
// Status
//-----------------------------------------------------------------------------

typedef enum {
    EXO_SPAN_STATUS_UNSET = 0,  // Default
    EXO_SPAN_STATUS_OK = 1,
    EXO_SPAN_STATUS_ERROR = 2,
} exo_span_status_code_t;

exo_status_t exo_span_set_status(
    exo_span_t* span,
    exo_span_status_code_t code,
    const char* description  // Only used for ERROR
);

// Convenience for recording exceptions
exo_status_t exo_span_record_exception(
    exo_span_t* span,
    const char* type,
    const char* message
);

//-----------------------------------------------------------------------------
// Update span name (rare, but allowed by spec)
//-----------------------------------------------------------------------------

exo_status_t exo_span_update_name(exo_span_t* span, const char* name);
```

### SpanContext API

```c
//=============================================================================
// SpanContext - immutable context for propagation
//=============================================================================

typedef struct {
    exo_trace_id_t trace_id;      // 16 bytes
    exo_span_id_t span_id;        // 8 bytes
    exo_trace_flags_t flags;      // 1 byte (sampled, etc.)
    bool is_remote;               // Whether context came from remote parent
} exo_span_context_t;

// Check if context is valid
bool exo_span_context_is_valid(const exo_span_context_t* ctx);

// Check if sampled
bool exo_span_context_is_sampled(const exo_span_context_t* ctx);

// Create context from IDs (for parsing propagation headers)
exo_span_context_t exo_span_context_create(
    exo_trace_id_t trace_id,
    exo_span_id_t span_id,
    exo_trace_flags_t flags,
    bool is_remote
);
```

### Convenience API (Simplified Usage)

```c
//=============================================================================
// Convenience functions using default tracer
//=============================================================================

// These use the default tracer and current context automatically

exo_span_t* exo_start_span(const char* name);
exo_span_t* exo_start_span_with_kind(const char* name, exo_span_kind_t kind);

// Start child span of currently active span
exo_span_t* exo_start_child_span(const char* name);

// Get/set active span in current context
exo_span_t* exo_get_current_span(void);
void exo_set_current_span(exo_span_t* span);
```

### C++ Wrapper API

```cpp
#include <exo/exo.hpp>

namespace exo {

// RAII Span wrapper
class Span {
public:
    Span(const std::string& name);
    Span(const std::string& name, SpanKind kind);
    ~Span();  // Automatically ends span
    
    Span& SetAttribute(const std::string& key, const std::string& value);
    Span& SetAttribute(const std::string& key, int64_t value);
    Span& SetAttribute(const std::string& key, double value);
    Span& SetAttribute(const std::string& key, bool value);
    
    Span& AddEvent(const std::string& name);
    Span& SetStatus(StatusCode code, const std::string& description = "");
    
    SpanContext GetContext() const;
    bool IsRecording() const;
    
private:
    exo_span_t* span_;
};

// Scoped span with lambda
template<typename F>
auto WithSpan(const std::string& name, F&& func) {
    Span span(name);
    return func();
}

// Builder pattern
class SpanBuilder {
public:
    SpanBuilder(const std::string& name);
    
    SpanBuilder& SetKind(SpanKind kind);
    SpanBuilder& SetParent(const SpanContext& parent);
    SpanBuilder& AddLink(const SpanContext& linked);
    SpanBuilder& SetAttribute(const std::string& key, /* value */);
    
    Span Start();
};

} // namespace exo

// Usage example
void process_inference() {
    exo::Span span("inference");
    span.SetAttribute("model", "resnet50")
        .SetAttribute("batch_size", 32);
    
    {
        exo::Span child("preprocessing");
        preprocess();
    }  // child ends
    
    {
        exo::Span child("execution");
        execute();
    }  // child ends
    
}  // span ends
```

---

## Pluggable Architecture

### Component Interfaces

Every major component has a well-defined interface that can be replaced:

```c
//=============================================================================
// TracerProvider Interface
//=============================================================================
typedef struct exo_tracer_provider_vtable {
    exo_tracer_t* (*get_tracer)(void* impl, const char* name, const char* version);
    exo_status_t (*force_flush)(void* impl, uint32_t timeout_ms);
    exo_status_t (*shutdown)(void* impl);
} exo_tracer_provider_vtable_t;

typedef struct exo_tracer_provider {
    const exo_tracer_provider_vtable_t* vtable;
    void* impl;
} exo_tracer_provider_t;

//=============================================================================
// SpanProcessor Interface (processes spans before export)
//=============================================================================
typedef struct exo_span_processor_vtable {
    void (*on_start)(void* impl, exo_span_t* span, const exo_span_context_t* parent);
    void (*on_end)(void* impl, exo_span_t* span);
    exo_status_t (*force_flush)(void* impl, uint32_t timeout_ms);
    exo_status_t (*shutdown)(void* impl);
} exo_span_processor_vtable_t;

typedef struct exo_span_processor {
    const exo_span_processor_vtable_t* vtable;
    void* impl;
} exo_span_processor_t;

//=============================================================================
// SpanExporter Interface (exports spans to backend)
//=============================================================================
typedef struct exo_span_exporter_vtable {
    exo_status_t (*export_spans)(void* impl, const exo_span_data_t* spans, size_t count);
    exo_status_t (*force_flush)(void* impl, uint32_t timeout_ms);
    exo_status_t (*shutdown)(void* impl);
} exo_span_exporter_vtable_t;

typedef struct exo_span_exporter {
    const exo_span_exporter_vtable_t* vtable;
    void* impl;
} exo_span_exporter_t;

//=============================================================================
// Sampler Interface (decides whether to record spans)
//=============================================================================
typedef struct exo_sampling_result {
    exo_sampling_decision_t decision;  // DROP, RECORD_ONLY, RECORD_AND_SAMPLE
    exo_attribute_t* attributes;       // Additional attributes to add
    size_t attribute_count;
} exo_sampling_result_t;

typedef struct exo_sampler_vtable {
    exo_sampling_result_t (*should_sample)(
        void* impl,
        const exo_span_context_t* parent_context,
        exo_trace_id_t trace_id,
        const char* name,
        exo_span_kind_t kind,
        const exo_attribute_t* attributes,
        size_t attribute_count,
        const exo_link_t* links,
        size_t link_count
    );
    const char* (*get_description)(void* impl);
} exo_sampler_vtable_t;

typedef struct exo_sampler {
    const exo_sampler_vtable_t* vtable;
    void* impl;
} exo_sampler_t;

//=============================================================================
// IDGenerator Interface (generates trace/span IDs)
//=============================================================================
typedef struct exo_id_generator_vtable {
    exo_trace_id_t (*generate_trace_id)(void* impl);
    exo_span_id_t (*generate_span_id)(void* impl);
} exo_id_generator_vtable_t;

typedef struct exo_id_generator {
    const exo_id_generator_vtable_t* vtable;
    void* impl;
} exo_id_generator_t;
```

### Built-in Implementations

```c
//=============================================================================
// Built-in Samplers
//=============================================================================

// Always sample
exo_sampler_t* exo_sampler_always_on_create(void);

// Never sample
exo_sampler_t* exo_sampler_always_off_create(void);

// Probability-based sampling
exo_sampler_t* exo_sampler_trace_id_ratio_create(double ratio);

// Follow parent's decision
exo_sampler_t* exo_sampler_parent_based_create(exo_sampler_t* root_sampler);

//=============================================================================
// Built-in SpanProcessors
//=============================================================================

// Simple processor (exports immediately)
exo_span_processor_t* exo_simple_span_processor_create(exo_span_exporter_t* exporter);

// Batching processor (exports in batches)
typedef struct {
    size_t max_queue_size;        // Default: 2048
    size_t scheduled_delay_ms;    // Default: 5000
    size_t max_export_batch_size; // Default: 512
    size_t export_timeout_ms;     // Default: 30000
} exo_batch_span_processor_options_t;

exo_span_processor_t* exo_batch_span_processor_create(
    exo_span_exporter_t* exporter,
    const exo_batch_span_processor_options_t* options
);

//=============================================================================
// Built-in SpanExporters
//=============================================================================

// OTLP gRPC exporter (if network available)
typedef struct {
    const char* endpoint;  // e.g., "http://localhost:4317"
    // ... headers, compression, etc.
} exo_otlp_grpc_exporter_options_t;

exo_span_exporter_t* exo_otlp_grpc_exporter_create(
    const exo_otlp_grpc_exporter_options_t* options
);

// OTLP HTTP/JSON exporter
typedef struct {
    const char* endpoint;  // e.g., "http://localhost:4318/v1/traces"
} exo_otlp_http_exporter_options_t;

exo_span_exporter_t* exo_otlp_http_exporter_create(
    const exo_otlp_http_exporter_options_t* options
);

// OTLP file exporter (for offline analysis)
exo_span_exporter_t* exo_otlp_file_exporter_create(const char* path);

// Console exporter (for debugging)
exo_span_exporter_t* exo_console_exporter_create(FILE* output);

// Null exporter (for benchmarking)
exo_span_exporter_t* exo_null_exporter_create(void);

// UART/Serial exporter (for embedded)
exo_span_exporter_t* exo_uart_exporter_create(int fd);
```

### Replacing Components

```c
// Example: Use custom tracer provider
exo_tracer_provider_t* my_provider = create_my_custom_provider();
exo_set_tracer_provider(my_provider);

// Example: Use Jaeger exporter instead of OTLP
exo_span_exporter_t* jaeger = exo_jaeger_exporter_create(&jaeger_options);
exo_span_processor_t* processor = exo_batch_span_processor_create(jaeger, NULL);
exo_tracer_provider_add_processor(provider, processor);

// Example: Custom sampler
exo_sampler_t* my_sampler = create_my_ai_aware_sampler();
exo_tracer_provider_set_sampler(provider, my_sampler);
```

---

## Memory Management

### Pool-Based Allocation

```c
// Configuration at init time
typedef struct {
    size_t max_spans;              // Pool size for spans (default: 256)
    size_t max_attributes_per_span; // Per-span limit (default: 16)
    size_t max_events_per_span;     // Per-span limit (default: 8)
    size_t max_links_per_span;      // Per-span limit (default: 4)
    
    // Optional: provide your own buffer
    void* buffer;
    size_t buffer_size;
    
    // Optional: custom allocator for non-pool allocations
    void* (*alloc)(size_t size, void* user_data);
    void (*free)(void* ptr, void* user_data);
    void* alloc_user_data;
} exo_resource_limits_t;

// Default configuration uses static pools
exo_init(NULL);  // Uses defaults

// Custom configuration
exo_config_t config = {
    .resource_limits = {
        .max_spans = 128,
        .max_attributes_per_span = 8,
    },
};
exo_init(&config);
```

### Memory Layout

```
┌──────────────────────────────────────────────────────────────┐
│                    Static Memory Region                       │
├────────────────────────────────────────────────────────────── │
│  Global State        │  ~500 bytes                           │
│  - Provider pointer  │                                       │
│  - Samplers          │                                       │
│  - Processors        │                                       │
├──────────────────────┼───────────────────────────────────────│
│  Span Pool           │  N × sizeof(span) ≈ N × 512 bytes    │
│  (pre-allocated)     │  Default: 256 × 512 = 128KB          │
├──────────────────────┼───────────────────────────────────────│
│  String Interning    │  Optional, for attribute keys        │
│  Table               │  ~4KB                                 │
├──────────────────────┼───────────────────────────────────────│
│  Export Buffer       │  Ring buffer for completed spans     │
│                      │  ~32KB                                │
└──────────────────────┴───────────────────────────────────────┘
```

---

## Context Propagation

### W3C Trace Context (Default)

```c
// Parse incoming traceparent header
// Format: VERSION-TRACEID-SPANID-FLAGS
// Example: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

exo_span_context_t ctx;
exo_status_t status = exo_parse_traceparent(
    "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
    &ctx
);

// Use as parent for new span
exo_span_options_t opts = {
    .parent = &ctx,
};
exo_span_t* span = exo_tracer_start_span_with_options(tracer, "child", &opts);

// Generate outgoing traceparent header
char traceparent[64];
exo_format_traceparent(&span_context, traceparent, sizeof(traceparent));
```

### Propagator Interface (Pluggable)

```c
// Propagator interface for different formats
typedef struct exo_propagator_vtable {
    // Inject context into carrier (e.g., HTTP headers)
    void (*inject)(
        void* impl,
        const exo_span_context_t* context,
        exo_carrier_t* carrier
    );
    
    // Extract context from carrier
    exo_span_context_t (*extract)(
        void* impl,
        const exo_carrier_t* carrier
    );
    
    // Get field names used by this propagator
    const char** (*fields)(void* impl, size_t* count);
} exo_propagator_vtable_t;

// Built-in propagators
exo_propagator_t* exo_propagator_w3c_create(void);      // W3C Trace Context
exo_propagator_t* exo_propagator_b3_create(void);       // Zipkin B3
exo_propagator_t* exo_propagator_jaeger_create(void);   // Jaeger native

// Composite propagator (tries multiple)
exo_propagator_t* exo_propagator_composite_create(
    exo_propagator_t** propagators,
    size_t count
);
```

---

## Sampling

### Sampling Decision Flow

```
                    ┌─────────────────────┐
                    │  should_sample()    │
                    │  called at span     │
                    │  start              │
                    └──────────┬──────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │  Sampling Decision             │
              ├────────────────────────────────┤
              │  DROP           → Span not     │
              │                   created      │
              │                                │
              │  RECORD_ONLY    → Span created │
              │                   but not      │
              │                   exported     │
              │                                │
              │  RECORD_AND_    → Span created │
              │  SAMPLE           and exported │
              └────────────────────────────────┘
```

### Built-in Samplers

```c
// Always sample everything
exo_sampler_t* always_on = exo_sampler_always_on_create();

// Sample based on trace ID (consistent across services)
// ratio = 0.1 means ~10% of traces sampled
exo_sampler_t* ratio = exo_sampler_trace_id_ratio_create(0.1);

// Parent-based: follow parent's decision, use root_sampler for root spans
exo_sampler_t* parent_based = exo_sampler_parent_based_create(ratio);
```

### Custom Sampler Example

```c
// AI-aware sampler: always sample slow inferences
static exo_sampling_result_t ai_sampler_should_sample(
    void* impl,
    const exo_span_context_t* parent_context,
    exo_trace_id_t trace_id,
    const char* name,
    exo_span_kind_t kind,
    const exo_attribute_t* attributes,
    size_t attribute_count,
    const exo_link_t* links,
    size_t link_count
) {
    // Check if this is an inference span
    if (strstr(name, "inference") != NULL) {
        // Always sample inference spans
        return (exo_sampling_result_t){
            .decision = EXO_SAMPLING_RECORD_AND_SAMPLE,
        };
    }
    
    // Default: probability sampling
    return default_probability_sample(trace_id);
}

static exo_sampler_vtable_t ai_sampler_vtable = {
    .should_sample = ai_sampler_should_sample,
    .get_description = ai_sampler_description,
};

exo_sampler_t ai_sampler = {
    .vtable = &ai_sampler_vtable,
    .impl = NULL,
};
```

---

## Export Mechanism

### Export Data Format

```c
// SpanData structure for export (read-only view of completed span)
typedef struct exo_span_data {
    // Identity
    exo_span_context_t span_context;
    exo_span_id_t parent_span_id;
    
    // Metadata
    const char* name;
    exo_span_kind_t kind;
    
    // Timing
    exo_timestamp_t start_time_ns;
    exo_timestamp_t end_time_ns;
    
    // Status
    exo_span_status_code_t status_code;
    const char* status_description;
    
    // Attributes
    const exo_attribute_t* attributes;
    size_t attribute_count;
    
    // Events
    const exo_event_t* events;
    size_t event_count;
    
    // Links
    const exo_link_t* links;
    size_t link_count;
    
    // Resource (service info)
    const exo_resource_t* resource;
    
    // Instrumentation scope (tracer info)
    const exo_instrumentation_scope_t* scope;
} exo_span_data_t;
```

### OTLP Export

The default export format is OTLP (OpenTelemetry Protocol):

```c
// OTLP export converts exo_span_data_t to OTLP protobuf/JSON
// This ensures compatibility with:
// - OpenTelemetry Collector
// - Jaeger (with OTLP receiver)
// - Grafana Tempo
// - Any OTLP-compatible backend
```

---

## AI-Specific Extensions

### Overview

AI extensions are optional layers that build on standard OpenTelemetry APIs:

```c
#include <exo/exo.h>
#include <exo/ai.h>  // Optional AI extensions

// AI extensions use standard spans with semantic conventions
exo_span_t* span = exo_ai_start_op_span(EXO_AI_OP_CONV2D);
// This creates a normal span with ai.* attributes pre-set
```

### AI Operator Tracing

```c
// Pre-defined AI operations
typedef enum {
    EXO_AI_OP_INFERENCE,      // Top-level inference
    EXO_AI_OP_CONV2D,
    EXO_AI_OP_MATMUL,
    EXO_AI_OP_ATTENTION,
    EXO_AI_OP_SOFTMAX,
    EXO_AI_OP_RELU,
    EXO_AI_OP_LAYERNORM,
    EXO_AI_OP_POOLING,
    EXO_AI_OP_MEMORY_COPY,
    EXO_AI_OP_DMA_TRANSFER,
    EXO_AI_OP_CUSTOM,
} exo_ai_op_type_t;

// Start an AI operator span (convenience)
exo_span_t* exo_ai_start_op_span(exo_ai_op_type_t op_type);

// Set tensor shapes
exo_status_t exo_ai_set_input_shape(exo_span_t* span, const int64_t* shape, size_t ndims);
exo_status_t exo_ai_set_output_shape(exo_span_t* span, const int64_t* shape, size_t ndims);

// Set compute characteristics
exo_status_t exo_ai_set_flops(exo_span_t* span, uint64_t flops);
exo_status_t exo_ai_set_memory_bytes(exo_span_t* span, uint64_t bytes_read, uint64_t bytes_written);
```

### Semantic Conventions for AI

Proposed semantic conventions (prefix: `ai.`):

| Attribute | Type | Description |
|-----------|------|-------------|
| `ai.model.name` | string | Model name (e.g., "resnet50") |
| `ai.model.version` | string | Model version |
| `ai.framework` | string | Framework (e.g., "tflite", "onnx") |
| `ai.operation.type` | string | Operation type (e.g., "conv2d") |
| `ai.batch_size` | int | Batch size |
| `ai.dtype` | string | Data type (e.g., "float16", "int8") |
| `ai.device` | string | Execution device (e.g., "npu", "cpu") |
| `ai.input.shape` | int[] | Input tensor shape |
| `ai.output.shape` | int[] | Output tensor shape |
| `ai.flops` | int | Floating point operations |
| `ai.memory.read_bytes` | int | Bytes read |
| `ai.memory.write_bytes` | int | Bytes written |
| `ai.hardware.utilization` | double | Hardware utilization (0-1) |
| `ai.hardware.power_mw` | int | Power consumption in milliwatts |

---

## Configuration Summary

```c
typedef struct exo_config {
    // Resource (service identification)
    const char* service_name;         // Required
    const char* service_version;      // Optional
    const char* service_instance_id;  // Optional
    const exo_attribute_t* resource_attributes;
    size_t resource_attribute_count;
    
    // Tracing
    bool tracing_enabled;             // Default: true
    exo_sampler_t* sampler;           // Default: always_on
    exo_span_processor_t* processor;  // Default: simple processor
    exo_span_exporter_t* exporter;    // Default: console (debug) or OTLP
    
    // Propagation
    exo_propagator_t* propagator;     // Default: W3C Trace Context
    
    // Resource limits (for embedded)
    exo_resource_limits_t resource_limits;
    
    // Clock source
    exo_clock_t* clock;               // Default: system clock
    
    // ID generator
    exo_id_generator_t* id_generator; // Default: random
    
} exo_config_t;

// Initialize with configuration
exo_status_t exo_init(const exo_config_t* config);

// Shutdown and cleanup
exo_status_t exo_shutdown(void);
```

---

## Appendix: Interoperability Matrix

| Backend | Protocol | Status |
|---------|----------|--------|
| OpenTelemetry Collector | OTLP gRPC/HTTP | ✅ Native |
| Jaeger | OTLP or Jaeger native | ✅ Supported |
| Zipkin | Zipkin JSON | ✅ Supported |
| Grafana Tempo | OTLP | ✅ Native |
| AWS X-Ray | OTLP (via Collector) | ✅ Via OTel Collector |
| Datadog | OTLP (via Collector) | ✅ Via OTel Collector |
| Honeycomb | OTLP | ✅ Native |
| Lightstep | OTLP | ✅ Native |
| Custom | Exporter interface | ✅ Pluggable |
