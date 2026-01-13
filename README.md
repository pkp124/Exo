# Exo - Execution Observability for AI Firmware

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Design%20Phase-yellow.svg)]()

**Exo** provides execution observability and performance intelligence for SoC firmware, with a focus on AI inference stacks. It bridges the gap between traditional application performance monitoring (APM) and the unique requirements of embedded AI systems.

## 🎯 Vision

Apply modern observability principles to firmware development:
- **Traces** → Understand execution flow across CPU, NPU, DMA, and other hardware units
- **Metrics** → Track performance counters, utilization, and resource consumption
- **Logs** → Correlate debug output with execution context
- **Models** → Build performance models for architecture exploration

## ✨ Key Features

### Open Standards First
- **OpenTelemetry compatible** - Standard APIs, standard data model
- **Perfetto support** - Native export for excellent timeline visualization
- **W3C Trace Context** - Standard distributed trace propagation
- **OTLP protocol** - Export to Jaeger, Grafana Tempo, Zipkin, and more
- **Prometheus compatible** - Metrics in standard format

### Pluggable Architecture
Every component can be replaced with alternatives from the ecosystem:
- Use libexo or OpenTelemetry C++ SDK for instrumentation
- Export to any OTLP-compatible backend
- Replace samplers, exporters, or the entire provider

### Firmware Optimized
- Minimal memory footprint (< 16KB RAM for core)
- No dynamic allocation in hot paths
- Deterministic latency for tracing calls
- Compile-time disable for zero-overhead production builds

### AI-Specific Extensions
- Pre-built instrumentation for common AI operators
- Semantic conventions for neural network operations
- Tensor lifecycle tracking
- Hardware counter correlation

## 📚 Documentation

### Design Documents

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture/ARCHITECTURE.md) | High-level system architecture and design goals |
| [Tracing Library Design](docs/design/TRACING_LIBRARY.md) | Core tracing API and implementation design |
| [Metrics & Logging](docs/design/METRICS_LOGGING.md) | Metrics and logging subsystem design |
| [Data Formats](docs/specs/DATA_FORMATS.md) | Wire formats: Perfetto, OTLP, Chrome Trace |
| [Performance Modeling](docs/design/PERFORMANCE_MODELING.md) | Building performance models from traces |
| [AI Integration Guide](docs/guides/AI_INTEGRATION.md) | Integrating with AI frameworks and engines |

## 🏗️ Project Structure

```
exo/
├── docs/
│   ├── architecture/     # Architecture overview
│   ├── design/           # Detailed design specs
│   ├── specs/            # Data format specifications
│   └── guides/           # Integration guides
├── libexo/               # Core C/C++ library (planned)
├── collectors/           # Trace collectors (planned)
├── analysis/             # Python analysis tools (planned)
└── examples/             # Usage examples (planned)
```

## 🗺️ Roadmap

### Phase 1: Design (Current)
- [x] Architecture overview
- [x] Tracing library design
- [x] Metrics and logging design
- [x] Data format specifications (Perfetto, OTLP, Chrome Trace)
- [x] Performance modeling design
- [x] AI integration guide

### Phase 2: Core Library
- [ ] Implement libexo core (C11)
- [ ] Span management and context propagation
- [ ] Memory pools and ring buffers
- [ ] Clock abstraction

### Phase 3: Exporters
- [ ] Perfetto exporter
- [ ] OTLP exporter
- [ ] Chrome Trace exporter
- [ ] Binary format exporter

### Phase 4: Analysis Tools
- [ ] Python trace analysis library
- [ ] Perfetto trace parsing
- [ ] Roofline model generation
- [ ] Performance report generation

### Phase 5: Advanced Features
- [ ] Hardware counter integration
- [ ] AI operator semantic conventions
- [ ] Architecture exploration tools
- [ ] What-if analysis framework

## 🔧 Planned Usage

```c
#include <exo/exo.h>

int main() {
    // Initialize with Perfetto export
    exo_init(&(exo_config_t){
        .service_name = "my_inference_engine",
        .exporter = exo_perfetto_file_exporter_create("trace.perfetto"),
    });
    
    // Trace inference
    exo_span_t* span = exo_start_span("inference");
    exo_span_set_attribute_string(span, "ai.model.name", "resnet50");
    
    run_inference(model, input, output);
    
    exo_span_end(span);
    exo_shutdown();
    
    // View in Perfetto UI: https://ui.perfetto.dev
    return 0;
}
```

## 📊 Visualization

Traces can be viewed in multiple tools:

- **[Perfetto UI](https://ui.perfetto.dev)** - Recommended for timeline visualization
- **[Jaeger](https://www.jaegertracing.io/)** - Distributed tracing UI
- **[Grafana](https://grafana.com/)** - Dashboards with Tempo backend
- **Chrome** - `chrome://tracing` for Chrome Trace format

## 🤝 Contributing

This project is in the design phase. Contributions to design discussions, documentation improvements, and implementation are welcome.

## 📄 License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

## 🔗 Related Projects

- [OpenTelemetry](https://opentelemetry.io/) - Observability framework
- [Perfetto](https://perfetto.dev/) - System profiling and tracing
- [Jaeger](https://www.jaegertracing.io/) - Distributed tracing platform
- [Grafana Tempo](https://grafana.com/oss/tempo/) - Trace storage backend
