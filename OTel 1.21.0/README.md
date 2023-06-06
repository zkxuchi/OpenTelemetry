# OpenTelemetry 规范 1.21.0

## 目录

- [概述](./overview.md)
- [Glossary](https://opentelemetry.io/docs/specs/otel/glossary/)
- [Versioning and stability for OpenTelemetry clients](https://opentelemetry.io/docs/specs/otel/versioning-and-stability/)
- Library Guidelines
  - [Package/Library Layout](https://opentelemetry.io/docs/specs/otel/library-layout/)
  - [General error handling guidelines](https://opentelemetry.io/docs/specs/otel/error-handling/)
- API Specification
  - Context
    - [Propagators](https://opentelemetry.io/docs/specs/otel/context/api-propagators/)
  - [Baggage](https://opentelemetry.io/docs/specs/otel/baggage/api/)
  - [Tracing](https://opentelemetry.io/docs/specs/otel/trace/api/)
  - [Metrics](https://opentelemetry.io/docs/specs/otel/metrics/api/)
  - Logs
    - [Bridge API](https://opentelemetry.io/docs/specs/otel/logs/bridge-api/)
    - [Event API](https://opentelemetry.io/docs/specs/otel/logs/event-api/)
- SDK Specification
  - [Tracing](https://opentelemetry.io/docs/specs/otel/trace/sdk/)
  - [Metrics](https://opentelemetry.io/docs/specs/otel/metrics/sdk/)
  - [Logs](https://opentelemetry.io/docs/specs/otel/logs/sdk/)
  - [Resource](https://opentelemetry.io/docs/specs/otel/resource/sdk/)
  - [Configuration](https://opentelemetry.io/docs/specs/otel/configuration/sdk-configuration/)
- Data Specification
  - [Semantic Conventions](https://opentelemetry.io/docs/specs/otel/overview/#semantic-conventions)
  - Protocol
    - [Metrics](https://opentelemetry.io/docs/specs/otel/metrics/data-model/)
    - [Logs](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
  - Compatibility
    - [OpenCensus](https://opentelemetry.io/docs/specs/otel/compatibility/opencensus/)
    - [OpenTracing](https://opentelemetry.io/docs/specs/otel/compatibility/opentracing/)
    - [Prometheus and OpenMetrics](https://opentelemetry.io/docs/specs/otel/compatibility/prometheus_and_openmetrics/)
    - [Trace Context in non-OTLP Log Formats](https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/)
