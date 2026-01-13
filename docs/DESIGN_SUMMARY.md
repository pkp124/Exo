# Exo Design Summary

**Quick Reference for Exo Design Decisions**

---

## Core Design Principles

### 1. Open Standards First

| Standard | Usage in Exo |
|----------|--------------|
| **OpenTelemetry** | API semantics, data model, OTLP protocol |
| **W3C Trace Context** | Distributed trace propagation |
| **Perfetto** | Timeline visualization, system tracing |
| **Prometheus/OpenMetrics** | Metrics exposition |

### 2. Pluggable Everything

```
User Code
    │
    ▼
┌─────────────────────────────────┐
│ Instrumentation API (Standard)  │  ← OpenTelemetry-compatible
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ Provider (Replaceable)          │  ← libexo or OTel C++ SDK
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ Exporter (Replaceable)          │  ← Perfetto, OTLP, custom
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ Backend (User's Choice)         │  ← Jaeger, Tempo, files
└─────────────────────────────────┘
```

### 3. Firmware-Appropriate

| Constraint | Solution |
|------------|----------|
| Limited memory | Pre-allocated pools, configurable sizes |
| No malloc | Static allocation, optional heap |
| Deterministic latency | Lock-free structures, bounded operations |
| Low overhead | Compile-time disable, sampling |

---

## Component Overview

### libexo (Core Library)

**Purpose:** Lightweight instrumentation for firmware

**Key APIs:**
- `exo_init()` / `exo_shutdown()` - Lifecycle
- `exo_start_span()` / `exo_span_end()` - Tracing
- `exo_counter_add()` / `exo_histogram_record()` - Metrics
- `exo_log()` - Logging

**Memory:** ~16KB RAM (configurable)
**Overhead:** <500ns per span start/end

### Exporters

| Exporter | Format | Use Case |
|----------|--------|----------|
| **Perfetto** | Protobuf | Timeline visualization |
| **OTLP** | Protobuf/JSON | Distributed tracing backends |
| **Chrome Trace** | JSON | Simple visualization |
| **Binary** | Custom | Minimal size embedded |

### Analysis Tools (Python)

**Purpose:** Post-collection analysis and modeling

**Capabilities:**
- Trace parsing (all formats)
- Roofline model generation
- Bottleneck detection
- Architecture exploration
- Report generation

---

## Data Format Selection

| Scenario | Recommended Format |
|----------|-------------------|
| Development/debugging | **Perfetto** |
| Production monitoring | **OTLP** → backend |
| Offline analysis | **Perfetto** or **OTLP file** |
| Minimal embedded | **Exo Binary** |
| Quick visualization | **Chrome Trace** |

---

## AI-Specific Extensions

### Semantic Conventions

```c
// Model identification
exo_span_set_attribute_string(span, "ai.model.name", "resnet50");
exo_span_set_attribute_string(span, "ai.framework", "tflite");

// Operation details
exo_span_set_attribute_string(span, "ai.operation.type", "conv2d");
exo_span_set_attribute_int(span, "ai.flops", 1000000);
exo_span_set_attribute_string(span, "ai.device", "npu");

// Tensor shapes
exo_span_set_attribute_string(span, "ai.input.shape", "[1,224,224,3]");
exo_span_set_attribute_string(span, "ai.output.shape", "[1,1000]");
```

### Hardware Tracks (Perfetto)

```
▼ inference_engine
  ├─ CPU Thread
  ├─ NPU
  ├─ DMA
  └─ Counters
     ├─ NPU Utilization
     └─ Memory Bandwidth
```

---

## Performance Modeling

### From Traces to Insights

```
Traces → Feature Extraction → Models → Predictions
                │
                ├─ Roofline Model
                ├─ Critical Path Analysis
                ├─ Bottleneck Detection
                └─ What-if Analysis
```

### Key Questions Answered

1. **Is it compute or memory bound?** → Roofline analysis
2. **What's the bottleneck?** → Critical path + utilization
3. **What if hardware changed?** → Parameterized models
4. **Where to optimize?** → Ranked recommendations

---

## Quick Start (Planned API)

```c
#include <exo/exo.h>

// Initialize
exo_init(&(exo_config_t){
    .service_name = "my_engine",
});

// Trace
exo_span_t* span = exo_start_span("inference");
run_inference();
exo_span_end(span);

// Shutdown
exo_shutdown();
```

---

---

## End Goal: Architecture Exploration Platform

The ultimate vision is to build an AI-powered platform for architecture exploration:

```
Traces from CI Runs  →  ML Training  →  Predictions for New Designs
                                ↓
                    Architecture Recommendations
                    Partitioning Optimization  
                    Regression Detection
```

### Key Use Cases

1. **Performance Prediction** - "How will ResNet-50 perform on this new NPU design?"
2. **Architecture Exploration** - "Should we double compute or memory bandwidth?"
3. **Partitioning** - "How should we distribute this model across CPU/NPU/DMA?"
4. **Regression Detection** - "Did this firmware change cause a regression?"

---

## Document Index

| Category | Documents |
|----------|-----------|
| **Architecture** | [ARCHITECTURE.md](architecture/ARCHITECTURE.md) |
| **Core Design** | [TRACING_LIBRARY.md](design/TRACING_LIBRARY.md), [METRICS_LOGGING.md](design/METRICS_LOGGING.md), [INSTRUMENTATION_SYSTEM.md](design/INSTRUMENTATION_SYSTEM.md) |
| **Platform** | [STORAGE_INFRASTRUCTURE.md](design/STORAGE_INFRASTRUCTURE.md), [ARCHITECTURE_EXPLORATION_PLATFORM.md](design/ARCHITECTURE_EXPLORATION_PLATFORM.md), [ML_TRAINING_FRAMEWORK.md](design/ML_TRAINING_FRAMEWORK.md) |
| **Analysis** | [PERFORMANCE_MODELING.md](design/PERFORMANCE_MODELING.md) |
| **Specs** | [DATA_FORMATS.md](specs/DATA_FORMATS.md) |
| **Guides** | [AI_INTEGRATION.md](guides/AI_INTEGRATION.md) |
