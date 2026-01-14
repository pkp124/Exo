# AI Inference Stack Integration Guide

**Integrating Exo with AI Frameworks and Inference Engines**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Integration Patterns](#integration-patterns)
3. [Framework Integration Examples](#framework-integration-examples)
4. [Semantic Conventions for AI](#semantic-conventions-for-ai)
5. [Best Practices](#best-practices)
6. [Common Instrumentation Patterns](#common-instrumentation-patterns)

---

## Overview

This guide describes how to integrate Exo observability into AI inference stacks, from high-level frameworks down to hardware drivers.

### Integration Points

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AI Application Layer                              │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Model Loading │ Input Preprocessing │ Output Postprocessing       │ │
│  │                        ↓                                           │ │
│  │              [Exo: Application-level tracing]                      │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                        Framework Layer                                   │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  TFLite │ ONNX Runtime │ PyTorch │ Custom Runtime                  │ │
│  │                        ↓                                           │ │
│  │              [Exo: Framework-level tracing]                        │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                        Operator/Kernel Layer                             │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Conv2D │ MatMul │ Attention │ Softmax │ ...                       │ │
│  │                        ↓                                           │ │
│  │              [Exo: Operator-level tracing]                         │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                        Hardware Abstraction Layer                        │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  NPU Driver │ DMA Controller │ Memory Manager                      │ │
│  │                        ↓                                           │ │
│  │              [Exo: Hardware-level tracing]                         │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                        Hardware                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  CPU │ NPU │ GPU │ DMA │ Memory                                    │ │
│  │                        ↓                                           │ │
│  │              [Exo: Hardware counter sampling]                       │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Integration Patterns

### Pattern 1: Wrapper Integration

Wrap existing functions without modifying them:

```c
// Original function
void conv2d_execute(conv2d_params_t* params, tensor_t* input, tensor_t* output);

// Wrapped version
void conv2d_execute_traced(conv2d_params_t* params, tensor_t* input, tensor_t* output) {
    exo_span_t* span = exo_start_span("conv2d");
    
    // Add operator attributes
    exo_span_set_attribute_string(span, "ai.operation.type", "conv2d");
    exo_span_set_attribute_int(span, "ai.kernel.height", params->kernel_h);
    exo_span_set_attribute_int(span, "ai.kernel.width", params->kernel_w);
    exo_span_set_attribute_int(span, "ai.stride.height", params->stride_h);
    exo_span_set_attribute_int(span, "ai.stride.width", params->stride_w);
    
    // Add tensor info
    char shape_str[64];
    format_shape(input->shape, input->ndims, shape_str);
    exo_span_set_attribute_string(span, "ai.input.shape", shape_str);
    
    // Execute original
    conv2d_execute(params, input, output);
    
    exo_span_end(span);
}

// Macro for easy wrapping
#define TRACE_OP(op_name, func_call) do { \
    exo_span_t* _span = exo_start_span(op_name); \
    func_call; \
    exo_span_end(_span); \
} while(0)

// Usage
TRACE_OP("conv2d", conv2d_execute(params, input, output));
```

### Pattern 2: Callback Integration

Use callbacks/hooks provided by frameworks:

```c
// TFLite-style callback integration
typedef struct {
    void (*on_op_start)(const char* op_name, void* user_data);
    void (*on_op_end)(const char* op_name, void* user_data);
    void* user_data;
} inference_callbacks_t;

// Exo callback implementation
typedef struct {
    exo_span_t* current_span;
} exo_callback_context_t;

void exo_on_op_start(const char* op_name, void* user_data) {
    exo_callback_context_t* ctx = (exo_callback_context_t*)user_data;
    ctx->current_span = exo_start_span(op_name);
}

void exo_on_op_end(const char* op_name, void* user_data) {
    exo_callback_context_t* ctx = (exo_callback_context_t*)user_data;
    exo_span_end(ctx->current_span);
    ctx->current_span = NULL;
}

// Register callbacks
exo_callback_context_t exo_ctx = {0};
inference_callbacks_t callbacks = {
    .on_op_start = exo_on_op_start,
    .on_op_end = exo_on_op_end,
    .user_data = &exo_ctx,
};
runtime_register_callbacks(&callbacks);
```

### Pattern 3: Compile-Time Integration

Use macros that compile out when tracing is disabled:

```c
#if EXO_TRACING_ENABLED

#define EXO_AI_OP_BEGIN(op_type, name) \
    exo_span_t* _exo_span = exo_ai_start_op_span(op_type); \
    exo_span_set_attribute_string(_exo_span, "ai.op.name", name)

#define EXO_AI_OP_END() \
    exo_span_end(_exo_span)

#define EXO_AI_SET_SHAPES(in_shape, out_shape) \
    exo_ai_set_input_shape(_exo_span, in_shape); \
    exo_ai_set_output_shape(_exo_span, out_shape)

#else

#define EXO_AI_OP_BEGIN(op_type, name) ((void)0)
#define EXO_AI_OP_END() ((void)0)
#define EXO_AI_SET_SHAPES(in_shape, out_shape) ((void)0)

#endif

// Usage in operator implementation
void conv2d_impl(conv2d_params_t* p, tensor_t* in, tensor_t* out) {
    EXO_AI_OP_BEGIN(EXO_AI_OP_CONV2D, "conv2d");
    EXO_AI_SET_SHAPES(in->shape, out->shape);
    
    // ... actual implementation ...
    
    EXO_AI_OP_END();
}
```

### Pattern 4: Aspect-Oriented Integration

For C++, use RAII for automatic tracing:

```cpp
// RAII span wrapper
class AIOp {
public:
    AIOp(const char* name, exo_ai_op_type_t type) {
        span_ = exo_ai_start_op_span(type);
        exo_span_set_attribute_string(span_, "ai.op.name", name);
    }
    
    ~AIOp() {
        exo_span_end(span_);
    }
    
    void SetInputShape(const std::vector<int64_t>& shape) {
        exo_ai_set_input_shape(span_, shape.data(), shape.size());
    }
    
    void SetOutputShape(const std::vector<int64_t>& shape) {
        exo_ai_set_output_shape(span_, shape.data(), shape.size());
    }
    
    void SetFlops(uint64_t flops) {
        exo_span_set_attribute_int(span_, "ai.flops", flops);
    }
    
private:
    exo_span_t* span_;
};

// Usage
void conv2d_impl(const Conv2dParams& params, const Tensor& input, Tensor& output) {
    AIOp op("conv2d", EXO_AI_OP_CONV2D);
    op.SetInputShape(input.shape());
    op.SetOutputShape(output.shape());
    op.SetFlops(calculate_conv2d_flops(params, input.shape()));
    
    // ... implementation ...
}  // Automatically ends span
```

---

## Framework Integration Examples

### TensorFlow Lite Integration

```c
#include <tensorflow/lite/c/c_api.h>
#include <exo/exo.h>
#include <exo/ai.h>

// Custom TFLite profiler using Exo
typedef struct {
    exo_span_t* spans[256];
    int span_count;
} ExoTfLiteProfiler;

void exo_tflite_begin_event(void* user_data, const char* tag, 
                            TfLiteProfilerEventType event_type) {
    ExoTfLiteProfiler* p = (ExoTfLiteProfiler*)user_data;
    
    if (event_type == kTfLiteProfilerOpInvoke) {
        exo_span_t* span = exo_start_span(tag);
        exo_span_set_attribute_string(span, "ai.framework", "tflite");
        p->spans[p->span_count++] = span;
    }
}

void exo_tflite_end_event(void* user_data, const char* tag,
                          TfLiteProfilerEventType event_type) {
    ExoTfLiteProfiler* p = (ExoTfLiteProfiler*)user_data;
    
    if (event_type == kTfLiteProfilerOpInvoke && p->span_count > 0) {
        exo_span_end(p->spans[--p->span_count]);
    }
}

// Usage
ExoTfLiteProfiler profiler = {0};
TfLiteInterpreterOptions* options = TfLiteInterpreterOptionsCreate();
TfLiteInterpreterOptionsSetProfiler(options, 
    exo_tflite_begin_event, 
    exo_tflite_end_event, 
    &profiler);
```

### ONNX Runtime Integration

```cpp
#include <onnxruntime/core/session/onnxruntime_cxx_api.h>
#include <exo/exo.hpp>

class ExoOrtProfiler : public Ort::ISessionProfiler {
public:
    void OnSessionStart(const std::string& session_name) override {
        session_span_ = exo::start_span("ort_session");
        session_span_.SetAttribute("session.name", session_name);
    }
    
    void OnSessionEnd() override {
        session_span_.End();
    }
    
    void OnOpStart(const std::string& op_name, 
                   const std::string& op_type) override {
        op_spans_.push(exo::start_span(op_name));
        op_spans_.top().SetAttribute("ai.operation.type", op_type);
        op_spans_.top().SetAttribute("ai.framework", "onnxruntime");
    }
    
    void OnOpEnd(const std::string& op_name) override {
        if (!op_spans_.empty()) {
            op_spans_.top().End();
            op_spans_.pop();
        }
    }
    
private:
    exo::Span session_span_;
    std::stack<exo::Span> op_spans_;
};

// Usage
Ort::Env env(ORT_LOGGING_LEVEL_WARNING, "exo_example");
Ort::SessionOptions session_options;
auto profiler = std::make_unique<ExoOrtProfiler>();
session_options.SetProfiler(profiler.get());
```

### Custom Inference Engine Integration

```c
// Example: Custom NPU inference engine

typedef struct {
    model_t* model;
    npu_context_t* npu_ctx;
} inference_engine_t;

// Comprehensive tracing for custom engine
void run_inference_traced(inference_engine_t* engine, 
                          input_buffer_t* input,
                          output_buffer_t* output) {
    // Top-level inference span
    exo_span_t* inference = exo_start_span("inference");
    exo_span_set_attribute_string(inference, "ai.model.name", engine->model->name);
    exo_span_set_attribute_int(inference, "ai.batch_size", input->batch_size);
    
    // Input preprocessing
    {
        exo_span_t* preprocess = exo_start_child_span("preprocess");
        exo_span_set_attribute_string(preprocess, "ai.device", "cpu");
        
        preprocess_input(input);
        
        exo_span_end(preprocess);
    }
    
    // DMA transfer to NPU
    {
        exo_span_t* dma_in = exo_start_child_span("dma_input");
        exo_span_set_attribute_string(dma_in, "ai.device", "dma");
        exo_span_set_attribute_int(dma_in, "ai.memory.write_bytes", input->size);
        
        dma_transfer_to_npu(engine->npu_ctx, input);
        
        exo_span_end(dma_in);
    }
    
    // Execute each layer
    for (int i = 0; i < engine->model->num_layers; i++) {
        layer_t* layer = &engine->model->layers[i];
        
        exo_span_t* layer_span = exo_start_child_span(layer->name);
        exo_span_set_attribute_string(layer_span, "ai.operation.type", 
                                      get_op_type_string(layer->op_type));
        exo_span_set_attribute_string(layer_span, "ai.device", "npu");
        exo_span_set_attribute_int(layer_span, "ai.flops", layer->flops);
        
        // Add shape information
        char shape_buf[64];
        snprintf(shape_buf, sizeof(shape_buf), "[%d,%d,%d,%d]",
                 layer->input_shape[0], layer->input_shape[1],
                 layer->input_shape[2], layer->input_shape[3]);
        exo_span_set_attribute_string(layer_span, "ai.input.shape", shape_buf);
        
        // Execute on NPU
        npu_execute_layer(engine->npu_ctx, layer);
        
        exo_span_end(layer_span);
    }
    
    // DMA transfer from NPU
    {
        exo_span_t* dma_out = exo_start_child_span("dma_output");
        exo_span_set_attribute_string(dma_out, "ai.device", "dma");
        exo_span_set_attribute_int(dma_out, "ai.memory.read_bytes", output->size);
        
        dma_transfer_from_npu(engine->npu_ctx, output);
        
        exo_span_end(dma_out);
    }
    
    // Output postprocessing
    {
        exo_span_t* postprocess = exo_start_child_span("postprocess");
        exo_span_set_attribute_string(postprocess, "ai.device", "cpu");
        
        postprocess_output(output);
        
        exo_span_end(postprocess);
    }
    
    exo_span_end(inference);
}
```

---

## Semantic Conventions for AI

### Proposed Attribute Names

Following OpenTelemetry naming conventions:

#### Model Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.model.name` | string | Model identifier | "resnet50" |
| `ai.model.version` | string | Model version | "1.0.0" |
| `ai.model.format` | string | Model format | "tflite", "onnx" |
| `ai.framework` | string | Inference framework | "tflite", "onnxruntime" |
| `ai.framework.version` | string | Framework version | "2.15.0" |

#### Operation Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.operation.type` | string | Operation type | "conv2d", "matmul" |
| `ai.operation.name` | string | Operation name in graph | "layer1/conv1" |
| `ai.batch_size` | int | Batch size | 32 |
| `ai.dtype` | string | Data type | "float16", "int8" |
| `ai.input.shape` | string | Input tensor shape | "[1,224,224,3]" |
| `ai.output.shape` | string | Output tensor shape | "[1,1000]" |

#### Compute Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.flops` | int | Floating point operations | 1000000 |
| `ai.macs` | int | Multiply-accumulate ops | 500000 |
| `ai.params` | int | Parameter count | 25000000 |

#### Memory Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.memory.read_bytes` | int | Bytes read | 4096 |
| `ai.memory.write_bytes` | int | Bytes written | 1024 |
| `ai.memory.workspace_bytes` | int | Scratch memory used | 8192 |
| `ai.memory.activation_bytes` | int | Activation memory | 2048 |

#### Hardware Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.device` | string | Execution device | "npu", "cpu", "gpu" |
| `ai.device.id` | string | Device identifier | "npu0" |
| `ai.hardware.utilization` | double | Utilization (0-1) | 0.85 |
| `ai.hardware.power_mw` | int | Power in milliwatts | 500 |
| `ai.hardware.temperature_c` | double | Temperature in Celsius | 45.5 |

#### Convolution-Specific

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.conv.kernel_height` | int | Kernel height | 3 |
| `ai.conv.kernel_width` | int | Kernel width | 3 |
| `ai.conv.stride_height` | int | Stride height | 1 |
| `ai.conv.stride_width` | int | Stride width | 1 |
| `ai.conv.padding` | string | Padding type | "same", "valid" |
| `ai.conv.groups` | int | Groups (for depthwise) | 1 |

#### Attention-Specific

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `ai.attention.num_heads` | int | Number of heads | 8 |
| `ai.attention.head_dim` | int | Dimension per head | 64 |
| `ai.attention.seq_length` | int | Sequence length | 512 |
| `ai.attention.type` | string | Attention type | "self", "cross" |

---

## Best Practices

### 1. Hierarchical Tracing

Create a meaningful hierarchy:

```
inference                           (top-level)
├── preprocess                      (stage)
│   ├── decode_image               (operation)
│   └── normalize                   (operation)
├── model_execution                 (stage)
│   ├── layer_0/conv2d             (operator)
│   ├── layer_0/batchnorm          (operator)
│   ├── layer_0/relu               (operator)
│   ├── layer_1/conv2d             (operator)
│   │   ├── dma_input              (hardware op)
│   │   ├── npu_compute            (hardware op)
│   │   └── dma_output             (hardware op)
│   └── ...
└── postprocess                     (stage)
    ├── softmax                     (operation)
    └── decode_output               (operation)
```

### 2. Attribute Consistency

Use consistent attribute names across all spans:

```c
// Good: Consistent naming
exo_span_set_attribute_string(span, "ai.model.name", "resnet50");
exo_span_set_attribute_string(span, "ai.operation.type", "conv2d");
exo_span_set_attribute_string(span, "ai.device", "npu");

// Bad: Inconsistent naming
exo_span_set_attribute_string(span, "model", "resnet50");
exo_span_set_attribute_string(span, "op_type", "conv2d");
exo_span_set_attribute_string(span, "hw", "npu");
```

### 3. Include Computational Metadata

Always include data needed for performance analysis:

```c
exo_span_t* span = exo_start_span("matmul");

// Essential for roofline analysis
exo_span_set_attribute_int(span, "ai.flops", m * n * k * 2);  // 2 ops per MAC
exo_span_set_attribute_int(span, "ai.memory.read_bytes", 
    (m * k + k * n) * sizeof(float));
exo_span_set_attribute_int(span, "ai.memory.write_bytes", 
    m * n * sizeof(float));

// Shape information
char shape[64];
snprintf(shape, sizeof(shape), "[%d,%d]*[%d,%d]", m, k, k, n);
exo_span_set_attribute_string(span, "ai.operation.shape", shape);
```

### 4. Handle Asynchronous Operations

For async hardware operations:

```c
// Start async operation
exo_span_t* span = exo_start_span("npu_execute_async");
exo_span_set_attribute_string(span, "ai.device", "npu");
exo_span_set_attribute_bool(span, "ai.async", true);

npu_submit_command(cmd);

// Record submit event
exo_span_add_event(span, "submitted");

// ... later, when complete ...

// Record completion event
exo_span_add_event(span, "completed");
exo_span_end(span);
```

### 5. Batch Tracing Control

Allow control over tracing granularity:

```c
typedef enum {
    TRACE_LEVEL_NONE,      // No tracing
    TRACE_LEVEL_INFERENCE, // Only top-level inference
    TRACE_LEVEL_LAYER,     // Inference + layers
    TRACE_LEVEL_OPERATOR,  // + individual operators
    TRACE_LEVEL_HARDWARE,  // + hardware operations
    TRACE_LEVEL_FULL,      // Everything
} trace_level_t;

// Check level before creating spans
if (g_trace_level >= TRACE_LEVEL_OPERATOR) {
    span = exo_start_span("conv2d");
}
```

---

## Common Instrumentation Patterns

### Memory Allocation Tracking

```c
void* ai_alloc_traced(size_t size, const char* purpose) {
    exo_span_t* span = exo_start_span("memory_alloc");
    exo_span_set_attribute_int(span, "ai.memory.size_bytes", size);
    exo_span_set_attribute_string(span, "ai.memory.purpose", purpose);
    
    void* ptr = ai_alloc(size);
    
    exo_span_set_attribute_string(span, "ai.memory.address", 
                                  format_ptr(ptr));
    exo_span_end(span);
    
    return ptr;
}
```

### Error Handling

```c
void run_operator_traced(operator_t* op) {
    exo_span_t* span = exo_start_span(op->name);
    
    int result = run_operator(op);
    
    if (result != SUCCESS) {
        exo_span_set_status(span, EXO_SPAN_STATUS_ERROR, 
                           get_error_string(result));
        exo_span_record_exception(span, "OperatorError", 
                                 get_error_string(result));
    }
    
    exo_span_end(span);
}
```

### Benchmark Mode

```c
// Specialized tracing for benchmarks
void benchmark_inference(model_t* model, int warmup, int iterations) {
    // Warmup (optional tracing)
    exo_set_sampler(exo_sampler_always_off_create());
    for (int i = 0; i < warmup; i++) {
        run_inference(model);
    }
    
    // Benchmark (full tracing)
    exo_set_sampler(exo_sampler_always_on_create());
    
    exo_span_t* bench = exo_start_span("benchmark");
    exo_span_set_attribute_string(bench, "benchmark.model", model->name);
    exo_span_set_attribute_int(bench, "benchmark.iterations", iterations);
    
    for (int i = 0; i < iterations; i++) {
        exo_span_t* iter = exo_start_child_span("iteration");
        exo_span_set_attribute_int(iter, "benchmark.iteration", i);
        
        run_inference(model);
        
        exo_span_end(iter);
    }
    
    exo_span_end(bench);
    exo_flush();
}
```

---

## Appendix: Quick Reference

### Initialization

```c
#include <exo/exo.h>

exo_config_t config = {
    .service_name = "my-inference-engine",
    .service_version = "1.0.0",
};
exo_init(&config);

// At shutdown
exo_shutdown();
```

### Basic Tracing

```c
exo_span_t* span = exo_start_span("operation");
exo_span_set_attribute_string(span, "key", "value");
// ... work ...
exo_span_end(span);
```

### AI-Specific Tracing

```c
#include <exo/ai.h>

exo_span_t* span = exo_ai_start_op_span(EXO_AI_OP_CONV2D);
exo_ai_set_input_shape(span, shape, 4);
exo_span_set_attribute_int(span, "ai.flops", 1000000);
// ... work ...
exo_span_end(span);
```

### Export to Perfetto

```c
#include <exo/exporters/perfetto.h>

exo_span_exporter_t* perfetto = exo_perfetto_file_exporter_create("trace.perfetto");
exo_register_span_exporter(perfetto);
```
