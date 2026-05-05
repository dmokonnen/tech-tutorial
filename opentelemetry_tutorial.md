# Comprehensive OpenTelemetry Tutorial

---

## Table of Contents

1. [What is OpenTelemetry?](#1-what-is-opentelemetry)
2. [Core Concepts & Architecture](#2-core-concepts--architecture)
3. [The Three Pillars: Traces, Metrics, Logs](#3-the-three-pillars-traces-metrics-logs)
4. [OpenTelemetry Collector](#4-opentelemetry-collector)
5. [Instrumentation — Auto vs Manual](#5-instrumentation--auto-vs-manual)
6. [Language SDKs](#6-language-sdks)
7. [Context Propagation](#7-context-propagation)
8. [Exporters & Backends](#8-exporters--backends)
9. [Kubernetes & Helm Deployment](#9-kubernetes--helm-deployment)
10. [Sampling Strategies](#10-sampling-strategies)
11. [Semantic Conventions](#11-semantic-conventions)
12. [Resource Attributes](#12-resource-attributes)
13. [Baggage](#13-baggage)
14. [Integrating with Prometheus & Grafana](#14-integrating-with-prometheus--grafana)
15. [Production Best Practices](#15-production-best-practices)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. What is OpenTelemetry?

OpenTelemetry (OTel) is a **vendor-neutral, open-source observability framework** for generating, collecting, and exporting telemetry data — **traces, metrics, and logs** — from your applications and infrastructure. It was formed by merging OpenCensus and OpenTracing in 2019 and is now a CNCF Incubating project.

### Why OpenTelemetry?

Before OTel, every observability vendor (Datadog, Jaeger, Zipkin, New Relic) had its own SDK and agent. Switching vendors meant re-instrumenting your entire codebase.

OTel solves this with a **single, standardized instrumentation layer**:

```
Your App Code
     │
     ▼
[OTel SDK] ──────────────────────────────────────────────────
     │                                                       │
     ▼                                                       ▼
[OTel Collector]                               [Direct Export]
     │                                                       │
     ├─► Jaeger / Tempo (traces)            Jaeger / Zipkin
     ├─► Prometheus (metrics)               Prometheus
     └─► Loki / Elasticsearch (logs)        Datadog / New Relic
```

### OTel Status by Signal (as of 2024)

| Signal  | Specification | Go | Java | Python | JS/TS | .NET |
|---------|-------------|-----|------|--------|-------|------|
| Traces  | Stable ✅   | ✅  | ✅   | ✅     | ✅    | ✅   |
| Metrics | Stable ✅   | ✅  | ✅   | ✅     | ✅    | ✅   |
| Logs    | Stable ✅   | ✅  | ✅   | ✅     | ✅    | ✅   |

---

## 2. Core Concepts & Architecture

### The Signal Pipeline

Every OTel signal follows the same pipeline:

```
[Instrumentation] → [SDK] → [Processor] → [Exporter] → [Backend]
```

| Stage | Role |
|-------|------|
| **Instrumentation** | Library/manual code that creates telemetry |
| **SDK** | Collects data from instrumentation, applies sampling |
| **Processor** | Batch, filter, enrich telemetry before export |
| **Exporter** | Serializes and sends data to a backend |
| **Backend** | Stores and visualizes (Jaeger, Prometheus, etc.) |

### Key Components

#### TracerProvider
The entry point for tracing. Your app requests a `Tracer` from it:
```
TracerProvider → Tracer → Span
```

#### MeterProvider
The entry point for metrics. Your app requests a `Meter` from it:
```
MeterProvider → Meter → Instrument (Counter, Histogram, etc.)
```

#### LoggerProvider
The entry point for logs:
```
LoggerProvider → Logger → LogRecord
```

#### OTLP — OpenTelemetry Protocol
The native wire protocol for OTel. Supports gRPC and HTTP/protobuf.

```
Default ports:
  gRPC:         4317
  HTTP/proto:   4318
```

---

## 3. The Three Pillars: Traces, Metrics, Logs

### Traces

A **trace** represents the journey of a single request through your system. It is composed of **spans**.

```
Trace ID: abc123
│
├── [Span] HTTP GET /api/orders          0ms → 120ms  (root span)
│       service: api-gateway
│       http.status_code: 200
│
├────── [Span] OrderService.getOrder     5ms → 80ms
│           service: order-service
│           db.system: postgresql
│
└────────── [Span] SELECT * FROM orders  8ms → 40ms
                service: order-service
                db.statement: SELECT * FROM orders WHERE id=?
```

Key span attributes:
- **Trace ID** — unique ID for the entire request chain
- **Span ID** — unique ID for this specific operation
- **Parent Span ID** — links child spans to parents
- **Start/End timestamp**
- **Status** (OK, ERROR, UNSET)
- **Attributes** — key-value metadata
- **Events** — timestamped logs within a span
- **Links** — references to other traces (e.g., async messages)

### Metrics

OTel metrics align with the same instrument types as Prometheus but with more flexibility:

| Instrument | Like Prometheus | Use For |
|------------|----------------|---------|
| **Counter** | Counter | Requests, errors (monotonic) |
| **UpDownCounter** | Gauge | Queue size, active sessions |
| **Gauge** | Gauge | CPU, memory (non-additive) |
| **Histogram** | Histogram | Latency, request size |
| **ObservableCounter** | Counter (async) | CPU time (polled) |
| **ObservableUpDownCounter** | Gauge (async) | Memory (polled) |
| **ObservableGauge** | Gauge (async) | Temperature (polled) |

### Logs

OTel logs are designed to **correlate with traces** using Trace ID and Span ID, enabling you to jump from a log line directly to its trace.

A LogRecord contains:
- Timestamp
- Severity (TRACE, DEBUG, INFO, WARN, ERROR, FATAL)
- Body (the log message)
- Attributes (structured key-value pairs)
- Trace ID / Span ID (for correlation)
- Resource (which service emitted this)

---

## 4. OpenTelemetry Collector

The Collector is a standalone binary that **receives, processes, and exports** telemetry. It decouples your applications from your backends.

```
App A ──────────────────────────────────────────► Jaeger
App B ──► [OTel Collector] ──► processor ──────► Prometheus
App C ──────────────────────────────────────────► Loki
```

### Why Use the Collector?

- **Vendor agnostic** — change backends without touching app code
- **Data processing** — filter, transform, sample, enrich telemetry
- **Batching** — reduce load on backends
- **Retry & buffering** — handle backend unavailability
- **Security** — apps don't need credentials for backends

### Collector Distributions

| Distribution | Use Case |
|-------------|---------|
| **otelcol** | Core, minimal components |
| **otelcol-contrib** | All community components (recommended for getting started) |
| **otelcol-k8s** | Optimized for Kubernetes |
| Custom (via OCB) | Build your own with only needed components |

### Installing the Collector

```bash
# Docker
docker run --rm \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 8888:8888 \
  -v $(pwd)/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml \
  otel/opentelemetry-collector-contrib:latest

# Binary
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.97.0/otelcol-contrib_0.97.0_linux_amd64.tar.gz
tar xvf otelcol-contrib_*.tar.gz
./otelcol-contrib --config=config.yaml
```

### Collector Configuration (Full Example)

```yaml
# otel-collector-config.yml

# =============================================================
# RECEIVERS — How data comes IN
# =============================================================
receivers:
  # OTLP — from your apps (gRPC + HTTP)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "http://localhost:*"

  # Prometheus scraping
  prometheus:
    config:
      scrape_configs:
        - job_name: "otel-collector"
          scrape_interval: 15s
          static_configs:
            - targets: ["localhost:8888"]

  # Host metrics (CPU, memory, disk)
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:
      load:

  # Jaeger (receive from legacy Jaeger clients)
  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250
      thrift_http:
        endpoint: 0.0.0.0:14268

  # Zipkin
  zipkin:
    endpoint: 0.0.0.0:9411

  # Filelog (collect logs from files)
  filelog:
    include:
      - /var/log/myapp/*.log
    start_at: beginning
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.time
          layout: "%Y-%m-%dT%H:%M:%S.%fZ"

# =============================================================
# PROCESSORS — Transform/filter data
# =============================================================
processors:
  # Batch — group data before export (required for performance)
  batch:
    timeout: 1s
    send_batch_size: 1024
    send_batch_max_size: 2048

  # Memory limiter — prevent OOM
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # Add resource attributes to all telemetry
  resource:
    attributes:
      - key: deployment.environment
        value: "production"
        action: insert
      - key: service.namespace
        value: "mycompany"
        action: insert

  # Filter spans
  filter/drop_health:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/health"'
        - 'attributes["http.target"] == "/readyz"'

  # Attribute manipulation
  attributes/redact:
    actions:
      - key: db.statement
        action: hash   # Hash sensitive SQL
      - key: user.email
        action: delete
      - key: http.url
        pattern: "token=([^&]*)"
        action: update
        value: "token=REDACTED"

  # Transform (OTTL — OTel Transformation Language)
  transform/add_labels:
    trace_statements:
      - context: span
        statements:
          - set(attributes["custom.version"], "v2.1")
          - set(status.code, STATUS_CODE_ERROR) where attributes["http.status_code"] >= 500

  # Tail sampling (sample based on complete trace data)
  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 1000}
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

  # Kubernetes attributes (adds pod, namespace, node labels)
  k8sattributes:
    auth_type: "serviceAccount"
    passthrough: false
    extract:
      metadata:
        - k8s.pod.name
        - k8s.namespace.name
        - k8s.node.name
        - k8s.deployment.name
      labels:
        - tag_name: app
          key: app
          from: pod

# =============================================================
# EXPORTERS — Where data goes OUT
# =============================================================
exporters:
  # OTLP gRPC (to Jaeger, Tempo, etc.)
  otlp:
    endpoint: jaeger:4317
    tls:
      insecure: true

  # OTLP HTTP
  otlphttp:
    endpoint: https://api.honeycomb.io
    headers:
      x-honeycomb-team: "${HONEYCOMB_API_KEY}"

  # Prometheus (expose /metrics endpoint)
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otelcol"
    send_timestamps: true
    metric_expiration: 180m

  # Prometheus Remote Write
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

  # Loki (logs)
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false
      job: true
      instance: true
      level: true

  # Datadog
  datadog:
    api:
      site: datadoghq.com
      key: "${DD_API_KEY}"

  # Debug (print to stdout — use only for dev)
  debug:
    verbosity: detailed

  # File (for testing)
  file:
    path: /tmp/otel-output.json

# =============================================================
# EXTENSIONS — Health, performance probes
# =============================================================
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

# =============================================================
# PIPELINES — Connect receivers → processors → exporters
# =============================================================
service:
  extensions: [health_check, pprof, zpages]

  pipelines:
    traces:
      receivers:  [otlp, jaeger, zipkin]
      processors: [memory_limiter, filter/drop_health, k8sattributes, resource, batch]
      exporters:  [otlp, debug]

    metrics:
      receivers:  [otlp, prometheus, hostmetrics]
      processors: [memory_limiter, resource, batch]
      exporters:  [prometheus, prometheusremotewrite]

    logs:
      receivers:  [otlp, filelog]
      processors: [memory_limiter, resource, batch]
      exporters:  [loki, debug]

  telemetry:
    logs:
      level: info
    metrics:
      level: detailed
      address: 0.0.0.0:8888
```

### zPages (Built-in Debug UI)

When `zpages` extension is enabled, visit:
- `http://localhost:55679/debug/tracez` — trace sampling stats
- `http://localhost:55679/debug/pipelinez` — pipeline stats
- `http://localhost:55679/debug/extensionz` — extension status

---

## 5. Instrumentation — Auto vs Manual

### Automatic Instrumentation

Zero-code instrumentation via agents/agents that monkey-patch popular libraries.

**What gets auto-instrumented:**
- HTTP servers/clients
- gRPC
- Database drivers (pg, mysql, redis, mongo)
- Message queues (Kafka, RabbitMQ)
- Cloud SDK calls

**Tradeoff:** Less control, but zero code changes. Best for getting started.

### Manual Instrumentation

You write OTel API calls in your business logic. Gives full control over span names, attributes, and events.

**Use both:** Auto-instrumentation handles frameworks; manual adds business context.

---

## 6. Language SDKs

### Go

#### Installation

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

#### Full Setup + Tracing

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

// initTracer sets up the OTel TracerProvider
func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    // Exporter — sends to OTel Collector
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithInsecure(),
        otlptracegrpc.WithEndpoint("localhost:4317"),
    )
    if err != nil {
        return nil, err
    }

    // Resource — identifies this service
    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("1.2.0"),
            semconv.DeploymentEnvironment("production"),
        ),
    )

    // TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )

    // Set global TracerProvider and propagator
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp, nil
}

func main() {
    ctx := context.Background()

    tp, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        if err := tp.Shutdown(ctx); err != nil {
            log.Printf("Error shutting down tracer provider: %v", err)
        }
    }()

    tracer := otel.Tracer("order-service")

    // Auto-instrument HTTP handler
    http.Handle("/api/orders", otelhttp.NewHandler(
        http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, span := tracer.Start(r.Context(), "getOrders",
                // Span kind
                trace.WithSpanKind(trace.SpanKindServer),
            )
            defer span.End()

            // Add attributes
            span.SetAttributes(
                attribute.String("user.id", r.Header.Get("X-User-ID")),
                attribute.Int("page.size", 20),
            )

            orders, err := fetchOrders(ctx, tracer)
            if err != nil {
                span.RecordError(err)
                span.SetStatus(codes.Error, err.Error())
                http.Error(w, "Internal error", 500)
                return
            }

            // Add event (timestamped log within span)
            span.AddEvent("orders fetched",
                trace.WithAttributes(attribute.Int("count", len(orders))),
            )

            w.WriteHeader(http.StatusOK)
        }),
        "getOrders",
    ))

    http.ListenAndServe(":8080", nil)
}

// Demonstrates child span creation
func fetchOrders(ctx context.Context, tracer trace.Tracer) ([]Order, error) {
    ctx, span := tracer.Start(ctx, "fetchOrders.db")
    defer span.End()

    span.SetAttributes(
        semconv.DBSystemPostgreSQL,
        semconv.DBStatement("SELECT * FROM orders WHERE user_id = $1"),
    )

    // ... database call ...
    time.Sleep(50 * time.Millisecond)

    return []Order{}, nil
}
```

#### Metrics in Go

```go
import (
    "go.opentelemetry.io/otel/metric"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
)

func initMeter(ctx context.Context) (*sdkmetric.MeterProvider, error) {
    exporter, _ := otlpmetricgrpc.New(ctx,
        otlpmetricgrpc.WithInsecure(),
        otlpmetricgrpc.WithEndpoint("localhost:4317"),
    )

    mp := sdkmetric.NewMeterProvider(
        sdkmetric.WithReader(
            sdkmetric.NewPeriodicReader(exporter,
                sdkmetric.WithInterval(30*time.Second),
            ),
        ),
    )

    otel.SetMeterProvider(mp)
    return mp, nil
}

// Using meters
func setupMetrics() {
    meter := otel.Meter("order-service")

    // Counter
    requestCounter, _ := meter.Int64Counter(
        "http.server.requests",
        metric.WithDescription("Total HTTP requests"),
        metric.WithUnit("{request}"),
    )

    // Histogram
    requestDuration, _ := meter.Float64Histogram(
        "http.server.request.duration",
        metric.WithDescription("HTTP request duration"),
        metric.WithUnit("s"),
    )

    // Gauge (observable)
    meter.Int64ObservableGauge(
        "process.memory.bytes",
        metric.WithDescription("Current memory usage"),
        metric.WithInt64Callback(func(ctx context.Context, obs metric.Int64Observer) error {
            obs.Observe(getCurrentMemoryUsage())
            return nil
        }),
    )

    // Usage
    attrs := metric.WithAttributes(
        attribute.String("http.method", "GET"),
        attribute.Int("http.status_code", 200),
    )
    requestCounter.Add(ctx, 1, attrs)
    requestDuration.Record(ctx, 0.123, attrs)
}
```

---

### Python

#### Installation

```bash
# Core SDK
pip install opentelemetry-sdk
pip install opentelemetry-exporter-otlp

# Auto-instrumentation
pip install opentelemetry-instrumentation
pip install opentelemetry-instrumentation-fastapi
pip install opentelemetry-instrumentation-requests
pip install opentelemetry-instrumentation-sqlalchemy
pip install opentelemetry-instrumentation-redis

# Or install all auto-instrumentation packages
pip install opentelemetry-bootstrap
opentelemetry-bootstrap --action=install
```

#### Zero-Code (Auto-instrumentation)

```bash
# Wrap your app with the agent — no code changes needed
opentelemetry-instrument \
  --service_name=my-service \
  --exporter_otlp_endpoint=http://localhost:4317 \
  --traces_exporter=otlp \
  --metrics_exporter=otlp \
  --logs_exporter=otlp \
  python app.py
```

#### Manual Setup + FastAPI

```python
# tracing.py
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.b3 import B3MultiFormat


def setup_telemetry(service_name: str, service_version: str = "1.0.0"):
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": "production",
    })

    # --- Traces ---
    trace_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(BatchSpanProcessor(trace_exporter))
    trace.set_tracer_provider(tracer_provider)

    # --- Metrics ---
    metric_exporter = OTLPMetricExporter(endpoint="http://localhost:4317", insecure=True)
    metric_reader = PeriodicExportingMetricReader(metric_exporter, export_interval_millis=30000)
    meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
    metrics.set_meter_provider(meter_provider)

    # --- Propagators ---
    set_global_textmap(B3MultiFormat())

    return trace.get_tracer(service_name), metrics.get_meter(service_name)
```

```python
# main.py — FastAPI app with OTel
from fastapi import FastAPI, Request, HTTPException
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.trace import Status, StatusCode
import time

from tracing import setup_telemetry

tracer, meter = setup_telemetry("order-service")

# Define metrics
request_counter = meter.create_counter(
    name="http.server.requests",
    description="Total HTTP requests",
    unit="{request}",
)
request_duration = meter.create_histogram(
    name="http.server.request.duration",
    description="HTTP request duration",
    unit="s",
)
active_requests = meter.create_up_down_counter(
    name="http.server.active_requests",
    description="Active requests in-flight",
)

app = FastAPI()

# Auto-instrument FastAPI — adds traces for all routes
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()


@app.get("/api/orders/{order_id}")
async def get_order(order_id: str, request: Request):
    active_requests.add(1)
    start = time.time()
    attrs = {"http.method": "GET", "http.route": "/api/orders/{order_id}"}

    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", request.headers.get("X-User-ID", "anonymous"))

        try:
            order = await fetch_order_from_db(order_id)
            span.set_attribute("order.status", order["status"])
            request_counter.add(1, {**attrs, "http.status_code": 200})
            return order

        except OrderNotFoundError as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            request_counter.add(1, {**attrs, "http.status_code": 404})
            raise HTTPException(status_code=404, detail="Order not found")

        finally:
            duration = time.time() - start
            request_duration.record(duration, attrs)
            active_requests.add(-1)


async def fetch_order_from_db(order_id: str):
    with tracer.start_as_current_span("db.query") as span:
        span.set_attribute("db.system", "postgresql")
        span.set_attribute("db.statement", f"SELECT * FROM orders WHERE id = $1")
        span.set_attribute("db.parameters", order_id)

        # Simulate DB call
        await asyncio.sleep(0.03)

        span.add_event("query executed", {"rows_returned": 1})
        return {"id": order_id, "status": "shipped"}
```

---

### Java

#### Maven Dependencies

```xml
<!-- pom.xml -->
<properties>
  <otel.version>1.36.0</otel.version>
</properties>

<dependencies>
  <!-- OTel API -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>${otel.version}</version>
  </dependency>

  <!-- OTel SDK -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>${otel.version}</version>
  </dependency>

  <!-- OTLP Exporter -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <version>${otel.version}</version>
  </dependency>

  <!-- Semantic conventions -->
  <dependency>
    <groupId>io.opentelemetry.semconv</groupId>
    <artifactId>opentelemetry-semconv</artifactId>
    <version>1.23.1-alpha</version>
  </dependency>

  <!-- Spring Boot auto-instrumentation -->
  <dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.3.0-alpha</version>
  </dependency>
</dependencies>
```

#### Java Agent (Zero-Code)

```bash
# Download agent
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Run with agent
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
  -Dotel.traces.exporter=otlp \
  -Dotel.metrics.exporter=otlp \
  -Dotel.logs.exporter=otlp \
  -jar myapp.jar
```

#### Manual Instrumentation

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.*;
import io.opentelemetry.api.metrics.*;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.semconv.trace.attributes.SemanticAttributes;

@Service
public class OrderService {
    private static final Tracer tracer =
        GlobalOpenTelemetry.getTracer("order-service", "1.0.0");

    private static final Meter meter =
        GlobalOpenTelemetry.getMeter("order-service");

    private final LongCounter orderCounter = meter.counterBuilder("orders.processed")
        .setDescription("Total orders processed")
        .setUnit("{order}")
        .build();

    private final DoubleHistogram processingTime = meter.histogramBuilder("order.processing.duration")
        .setDescription("Order processing time")
        .setUnit("s")
        .build();

    public Order processOrder(String orderId, String userId) {
        Span span = tracer.spanBuilder("processOrder")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();

        long startNanos = System.nanoTime();

        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("order.id", orderId);
            span.setAttribute("user.id", userId);
            span.setAttribute(SemanticAttributes.DB_SYSTEM, "postgresql");

            Order order = fetchFromDatabase(orderId);

            span.addEvent("order fetched", Attributes.of(
                AttributeKey.stringKey("order.status"), order.getStatus()
            ));

            orderCounter.add(1, Attributes.of(
                AttributeKey.stringKey("order.type"), order.getType()
            ));

            return order;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            double duration = (System.nanoTime() - startNanos) / 1e9;
            processingTime.record(duration, Attributes.of(
                AttributeKey.stringKey("order.type"), "standard"
            ));
            span.end();
        }
    }
}
```

---

### Node.js / TypeScript

#### Installation

```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-grpc \
            @opentelemetry/exporter-metrics-otlp-grpc \
            @opentelemetry/resources \
            @opentelemetry/semantic-conventions
```

#### Setup (instrumentation.ts)

```typescript
// instrumentation.ts — must be loaded BEFORE your app
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'order-service',
    [SEMRESATTRS_SERVICE_VERSION]: '1.0.0',
    'deployment.environment': process.env.NODE_ENV || 'development',
  }),

  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4317',
  }),

  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: 'http://localhost:4317' }),
    exportIntervalMillis: 30_000,
  }),

  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
      '@opentelemetry/instrumentation-grpc': { enabled: true },
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown().then(() => process.exit(0));
});
```

```typescript
// app.ts
import './instrumentation';  // MUST be first import
import express from 'express';
import { trace, metrics, context, SpanStatusCode } from '@opentelemetry/api';
import { SEMATTRS_HTTP_METHOD, SEMATTRS_HTTP_STATUS_CODE } from '@opentelemetry/semantic-conventions';

const tracer = trace.getTracer('order-service', '1.0.0');
const meter = metrics.getMeter('order-service', '1.0.0');

// Metrics
const requestCounter = meter.createCounter('http.server.requests', {
  description: 'Total HTTP requests',
});
const requestDuration = meter.createHistogram('http.server.request.duration', {
  description: 'HTTP request duration',
  unit: 's',
});

const app = express();

app.get('/api/orders/:orderId', async (req, res) => {
  const start = Date.now();

  await tracer.startActiveSpan('getOrder', async (span) => {
    try {
      span.setAttribute('order.id', req.params.orderId);
      span.setAttribute('user.id', req.headers['x-user-id'] as string);

      const order = await fetchOrder(req.params.orderId);

      span.setStatus({ code: SpanStatusCode.OK });
      requestCounter.add(1, { [SEMATTRS_HTTP_METHOD]: 'GET', [SEMATTRS_HTTP_STATUS_CODE]: 200 });
      res.json(order);

    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      requestCounter.add(1, { [SEMATTRS_HTTP_METHOD]: 'GET', [SEMATTRS_HTTP_STATUS_CODE]: 500 });
      res.status(500).json({ error: 'Internal error' });

    } finally {
      requestDuration.record((Date.now() - start) / 1000, { [SEMATTRS_HTTP_METHOD]: 'GET' });
      span.end();
    }
  });
});

async function fetchOrder(orderId: string) {
  return tracer.startActiveSpan('db.findOrder', async (span) => {
    span.setAttribute('db.system', 'postgresql');
    span.setAttribute('db.statement', 'SELECT * FROM orders WHERE id = $1');

    try {
      // ... db query ...
      return { id: orderId, status: 'shipped' };
    } finally {
      span.end();
    }
  });
}

// Start with: node -r ./instrumentation.js app.js
app.listen(3000);
```

---

## 7. Context Propagation

Context propagation allows trace context to flow **across process boundaries** — from service to service, across HTTP, gRPC, Kafka, etc.

### How It Works

```
Service A                                   Service B
  │                                             │
  │ span = tracer.start("call-b")               │
  │                                             │
  │  inject(headers, ctx)  ──────────────────►  │  ctx = extract(headers)
  │  "traceparent: 00-abc-def-01"               │  span = tracer.start("handle", ctx)
  │                                             │
```

### W3C TraceContext (Standard)

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
             ver  trace-id (128-bit hex)         span-id (64-bit)  flags
```

### Propagation Formats

| Format | Header | Notes |
|--------|--------|-------|
| **W3C TraceContext** | `traceparent`, `tracestate` | Recommended standard |
| **B3 Single** | `b3` | Zipkin compatible |
| **B3 Multi** | `X-B3-TraceId`, `X-B3-SpanId`, etc. | Legacy Zipkin |
| **Jaeger** | `uber-trace-id` | Legacy Jaeger |
| **AWS X-Ray** | `X-Amzn-Trace-Id` | AWS services |

### Manual Propagation Examples

```python
# Python — inject into outgoing HTTP headers
from opentelemetry.propagate import inject, extract
import requests

def call_downstream():
    headers = {}
    inject(headers)  # Adds traceparent header
    response = requests.get("http://service-b/api", headers=headers)
    return response


# Python — extract from incoming headers
from opentelemetry import context
from opentelemetry.propagate import extract
from opentelemetry import trace

def handle_request(incoming_headers: dict):
    ctx = extract(incoming_headers)           # Restore parent context
    with tracer.start_as_current_span("handle", context=ctx) as span:
        # This span is now a child of the upstream span
        pass
```

```go
// Go — inject into outgoing HTTP request
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
client.Do(req)

// Go — extract from incoming request
ctx = otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))
ctx, span := tracer.Start(ctx, "handle-request")
defer span.End()
```

### Kafka Propagation

```python
# Producer — inject trace context into Kafka headers
from opentelemetry.propagate import inject

headers = {}
inject(headers)
producer.produce(
    topic="orders",
    value=message,
    headers=[(k, v.encode()) for k, v in headers.items()]
)

# Consumer — extract and link spans
from opentelemetry.propagate import extract
from opentelemetry.trace import Link

msg_headers = dict((k, v.decode()) for k, v in msg.headers())
ctx = extract(msg_headers)

links = [Link(context=trace.get_current_span(ctx).get_span_context())]
with tracer.start_as_current_span("process-order", links=links):
    # Linked (not child) because async — avoids trace bloat
    process(msg)
```

---

## 8. Exporters & Backends

### OTLP (Recommended)

Always prefer OTLP — it's the native OTel protocol and supported by all modern backends.

```python
# Python — OTLP gRPC
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)

# Python — OTLP HTTP
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
```

### Jaeger

```yaml
# Docker Compose
services:
  jaeger:
    image: jaegertracing/all-in-one:1.56
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

Point your exporter at `http://jaeger:4317`.

### Grafana Tempo

```yaml
# tempo.yml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
```

### Backend Comparison

| Backend | Traces | Metrics | Logs | Hosted Option |
|---------|--------|---------|------|---------------|
| Jaeger | ✅ | ❌ | ❌ | ❌ |
| Grafana Tempo | ✅ | ❌ | ❌ | ✅ (Grafana Cloud) |
| Prometheus | ❌ | ✅ | ❌ | ✅ |
| Grafana Loki | ❌ | ❌ | ✅ | ✅ (Grafana Cloud) |
| Grafana OSS Stack | ✅ | ✅ | ✅ | ✅ |
| Datadog | ✅ | ✅ | ✅ | ✅ |
| Honeycomb | ✅ | ✅ | ✅ | ✅ |
| New Relic | ✅ | ✅ | ✅ | ✅ |
| AWS X-Ray | ✅ | ❌ | ❌ | ✅ |
| Lightstep | ✅ | ✅ | ❌ | ✅ |
| SigNoz | ✅ | ✅ | ✅ | ✅ (self-hosted) |

---

## 9. Kubernetes & Helm Deployment

### Install OTel Operator

```bash
# Install cert-manager (required by operator)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Install OTel Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### Deploy Collector via Operator

```yaml
# collector.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  mode: daemonset    # daemonset | deployment | statefulset | sidecar

  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      kubeletstats:
        collection_interval: 30s
        auth_type: serviceAccount
        endpoint: "${K8S_NODE_NAME}:10250"
        insecure_skip_verify: true

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
      k8sattributes:
        auth_type: serviceAccount
        extract:
          metadata:
            - k8s.pod.name
            - k8s.namespace.name
            - k8s.node.name
      batch:

    exporters:
      otlp:
        endpoint: tempo:4317
        tls:
          insecure: true
      prometheusremotewrite:
        endpoint: http://prometheus:9090/api/v1/write

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp, kubeletstats]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [prometheusremotewrite]
```

### Auto-Instrumentation via Operator

```yaml
# auto-instrumentation.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otel-collector:4317

  propagators:
    - tracecontext
    - baggage
    - b3

  sampler:
    type: parentbased_traceidratio
    argument: "0.1"   # 10% sampling

  python:
    env:
      - name: OTEL_PYTHON_LOG_CORRELATION
        value: "true"

  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest

  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest

  go:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-go:latest
```

```yaml
# Annotate your deployment to enable auto-instrumentation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
        # or: inject-java, inject-nodejs, inject-go, inject-dotnet
    spec:
      containers:
        - name: order-service
          image: mycompany/order-service:latest
```

### Helm (kube-otel-stack)

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  --create-namespace \
  -f otel-values.yaml
```

```yaml
# otel-values.yaml
mode: daemonset

config:
  receivers:
    otlp:
      protocols:
        grpc: {}
        http: {}

  exporters:
    otlp:
      endpoint: "jaeger:4317"
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        exporters: [otlp]

presets:
  kubernetesAttributes:
    enabled: true
  kubeletMetrics:
    enabled: true
  logsCollection:
    enabled: true
    includeCollectorLogs: false
  hostMetrics:
    enabled: false
```

---

## 10. Sampling Strategies

Sampling controls **what percentage of traces to keep**. Every trace is captured, but only sampled traces are exported.

### Head Sampling (SDK-level)

Decision is made at the **start** of the trace — before any data is collected.

```python
from opentelemetry.sdk.trace.sampling import (
    ALWAYS_ON,
    ALWAYS_OFF,
    TraceIdRatioBased,
    ParentBased,
    DEFAULT_ON,
)

# Always sample (dev/testing)
sampler = ALWAYS_ON

# Never sample
sampler = ALWAYS_OFF

# Sample 10% of traces
sampler = TraceIdRatioBased(rate=0.1)

# Respect parent's sampling decision; sample 10% of root spans
sampler = ParentBased(root=TraceIdRatioBased(rate=0.1))

# Use in TracerProvider
tracer_provider = TracerProvider(sampler=sampler)
```

### Tail Sampling (Collector-level)

Decision is made **after** the full trace is collected. Allows intelligent sampling based on trace outcome (errors, latency).

```yaml
# In otel-collector-config.yml
processors:
  tail_sampling:
    decision_wait: 10s         # Wait for all spans before deciding
    num_traces: 50000          # Max traces held in memory
    expected_new_traces_per_sec: 1000

    policies:
      # Always keep errors
      - name: keep-errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Always keep slow traces (> 1s)
      - name: keep-slow
        type: latency
        latency:
          threshold_ms: 1000

      # Keep all traces from a specific user (debugging)
      - name: keep-debug-user
        type: string_attribute
        string_attribute:
          key: user.id
          values: ["debug-user-123"]

      # Sample 5% of everything else
      - name: default-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5

      # Composite: keep errors AND slow traces
      - name: composite-policy
        type: composite
        composite:
          max_total_spans_per_second: 1000
          policy_order: [keep-errors, keep-slow, default-policy]
          composite_sub_policy:
            - name: keep-errors
              type: status_code
              status_code:
                status_codes: [ERROR]
```

### Sampling Strategy Guide

| Scenario | Strategy |
|----------|---------|
| Development | `ALWAYS_ON` |
| Low-traffic production | `ParentBased(TraceIdRatioBased(0.5))` |
| High-traffic production | Tail sampling with error + latency policies |
| Compliance/debugging | `ALWAYS_ON` for specific services |
| Cost control | Tail sampling + probabilistic fallback |

---

## 11. Semantic Conventions

Semantic conventions define **standard attribute names** so all vendors and backends understand the same data.

### HTTP

```python
# Span attributes for HTTP servers
span.set_attribute("http.method", "GET")
span.set_attribute("http.url", "https://api.example.com/orders")
span.set_attribute("http.target", "/orders")
span.set_attribute("http.host", "api.example.com")
span.set_attribute("http.scheme", "https")
span.set_attribute("http.status_code", 200)
span.set_attribute("http.response_content_length", 1234)
span.set_attribute("http.user_agent", "Mozilla/5.0...")
span.set_attribute("net.peer.ip", "203.0.113.1")
```

### Database

```python
span.set_attribute("db.system", "postgresql")       # postgresql, mysql, redis, mongodb
span.set_attribute("db.name", "orders_db")
span.set_attribute("db.user", "appuser")
span.set_attribute("db.statement", "SELECT * FROM orders WHERE id = $1")
span.set_attribute("db.operation", "SELECT")
span.set_attribute("net.peer.name", "db.internal")
span.set_attribute("net.peer.port", 5432)
```

### Messaging (Kafka, RabbitMQ, SQS)

```python
span.set_attribute("messaging.system", "kafka")
span.set_attribute("messaging.destination", "orders-topic")
span.set_attribute("messaging.operation", "send")   # send | receive | process
span.set_attribute("messaging.message_id", "msg-123")
span.set_attribute("messaging.kafka.partition", 3)
```

### RPC / gRPC

```python
span.set_attribute("rpc.system", "grpc")
span.set_attribute("rpc.service", "OrderService")
span.set_attribute("rpc.method", "GetOrder")
span.set_attribute("rpc.grpc.status_code", 0)   # 0 = OK
```

### Cloud & Kubernetes

```python
# Resource attributes (not span attributes)
resource = Resource.create({
    "cloud.provider": "aws",
    "cloud.region": "us-east-1",
    "cloud.availability_zone": "us-east-1a",
    "k8s.cluster.name": "prod-cluster",
    "k8s.namespace.name": "default",
    "k8s.pod.name": "order-service-abc123",
    "k8s.container.name": "order-service",
    "k8s.deployment.name": "order-service",
})
```

### Using the Semconv Package

Always import from the semconv package rather than hardcoding strings:

```python
# Python
from opentelemetry.semconv.trace import SpanAttributes
span.set_attribute(SpanAttributes.HTTP_METHOD, "GET")
span.set_attribute(SpanAttributes.DB_SYSTEM, "postgresql")
```

```go
// Go
import semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
span.SetAttributes(semconv.HTTPMethod("GET"))
span.SetAttributes(semconv.DBSystemPostgreSQL)
```

---

## 12. Resource Attributes

Resources describe **the entity producing telemetry** — your service, its environment, and where it's running.

```yaml
# Standard resource attributes
service.name: "order-service"         # REQUIRED
service.version: "2.1.0"
service.namespace: "ecommerce"
service.instance.id: "pod-abc123"

# Deployment
deployment.environment: "production"  # production | staging | development

# Container
container.name: "order-service"
container.id: "abc123def456"
container.image.name: "mycompany/order-service"
container.image.tag: "2.1.0"

# Kubernetes
k8s.cluster.name: "prod-us-east"
k8s.namespace.name: "default"
k8s.pod.name: "order-service-abc"
k8s.node.name: "ip-10-0-1-100"
k8s.deployment.name: "order-service"

# Cloud
cloud.provider: "aws"               # aws | gcp | azure
cloud.region: "us-east-1"
cloud.account.id: "123456789012"
```

### Auto-detect Resources

```python
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.extension.aws.resource import AwsEcsResourceDetector
from opentelemetry.sdk.extension.aws.resource import AwsEc2ResourceDetector

resource = Resource.create().merge(
    Resource.create({
        "service.name": "order-service",
        "service.version": "2.1.0",
    })
)
```

```go
// Go — auto-detect resource
res, _ := resource.New(ctx,
    resource.WithFromEnv(),          // Read OTEL_RESOURCE_ATTRIBUTES env var
    resource.WithProcess(),          // PID, command line
    resource.WithOS(),               // OS type
    resource.WithContainer(),        // Container metadata
    resource.WithHost(),             // Hostname
    resource.WithAttributes(         // Override/add
        semconv.ServiceName("order-service"),
    ),
)
```

### Environment Variable Config

Set resource attributes without code changes:

```bash
export OTEL_SERVICE_NAME=order-service
export OTEL_SERVICE_VERSION=2.1.0
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=production,team=platform"
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1
```

---

## 13. Baggage

Baggage allows you to propagate **key-value pairs across the entire trace**, making values available to all downstream services.

```python
from opentelemetry import baggage
from opentelemetry.baggage.propagation import W3CBaggagePropagator

# Set baggage (in upstream service)
ctx = baggage.set_baggage("user.id", "user-123")
ctx = baggage.set_baggage("tenant.id", "tenant-456", context=ctx)

# Read baggage (in any downstream service)
user_id = baggage.get_baggage("user.id")
tenant_id = baggage.get_baggage("tenant.id")

# Add to span for visibility
span.set_attribute("user.id", user_id)
```

**Caution:** Baggage is propagated in HTTP headers and visible to all services. Never put secrets or PII in baggage.

---

## 14. Integrating with Prometheus & Grafana

### Full Observability Stack (Docker Compose)

```yaml
# docker-compose.yml — Full OTel + Prometheus + Grafana + Jaeger + Loki
version: "3.8"
services:

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
      - "8889:8889"    # Prometheus metrics
      - "8888:8888"    # Collector self-metrics
      - "55679:55679"  # zPages
    volumes:
      - ./otel-collector-config.yml:/etc/otelcol-contrib/config.yaml

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "14250:14250"   # gRPC

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Admin"
    volumes:
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yaml
```

```yaml
# grafana-datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true

  - name: Jaeger
    type: jaeger
    url: http://jaeger:16686

  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: '"trace_id":"(\w+)"'
          url: "$${__value.raw}"
          datasourceUid: jaeger
```

### Connecting Traces to Metrics in Grafana

```yaml
# prometheus.yml — scrape OTel Collector metrics
scrape_configs:
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]

  - job_name: "my-services"
    static_configs:
      - targets: ["order-service:8080"]
```

In Grafana, configure **Exemplars** to link from a Prometheus metric to a Jaeger trace:

```yaml
# Prometheus data source config in Grafana
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    jsonData:
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: jaeger
```

Then in your app, add exemplars to histograms:

```python
# Python — add exemplar to histogram observation
from opentelemetry import trace

span = trace.get_current_span()
ctx = span.get_span_context()

histogram.record(
    duration,
    attributes={"http.method": "GET"},
    # Exemplar automatically attached by SDK if trace is active
)
```

---

## 15. Production Best Practices

### 1. Use the Collector as a Buffer

Never export directly from app to backend in production. The Collector handles retries, batching, and buffering:

```yaml
exporters:
  otlp:
    endpoint: backend:4317
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 1000
```

### 2. Always Use the memory_limiter Processor

Prevents OOM crashes under load:

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512         # Hard limit
    spike_limit_mib: 128   # Soft limit buffer
```

Place `memory_limiter` **first** in all pipelines.

### 3. Batch Everything

```yaml
processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
    send_batch_max_size: 2048
```

Place `batch` **last** in all pipelines (after memory_limiter and other processors).

### 4. Processor Order Matters

```yaml
# Correct order
processors: [memory_limiter, filter, k8sattributes, resource, attributes, batch]
#            ^1st            ^early  ^enrich        ^enrich   ^transform  ^last
```

### 5. Cardinality Control

OTel metrics can explode in cardinality just like Prometheus. Use the `filter` processor to drop high-cardinality attributes:

```yaml
processors:
  attributes/drop_high_cardinality:
    actions:
      - key: http.url          # Full URL has query params — use http.target instead
        action: delete
      - key: user.id           # Never use user IDs as metric attributes
        action: delete
      - key: request.id        # Unique per request — infinite cardinality
        action: delete
```

### 6. Environment Variable Configuration

Configure exporters and sampling via env vars for 12-factor compliance:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
OTEL_SERVICE_NAME=order-service
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,team=platform
OTEL_PROPAGATORS=tracecontext,baggage
```

### 7. Span Naming Conventions

```python
# Good — specific and consistent
tracer.start_span("HTTP GET /api/orders/{id}")
tracer.start_span("PostgreSQL SELECT orders")
tracer.start_span("Kafka SEND orders-topic")

# Bad — too generic or too specific
tracer.start_span("request")          # Too vague
tracer.start_span("GET /orders/123")  # ID in name = high cardinality
```

### 8. Handle SDK Shutdown Gracefully

```python
import atexit
from opentelemetry.sdk.trace import TracerProvider

tp = TracerProvider(...)
trace.set_tracer_provider(tp)

# Ensure all spans are flushed on exit
atexit.register(tp.shutdown)
```

### 9. Sensitive Data Handling

```yaml
# Collector — hash or redact sensitive attributes
processors:
  attributes/redact:
    actions:
      - key: db.statement
        action: hash
      - key: user.email
        action: delete
      - key: http.request.header.authorization
        action: delete
```

### 10. Monitor the Collector Itself

```promql
# Collector dropped spans (should be 0)
rate(otelcol_processor_dropped_spans_total[5m])

# Queue size (approaching limit = backpressure)
otelcol_exporter_queue_size

# Export failures
rate(otelcol_exporter_send_failed_spans_total[5m])

# Memory usage
otelcol_process_memory_rss
```

---

## 16. Troubleshooting

### Enable Debug Exporter

```yaml
exporters:
  debug:
    verbosity: detailed   # normal | detailed

service:
  pipelines:
    traces:
      exporters: [otlp, debug]   # Add debug alongside real exporter
```

### Verify Data Reaches Collector

```bash
# Check collector health
curl http://localhost:13133/

# Check zPages
open http://localhost:55679/debug/tracez

# Test OTLP endpoint directly
grpcurl -plaintext localhost:4317 list
```

### Check SDK is Sending Data

```python
# Python — use ConsoleSpanExporter to print to stdout
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

tp = TracerProvider()
tp.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
```

```go
// Go — stdout exporter
import "go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
```

### Common Issues

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| No spans in Jaeger | Exporter misconfigured | Check `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Spans not connected | Missing propagation | Ensure `inject/extract` around HTTP calls |
| High memory in Collector | No `memory_limiter` | Add as first processor |
| Metrics missing labels | High-cardinality filter | Check filter processor config |
| Spans dropping | Collector queue full | Increase `queue_size` or scale collector |
| Sampling too aggressive | Wrong sampler config | Check `OTEL_TRACES_SAMPLER_ARG` |
| Context not propagating | Wrong propagator | Ensure sender and receiver use same format |

### Validate Collector Config

```bash
# Validate config file
otelcol-contrib validate --config=config.yaml

# Check running config
curl http://localhost:55679/debug/configz
```

### Environment Variable Debugging

```bash
# List all OTel env vars currently set
env | grep OTEL_

# Dump OTel SDK config (Go)
OTEL_LOG_LEVEL=debug ./myapp

# Python verbose logging
OTEL_PYTHON_LOG_LEVEL=debug python app.py
```

---

## Quick Reference Card

```bash
# OTLP ports
gRPC:   4317
HTTP:   4318

# Collector internal
Health:  13133
Metrics: 8888
zPages:  55679
pprof:   1777

# Key env vars
OTEL_SERVICE_NAME
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_EXPORTER_OTLP_PROTOCOL     # grpc | http/protobuf | http/json
OTEL_TRACES_SAMPLER             # always_on | always_off | traceidratio | parentbased_*
OTEL_TRACES_SAMPLER_ARG         # 0.0 - 1.0
OTEL_PROPAGATORS                # tracecontext,baggage,b3,b3multi,jaeger,xray
OTEL_RESOURCE_ATTRIBUTES        # key=val,key2=val2
OTEL_LOG_LEVEL                  # debug | info | warn | error

# Validate collector config
otelcol-contrib validate --config=config.yaml

# Run collector (Docker)
docker run -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/config.yaml:/etc/otelcol-contrib/config.yaml \
  otel/opentelemetry-collector-contrib

# Test OTLP endpoint
grpcurl -plaintext localhost:4317 list
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
```

---

*This tutorial covers OpenTelemetry SDK ~v1.x and Collector ~v0.97+. Refer to the [official documentation](https://opentelemetry.io/docs/) for the latest updates.*
