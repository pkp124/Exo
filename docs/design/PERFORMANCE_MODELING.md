# Performance Modeling Design

**Building Performance Models from Execution Traces**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Goals and Use Cases](#goals-and-use-cases)
3. [Data Collection Strategy](#data-collection-strategy)
4. [Model Types](#model-types)
5. [Architecture Exploration](#architecture-exploration)
6. [Analysis Pipeline](#analysis-pipeline)
7. [Visualization and Reporting](#visualization-and-reporting)

---

## Overview

Performance modeling transforms execution traces into predictive models that can:
- **Characterize workloads** - Understand compute, memory, and I/O patterns
- **Identify bottlenecks** - Find limiting factors in the system
- **Enable what-if analysis** - Predict impact of HW/SW changes
- **Guide architecture decisions** - Data-driven hardware exploration

### From Traces to Models

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Trace Collection                                  │
│                                                                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ Run 1   │ │ Run 2   │ │ Run 3   │ │ Run N   │ │ Perf    │           │
│  │ Trace   │ │ Trace   │ │ Trace   │ │ Trace   │ │ Counters│           │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │
│       │           │           │           │           │                 │
│       └───────────┴───────────┴───────────┴───────────┘                 │
│                               │                                          │
└───────────────────────────────┼──────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        Analysis Pipeline                                   │
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐   │
│  │ Trace Parsing   │─▶│ Feature         │─▶│ Model Building          │   │
│  │ & Aggregation   │  │ Extraction      │  │                         │   │
│  └─────────────────┘  └─────────────────┘  └────────────┬────────────┘   │
│                                                          │                │
└──────────────────────────────────────────────────────────┼────────────────┘
                                                           │
                                                           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        Performance Models                                  │
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐   │
│  │ Roofline Model  │  │ Bottleneck      │  │ Prediction Model        │   │
│  │ (Compute/Mem)   │  │ Analysis        │  │ (What-if)               │   │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘   │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        Architecture Exploration                            │
│                                                                            │
│  "What if NPU had 2x compute?"    "What if memory bandwidth doubled?"     │
│  "What if we added a second DMA?" "What latency can we achieve?"          │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Goals and Use Cases

### Primary Goals

| Goal | Description | Deliverable |
|------|-------------|-------------|
| **Workload Characterization** | Understand what the workload demands | Compute/memory/IO profiles |
| **Bottleneck Identification** | Find limiting resources | Bottleneck reports, Roofline plots |
| **Performance Prediction** | Estimate performance under changes | Prediction models |
| **Optimization Guidance** | Recommend improvements | Prioritized recommendations |

### Use Cases

#### 1. AI Inference Optimization

```
Question: "Why is my inference slow?"

Analysis Flow:
1. Collect traces with hardware counters
2. Build per-operator profiles
3. Identify: Is it compute-bound? Memory-bound? Synchronization-bound?
4. Output: "Conv2d layer 5 is memory-bound. Memory bandwidth utilization is 95%."
```

#### 2. Hardware Architecture Exploration

```
Question: "Should we double NPU compute or memory bandwidth?"

Analysis Flow:
1. Collect traces across multiple workloads
2. Build parameterized performance model
3. Simulate both scenarios
4. Output: "2x memory bandwidth improves average throughput by 40%.
           2x compute improves throughput by 15%."
```

#### 3. Firmware Optimization

```
Question: "Which operators should we optimize first?"

Analysis Flow:
1. Collect traces with timing breakdown
2. Rank operators by total execution time
3. Analyze optimization potential (how far from theoretical peak?)
4. Output: "Top 3 optimization targets:
           1. MatMul (35% of time, 60% of peak efficiency)
           2. Attention (25% of time, 40% of peak efficiency)
           3. Conv2d (20% of time, 85% of peak efficiency)"
```

#### 4. Regression Detection

```
Question: "Did this firmware update cause a regression?"

Analysis Flow:
1. Compare traces from before and after
2. Statistical analysis of timing distributions
3. Identify significant differences
4. Output: "Regression detected: LayerNorm latency increased by 15% (p<0.01)"
```

---

## Data Collection Strategy

### Required Data Points

For effective performance modeling, collect:

| Data Category | Examples | Collection Method |
|---------------|----------|-------------------|
| **Timing** | Span durations, event timestamps | Tracing API |
| **Compute** | FLOPS, operations count | Attributes |
| **Memory** | Bytes read/written, allocations | Attributes, events |
| **Hardware Counters** | Cycles, cache misses, bandwidth | HW counter sampling |
| **Resource Utilization** | NPU %, memory %, DMA % | Metrics |
| **Workload Metadata** | Model name, batch size, shapes | Attributes |

### Trace Annotations for Modeling

```c
// Annotate spans with data needed for modeling
exo_span_t* span = exo_start_span("matmul");

// Compute characteristics
exo_span_set_attribute_int(span, "ai.flops", 1000000);
exo_span_set_attribute_string(span, "ai.operation.type", "matmul");

// Memory characteristics
exo_span_set_attribute_int(span, "ai.memory.read_bytes", 4096);
exo_span_set_attribute_int(span, "ai.memory.write_bytes", 1024);

// Tensor shapes (for computing theoretical requirements)
exo_span_set_attribute_string(span, "ai.input.shape", "[1024,512]");
exo_span_set_attribute_string(span, "ai.output.shape", "[1024,256]");

// Hardware assignment
exo_span_set_attribute_string(span, "ai.device", "npu");

// ... execute operation ...

exo_span_end(span);
```

### Hardware Counter Collection

```c
// Enable hardware counter sampling
exo_hw_counters_config_t hw_config = {
    .sample_cycles = true,
    .sample_instructions = true,
    .sample_cache_misses = true,
    .sample_memory_bandwidth = true,
    .sample_interval_us = 100,  // Sample every 100µs
};
exo_enable_hw_counters(&hw_config);

// Counters are automatically attached to spans
```

### Benchmark Suite Integration

```c
// Run benchmark with comprehensive data collection
void run_benchmark(const char* model_name, int iterations) {
    for (int i = 0; i < iterations; i++) {
        exo_span_t* span = exo_start_span("benchmark_iteration");
        exo_span_set_attribute_string(span, "benchmark.model", model_name);
        exo_span_set_attribute_int(span, "benchmark.iteration", i);
        
        run_inference(model);
        
        exo_span_end(span);
    }
    
    // Flush to ensure all data is exported
    exo_flush();
}
```

---

## Model Types

### 1. Roofline Model

The Roofline model characterizes performance relative to hardware limits:

```
Performance (FLOPS/s)
        │
        │         ╱ Peak Compute
        │        ╱
        │       ╱
        │      ╱
        │     ╱ ← Memory Bound Region │ Compute Bound Region
        │    ╱
        │   ╱____________________________________
        │  ╱  Peak Memory Bandwidth
        │ ╱
        │╱
        └────────────────────────────────────────
          Arithmetic Intensity (FLOPS/Byte)
```

```python
from exo_analysis.models import RooflineModel

# Build roofline from traces
roofline = RooflineModel.from_traces(
    traces,
    peak_compute_gflops=100,
    peak_memory_bandwidth_gbps=50
)

# Plot operations on roofline
roofline.plot(
    output="roofline.png",
    annotate_ops=True,
    show_bound_regions=True
)

# Identify bottlenecks
for op in roofline.memory_bound_ops:
    print(f"{op.name}: Memory bound, achieving {op.efficiency:.1%} of peak")

for op in roofline.compute_bound_ops:
    print(f"{op.name}: Compute bound, achieving {op.efficiency:.1%} of peak")
```

### 2. Critical Path Analysis

Identify the longest path through the execution graph:

```python
from exo_analysis.models import CriticalPathAnalysis

# Analyze critical path
cpa = CriticalPathAnalysis.from_traces(traces)

# Get critical path
critical_path = cpa.get_critical_path()
print(f"Critical path length: {critical_path.total_duration_ms}ms")

for span in critical_path.spans:
    print(f"  {span.name}: {span.duration_ms}ms ({span.percentage:.1%})")

# Identify opportunities for parallelization
parallel_ops = cpa.get_parallelization_opportunities()
for op in parallel_ops:
    print(f"  {op.name}: Could run in parallel with {op.parallel_with}")
```

### 3. Queueing Model

Model the system as a network of queues:

```python
from exo_analysis.models import QueueingModel

# Build queueing model
qm = QueueingModel.from_traces(traces)

# Define service centers
qm.add_service_center("cpu", service_rate=1000)  # ops/sec
qm.add_service_center("npu", service_rate=10000)
qm.add_service_center("dma", service_rate=50000)

# Analyze utilization
for center in qm.service_centers:
    print(f"{center.name}: {center.utilization:.1%} utilization, "
          f"avg queue length: {center.avg_queue_length:.2f}")

# Find bottleneck
bottleneck = qm.get_bottleneck()
print(f"Bottleneck: {bottleneck.name} ({bottleneck.utilization:.1%})")
```

### 4. Regression Model

Predict performance from workload characteristics:

```python
from exo_analysis.models import PerformanceRegression

# Build regression model
model = PerformanceRegression.from_traces(traces)

# Features: batch_size, input_size, num_layers, etc.
# Target: latency_ms

# Predict for new workload
prediction = model.predict({
    "batch_size": 64,
    "input_height": 224,
    "input_width": 224,
    "num_layers": 50
})
print(f"Predicted latency: {prediction.latency_ms}ms")
print(f"Prediction interval: [{prediction.ci_lower}ms, {prediction.ci_upper}ms]")

# Feature importance
for feature, importance in model.feature_importance:
    print(f"  {feature}: {importance:.2f}")
```

### 5. Analytical Model

Build analytical models based on hardware specifications:

```python
from exo_analysis.models import AnalyticalModel

# Define hardware model
hw = HardwareModel(
    compute_units=4,
    compute_frequency_ghz=1.0,
    ops_per_cycle_per_unit=256,  # SIMD width
    memory_bandwidth_gbps=50,
    memory_latency_ns=100,
    dma_channels=2,
    dma_bandwidth_gbps=25
)

# Define workload from traces
workload = Workload.from_traces(traces)

# Build analytical model
model = AnalyticalModel(hw, workload)

# Predict performance
prediction = model.predict()
print(f"Predicted latency: {prediction.latency_ms}ms")
print(f"Limiting factor: {prediction.bottleneck}")
print(f"Compute utilization: {prediction.compute_utilization:.1%}")
print(f"Memory utilization: {prediction.memory_utilization:.1%}")
```

---

## Architecture Exploration

### What-If Analysis

```python
from exo_analysis.explore import ArchitectureExplorer

# Load baseline traces
baseline = TraceCollection.load("baseline_traces/")

# Create explorer
explorer = ArchitectureExplorer(baseline)

# Define exploration space
explorer.add_parameter("npu_compute_gflops", [50, 100, 200, 400])
explorer.add_parameter("memory_bandwidth_gbps", [25, 50, 100])
explorer.add_parameter("dma_channels", [1, 2, 4])

# Run exploration
results = explorer.explore()

# Analyze results
results.plot_pareto_frontier(
    x="cost",  # Estimated silicon area
    y="latency_ms",
    output="pareto.png"
)

# Find optimal configurations
optimal = results.get_optimal(
    objective="latency_ms",
    constraints={"cost": ("<=", 1.5)}  # Max 1.5x baseline cost
)
print(f"Optimal config: {optimal.config}")
print(f"Predicted latency: {optimal.latency_ms}ms")
```

### Hardware Configuration Comparison

```python
# Compare different hardware configurations
configs = {
    "baseline": HardwareModel(compute=100, memory_bw=50),
    "compute_2x": HardwareModel(compute=200, memory_bw=50),
    "memory_2x": HardwareModel(compute=100, memory_bw=100),
    "balanced": HardwareModel(compute=150, memory_bw=75),
}

comparison = explorer.compare_configs(configs)
comparison.plot_comparison("comparison.png")
comparison.to_csv("comparison.csv")
```

### Sensitivity Analysis

```python
# Analyze sensitivity to different parameters
sensitivity = explorer.sensitivity_analysis(
    parameter="memory_bandwidth_gbps",
    range=(25, 200),
    steps=20
)

sensitivity.plot("sensitivity_memory_bw.png")
print(f"Sensitivity coefficient: {sensitivity.coefficient}")
# Coefficient > 1: Super-linear improvement (bottleneck)
# Coefficient ≈ 1: Linear improvement
# Coefficient < 1: Diminishing returns
```

---

## Analysis Pipeline

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Analysis Pipeline                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Stage 1: Ingest                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - Load traces (Perfetto, OTLP, Exo binary)                     │   │
│  │  - Validate schema                                               │   │
│  │  - Handle multiple runs                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  Stage 2: Transform                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - Parse spans, events, counters                                │   │
│  │  - Build execution DAG                                          │   │
│  │  - Correlate with hardware counters                             │   │
│  │  - Aggregate statistics                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  Stage 3: Analyze                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - Build performance models                                     │   │
│  │  - Detect bottlenecks                                           │   │
│  │  - Compare against baselines                                    │   │
│  │  - Statistical analysis                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  Stage 4: Report                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - Generate visualizations                                      │   │
│  │  - Create reports                                               │   │
│  │  - Export to various formats                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Python API

```python
from exo_analysis import Pipeline, stages

# Define analysis pipeline
pipeline = Pipeline([
    stages.LoadTraces("traces/"),
    stages.ParseSpans(),
    stages.ExtractOperatorStats(),
    stages.BuildRooflineModel(
        peak_compute=100,
        peak_memory_bw=50
    ),
    stages.DetectBottlenecks(),
    stages.GenerateReport("report.html"),
])

# Run pipeline
results = pipeline.run()

# Access results
print(results.roofline_model)
print(results.bottlenecks)
print(results.summary)
```

### CLI Interface

```bash
# Run analysis pipeline
exo analyze traces/ \
    --model roofline \
    --peak-compute 100 \
    --peak-memory-bw 50 \
    --output report.html

# Compare two trace sets
exo compare baseline/ optimized/ \
    --output comparison.html

# Architecture exploration
exo explore traces/ \
    --sweep compute:50,100,200 \
    --sweep memory_bw:25,50,100 \
    --output exploration.html

# Regression detection
exo regression baseline/ current/ \
    --threshold 5% \
    --output regression.html
```

---

## Visualization and Reporting

### Built-in Visualizations

| Visualization | Description | Use Case |
|---------------|-------------|----------|
| **Roofline Plot** | Ops plotted against hardware limits | Bottleneck identification |
| **Timeline View** | Execution timeline with tracks | Understanding parallelism |
| **Flame Graph** | Hierarchical time breakdown | Finding hot spots |
| **Operator Breakdown** | Pie/bar chart of time by operator | Time distribution |
| **Latency Distribution** | Histogram of latencies | Variance analysis |
| **Correlation Heatmap** | Parameter vs performance | Sensitivity analysis |
| **Pareto Frontier** | Cost vs performance trade-offs | Architecture exploration |

### Perfetto Integration

```python
from exo_analysis.viz import PerfettoAnnotator

# Add analysis annotations to Perfetto trace
annotator = PerfettoAnnotator("trace.perfetto")

# Add bottleneck markers
annotator.add_markers(bottlenecks)

# Add efficiency annotations
annotator.add_efficiency_overlay(roofline_model)

# Save annotated trace
annotator.save("trace_annotated.perfetto")

# Open in Perfetto UI with analysis visible
```

### Report Generation

```python
from exo_analysis.reports import ReportBuilder

# Build comprehensive report
report = ReportBuilder() \
    .add_summary(results) \
    .add_roofline_plot(roofline) \
    .add_bottleneck_analysis(bottlenecks) \
    .add_operator_breakdown(op_stats) \
    .add_recommendations(recommendations) \
    .build()

# Export formats
report.to_html("report.html")
report.to_pdf("report.pdf")
report.to_markdown("report.md")
```

### Example Report Structure

```markdown
# Performance Analysis Report

## Executive Summary
- Total inference time: 45.2ms
- Primary bottleneck: Memory bandwidth (NPU)
- Efficiency: 65% of theoretical peak

## Roofline Analysis
[Roofline Plot]
- 3 operators are memory-bound
- 12 operators are compute-bound
- Average arithmetic intensity: 25 FLOPS/byte

## Bottleneck Analysis
| Rank | Operator | Time (ms) | % Total | Bottleneck |
|------|----------|-----------|---------|------------|
| 1 | Attention | 12.5 | 28% | Memory |
| 2 | MatMul | 10.2 | 23% | Compute |
| 3 | Conv2d | 8.1 | 18% | Compute |

## Recommendations
1. **Optimize Attention memory access patterns** - Potential 20% improvement
2. **Enable operator fusion for MatMul+ReLU** - Potential 10% improvement
3. **Increase batch size** - Better compute utilization

## Appendix
- Hardware configuration
- Detailed operator statistics
- Raw data tables
```

---

## Implementation Roadmap

### Phase 1: Foundation
- [ ] Trace parsing for Perfetto, OTLP, Exo Binary
- [ ] Basic statistical analysis
- [ ] Operator breakdown visualization
- [ ] CLI interface

### Phase 2: Modeling
- [ ] Roofline model implementation
- [ ] Critical path analysis
- [ ] Bottleneck detection
- [ ] Report generation

### Phase 3: Exploration
- [ ] What-if analysis framework
- [ ] Parameter sweep infrastructure
- [ ] Pareto analysis
- [ ] Sensitivity analysis

### Phase 4: Advanced
- [ ] Machine learning-based prediction models
- [ ] Automatic optimization recommendations
- [ ] Integration with hardware simulators
- [ ] Continuous performance monitoring
