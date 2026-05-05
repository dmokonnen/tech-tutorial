# Comprehensive Prometheus Tutorial

---

## Table of Contents

1. [What is Prometheus?](#1-what-is-prometheus)
2. [Core Architecture](#2-core-architecture)
3. [Installation & Setup](#3-installation--setup)
4. [Configuration Deep-Dive](#4-configuration-deep-dive)
5. [Data Model & Metric Types](#5-data-model--metric-types)
6. [PromQL — Prometheus Query Language](#6-promql--prometheus-query-language)
7. [Exporters](#7-exporters)
8. [Service Discovery](#8-service-discovery)
9. [Alerting with Alertmanager](#9-alerting-with-alertmanager)
10. [Instrumentation (Client Libraries)](#10-instrumentation-client-libraries)
11. [Storage & Retention](#11-storage--retention)
12. [Federation & Remote Storage](#12-federation--remote-storage)
13. [Security](#13-security)
14. [Grafana Integration](#14-grafana-integration)
15. [Production Best Practices](#15-production-best-practices)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. What is Prometheus?

Prometheus is an open-source **systems monitoring and alerting toolkit** originally built at SoundCloud in 2012 and donated to the CNCF in 2016. It is now the de-facto standard for monitoring cloud-native and Kubernetes-based workloads.

### Key Characteristics

- **Pull-based model** — Prometheus scrapes metrics from targets over HTTP (unlike push-based systems like Graphite/InfluxDB)
- **Dimensional data model** — every metric is identified by a name and a set of key-value labels
- **Powerful query language (PromQL)** — for slicing, aggregating, and alerting on time-series data
- **No external dependencies** — single Go binary; no distributed storage required for a single node
- **Reliable** — designed so you can still get accurate readings even when other parts of the system are down

### When to Use Prometheus

Prometheus excels at recording **numeric time-series** data — CPU, memory, request rates, error rates, latencies. It is **not** designed for:

- Log aggregation (use Loki, ELK)
- Event streaming (use Kafka)
- 100% accuracy billing data (it uses sampling)

---

## 2. Core Architecture

```
+-------------------+        scrape         +-------------------+
|   Your Services   |  <------------------  |    Prometheus     |
|   (Instrumented   |                        |    Server         |
|    with /metrics) |                        |                   |
+-------------------+                        |  - TSDB (storage) |
                                             |  - Rule engine    |
+-------------------+        scrape          |  - HTTP API       |
|   Exporters       |  <------------------   +--------+----------+
| (node, blackbox,  |                                 |
|  postgres, etc.)  |                                 | fire alerts
+-------------------+                                 v
                                             +-------------------+
+-------------------+   service discovery   |   Alertmanager    |
|  Consul / K8s /   |  ------------------>  |                   |
|  EC2 / DNS        |                        | - dedup, group    |
+-------------------+                        | - route, silence  |
                                             | - notify          |
                                             +-------------------+
                                                      |
                                             PagerDuty / Slack / Email
```

### Components

| Component | Role |
|-----------|------|
| **Prometheus Server** | Scrapes, stores, and evaluates rules |
| **Alertmanager** | Handles alert routing, deduplication, silences |
| **Pushgateway** | Accepts metrics pushed from short-lived jobs |
| **Exporters** | Bridge between Prometheus and third-party systems |
| **Client Libraries** | Instrument application code directly |

---

## 3. Installation & Setup

### Option A — Binary (Linux)

```bash
# Download latest release
PROM_VERSION="2.51.2"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz

tar xvf prometheus-*.tar.gz
cd prometheus-*/

# Run with default config
./prometheus --config.file=prometheus.yml
```

### Option B — Docker

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### Option C — Docker Compose (Recommended for local dev)

```yaml
# docker-compose.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:v2.51.2
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.console.libraries=/usr/share/prometheus/console_libraries"
      - "--web.console.templates=/usr/share/prometheus/consoles"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.7.0
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus_data:
```

### Option D — Kubernetes (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=my-secret-password
```

### Verify Installation

After starting, open `http://localhost:9090`. You should see the Prometheus UI. Navigate to **Status → Targets** to confirm Prometheus is scraping itself.

---

## 4. Configuration Deep-Dive

The main config file is `prometheus.yml`. Here is a fully annotated example:

```yaml
# prometheus.yml

# Global settings — apply to all scrape jobs unless overridden
global:
  scrape_interval: 15s        # How often to scrape targets
  scrape_timeout: 10s         # Timeout per scrape request
  evaluation_interval: 15s    # How often to evaluate alerting rules

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"

# Rule files — alerts and recording rules
rule_files:
  - "rules/alerts.yml"
  - "rules/recording_rules.yml"

# Scrape configurations
scrape_configs:

  # Prometheus scrapes itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Node Exporter (host metrics)
  - job_name: "node"
    static_configs:
      - targets:
          - "node-exporter:9100"
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Your application
  - job_name: "myapp"
    metrics_path: "/metrics"   # default; can be customized
    scheme: "http"
    static_configs:
      - targets:
          - "myapp:8080"
        labels:
          env: "production"
          team: "platform"

  # Basic auth example
  - job_name: "secure-service"
    basic_auth:
      username: "prometheus"
      password_file: "/etc/prometheus/secrets/password"
    static_configs:
      - targets: ["secure-service:8080"]

  # TLS example
  - job_name: "tls-service"
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/certs/ca.pem
      cert_file: /etc/prometheus/certs/cert.pem
      key_file: /etc/prometheus/certs/key.pem
    static_configs:
      - targets: ["tls-service:8443"]
```

### Reload Config Without Restart

```bash
# Send SIGHUP
kill -HUP $(pgrep prometheus)

# Or via HTTP API (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload
```

---

## 5. Data Model & Metric Types

### Labels

Every time-series is uniquely identified by its **metric name** and a set of **labels** (key=value pairs):

```
http_requests_total{method="GET", status="200", handler="/api/v1/users"}
```

Labels are powerful but have a cost — each unique label combination is a separate time-series. This is called **cardinality**. Avoid high-cardinality labels like user IDs, request IDs, or IP addresses.

### The Four Metric Types

#### 1. Counter

A **monotonically increasing** value that only goes up (or resets to 0 on restart). Use for: requests, errors, bytes sent.

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="500"} 5
```

Always use `rate()` or `increase()` with counters in PromQL — never graph raw counters.

#### 2. Gauge

A value that **can go up or down**. Use for: memory usage, active connections, temperature, queue size.

```
# HELP memory_usage_bytes Current memory usage
# TYPE memory_usage_bytes gauge
memory_usage_bytes{instance="server-1"} 536870912
```

#### 3. Histogram

Samples observations and counts them in **configurable buckets**. Automatically creates `_bucket`, `_sum`, and `_count` series. Use for: request durations, response sizes.

```
# HELP http_request_duration_seconds Request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 120
http_request_duration_seconds_bucket{le="0.01"}  200
http_request_duration_seconds_bucket{le="0.025"} 280
http_request_duration_seconds_bucket{le="0.05"}  300
http_request_duration_seconds_bucket{le="+Inf"}  310
http_request_duration_seconds_sum   15.7
http_request_duration_seconds_count 310
```

Use `histogram_quantile()` to calculate percentiles (p50, p95, p99).

#### 4. Summary

Similar to Histogram but **pre-calculates quantiles on the client side**. Less flexible than histograms — you can't aggregate across instances. Prefer histograms in most cases.

```
# HELP rpc_duration_seconds RPC duration
# TYPE rpc_duration_seconds summary
rpc_duration_seconds{quantile="0.5"}  0.012
rpc_duration_seconds{quantile="0.9"}  0.018
rpc_duration_seconds{quantile="0.99"} 0.045
rpc_duration_seconds_sum   1800.3
rpc_duration_seconds_count 100000
```

### Metric Naming Conventions

```
<namespace>_<subsystem>_<name>_<unit>

# Good examples:
http_requests_total           # counter — _total suffix for counters
http_request_duration_seconds # histogram — always use base units (seconds, bytes)
process_memory_bytes          # gauge — never use _mb, _kb, etc.
node_cpu_seconds_total        # counter with unit
```

---

## 6. PromQL — Prometheus Query Language

PromQL is a functional query language for selecting and aggregating time-series data.

### Selectors

```promql
# Exact match
http_requests_total{job="myapp"}

# Regex match
http_requests_total{method=~"GET|POST"}

# Negative match
http_requests_total{status!="200"}

# Negative regex
http_requests_total{status!~"2.."}

# Range vector — last 5 minutes of data
http_requests_total[5m]
```

### Operators

```promql
# Arithmetic
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Comparison
http_requests_total > 1000

# Logical/set operators (on vector matching)
up == 0                          # targets that are down
```

### Functions — The Most Important Ones

#### rate() and irate()

```promql
# Per-second rate of requests over last 5 minutes (smoothed)
rate(http_requests_total[5m])

# Per-second rate using only last two data points (more responsive)
irate(http_requests_total[5m])

# Total increase over time window
increase(http_requests_total[1h])
```

Use `rate()` for alerts and dashboards. Use `irate()` only for volatile, fast-moving metrics.

#### Aggregation Operators

```promql
# Sum across all instances
sum(rate(http_requests_total[5m]))

# Sum, grouped by status code
sum by (status) (rate(http_requests_total[5m]))

# Sum, removing the instance label
sum without (instance) (rate(http_requests_total[5m]))

# Average
avg(node_cpu_seconds_total)

# Max across all pods
max by (namespace) (container_memory_usage_bytes)

# Count targets
count(up == 1)

# Top 5 by value
topk(5, rate(http_requests_total[5m]))
```

#### histogram_quantile()

```promql
# 95th percentile request duration
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# Per-service p99
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

#### Other Useful Functions

```promql
# Detect value change (useful for version tracking)
changes(process_start_time_seconds[1h])

# Derivative (rate of change for gauges)
deriv(disk_usage_bytes[10m])

# Predict value in 4 hours using linear regression
predict_linear(disk_usage_bytes[1h], 4 * 3600)

# Offset — compare current to 1 week ago
rate(http_requests_total[5m]) 
  / rate(http_requests_total[5m] offset 1w)

# Clamp values
clamp_min(some_metric, 0)
clamp_max(some_metric, 100)

# Days/hours since Unix timestamp
(time() - process_start_time_seconds) / 3600  # uptime in hours
```

### Multi-Vector Operations (Matching)

```promql
# Divide two metrics with matching labels
rate(http_request_errors_total[5m])
  / rate(http_requests_total[5m])

# Match on specific labels (when cardinality differs)
sum by (job) (rate(http_request_errors_total[5m]))
  / on(job) sum by (job) (rate(http_requests_total[5m]))

# Many-to-one matching
rate(http_requests_total[5m])
  * on(instance) group_left(version)
    app_info
```

### Recording Rules

Pre-compute expensive queries to speed up dashboards and reduce load:

```yaml
# rules/recording_rules.yml
groups:
  - name: http_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_request_duration_p99:rate5m
        expr: |
          histogram_quantile(0.99,
            sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

Use the naming convention `level:metric:operations` for recorded metrics.

---

## 7. Exporters

Exporters translate third-party systems into Prometheus metrics.

### Node Exporter (Host Metrics)

```bash
docker run -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  prom/node-exporter \
  --path.rootfs=/host
```

Key metrics:
```promql
# CPU usage %
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory available %
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Disk usage %
100 - (node_filesystem_avail_bytes{mountpoint="/"} 
       / node_filesystem_size_bytes{mountpoint="/"} * 100)

# Network traffic
rate(node_network_receive_bytes_total{device="eth0"}[5m])
```

### Commonly Used Exporters

| Exporter | Default Port | What It Monitors |
|----------|-------------|-----------------|
| node_exporter | 9100 | Linux host metrics |
| blackbox_exporter | 9115 | HTTP, TCP, DNS, ICMP probes |
| postgres_exporter | 9187 | PostgreSQL |
| mysql_exporter | 9104 | MySQL/MariaDB |
| redis_exporter | 9121 | Redis |
| mongodb_exporter | 9216 | MongoDB |
| elasticsearch_exporter | 9114 | Elasticsearch |
| kafka_exporter | 9308 | Apache Kafka |
| nginx-vts-exporter | 9913 | NGINX |
| haproxy_exporter | 9101 | HAProxy |
| snmp_exporter | 9116 | SNMP devices |
| jmx_exporter | 9999 | JVM/Java apps |

### Blackbox Exporter (Endpoint Probing)

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      follow_redirects: true
      tls_config:
        insecure_skip_verify: false

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
```

```yaml
# prometheus.yml — scrape via blackbox
- job_name: "blackbox-http"
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://example.com
        - https://api.myapp.com/health
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

Useful blackbox queries:
```promql
# SSL certificate expiry (days)
(probe_ssl_earliest_cert_expiry - time()) / 86400

# Target up/down
probe_success == 0

# HTTP response time
probe_http_duration_seconds{phase="processing"}
```

---

## 8. Service Discovery

Prometheus supports dynamic target discovery, eliminating the need for manual `static_configs`.

### Kubernetes Service Discovery

```yaml
scrape_configs:
  # Scrape all pods with prometheus.io/scrape annotation
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"

      # Use custom port annotation if present
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (\S+)
        replacement: $1

      # Use custom path annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Add namespace label
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      # Add pod name label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

  # Scrape Kubernetes API server
  - job_name: "kubernetes-apiservers"
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
```

### Consul Service Discovery

```yaml
- job_name: "consul-services"
  consul_sd_configs:
    - server: "consul:8500"
      services: []    # empty = all services
  relabel_configs:
    - source_labels: [__meta_consul_service]
      target_label: job
    - source_labels: [__meta_consul_node]
      target_label: instance
    - source_labels: [__meta_consul_tags]
      regex: .*,prometheus,.*
      action: keep
```

### EC2 Service Discovery

```yaml
- job_name: "ec2"
  ec2_sd_configs:
    - region: us-east-1
      access_key: AKIAIOSFODNN7EXAMPLE
      secret_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
      port: 9100
  relabel_configs:
    - source_labels: [__meta_ec2_tag_Name]
      target_label: instance
    - source_labels: [__meta_ec2_tag_Environment]
      target_label: env
```

### Relabeling Cheat Sheet

```yaml
relabel_configs:
  # Keep only matching targets
  - source_labels: [__meta_kubernetes_namespace]
    action: keep
    regex: production

  # Drop matching targets
  - source_labels: [__meta_kubernetes_pod_name]
    action: drop
    regex: test-.*

  # Rename/copy label
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app

  # Replace label value
  - source_labels: [__address__]
    regex: "([^:]+):.*"
    replacement: "${1}:9100"
    target_label: __address__

  # Drop a label
  - action: labeldrop
    regex: __meta_.*

  # Keep only specific labels
  - action: labelkeep
    regex: (job|instance|env)
```

---

## 9. Alerting with Alertmanager

### Step 1 — Define Alerting Rules

```yaml
# rules/alerts.yml
groups:
  - name: availability
    rules:
      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} (job {{ $labels.job }}) has been unreachable for >2 minutes."

      # High CPU
      - alert: HighCPUUsage
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is {{ printf \"%.1f\" $value }}% (threshold: 85%)"

      # High memory
      - alert: HighMemoryUsage
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }}"
          description: "Memory usage is {{ printf \"%.1f\" $value }}%"

      # Disk filling up
      - alert: DiskSpaceLow
        expr: |
          predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24 * 3600) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Disk will fill within 24h on {{ $labels.instance }}"

  - name: application
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in {{ $labels.job }}"
          description: "Error rate is {{ printf \"%.1f\" (mul $value 100) }}%"

      # Slow response times
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95,
            sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow p95 latency in {{ $labels.job }}"
          description: "p95 latency is {{ printf \"%.3f\" $value }}s"
```

### Step 2 — Configure Alertmanager

```yaml
# alertmanager.yml
global:
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alerts@mycompany.com"
  smtp_auth_username: "alerts@mycompany.com"
  smtp_auth_password: "app-password-here"
  slack_api_url: "https://hooks.slack.com/services/T.../B.../xxx"

# Templates
templates:
  - "/etc/alertmanager/templates/*.tmpl"

# Routing tree
route:
  # Default receiver for unmatched alerts
  receiver: "slack-general"

  # Route critical alerts to PagerDuty
  routes:
    - match:
        severity: critical
      receiver: "pagerduty-critical"
      continue: false

    - match:
        severity: warning
      receiver: "slack-warnings"
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

  group_by: ["alertname", "cluster", "job"]
  group_wait: 30s          # Wait to batch alerts
  group_interval: 5m       # Wait between alert groups
  repeat_interval: 12h     # How often to re-notify

# Receivers
receivers:
  - name: "slack-general"
    slack_configs:
      - channel: "#alerts"
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Labels:* {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}

  - name: "slack-warnings"
    slack_configs:
      - channel: "#alerts-warning"
        send_resolved: true

  - name: "pagerduty-critical"
    pagerduty_configs:
      - routing_key: "your-pagerduty-integration-key"
        description: '{{ template "pagerduty.default.description" . }}'

  - name: "email-oncall"
    email_configs:
      - to: "oncall@mycompany.com"
        send_resolved: true

# Inhibition rules — suppress lower severity if higher severity fires
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["alertname", "instance"]
```

### Silences & Maintenance

```bash
# Create a silence via API
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "instance", "value": "server-1", "isRegex": false}
    ],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt":   "2024-01-01T02:00:00Z",
    "comment":  "Maintenance window",
    "createdBy": "admin"
  }'
```

---

## 10. Instrumentation (Client Libraries)

### Go

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )

    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method"},
    )

    activeConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "active_connections",
        Help: "Current active connections",
    })
)

func handler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(requestDuration.WithLabelValues(r.Method))
    defer timer.ObserveDuration()

    activeConnections.Inc()
    defer activeConnections.Dec()

    // ... handle request ...

    requestsTotal.WithLabelValues(r.Method, "200").Inc()
}

func main() {
    http.HandleFunc("/api", handler)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### Python

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

ACTIVE_REQUESTS = Gauge(
    'active_requests',
    'Current active requests'
)

# Use as decorator
def track_requests(method, endpoint):
    def decorator(func):
        def wrapper(*args, **kwargs):
            ACTIVE_REQUESTS.inc()
            start = time.time()
            status = "200"
            try:
                result = func(*args, **kwargs)
                return result
            except Exception as e:
                status = "500"
                raise
            finally:
                REQUEST_COUNT.labels(method, endpoint, status).inc()
                REQUEST_LATENCY.labels(method, endpoint).observe(time.time() - start)
                ACTIVE_REQUESTS.dec()
        return wrapper
    return decorator

@track_requests("GET", "/api/users")
def get_users():
    time.sleep(0.05)
    return {"users": []}

if __name__ == "__main__":
    start_http_server(8000)  # Exposes /metrics on port 8000
    while True:
        get_users()
        time.sleep(1)
```

### Node.js

```javascript
const express = require('express');
const client = require('prom-client');

const app = express();
const register = new client.Registry();

// Default metrics (process, event loop, etc.)
client.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register]
});

// Middleware
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({ method: req.method, route: req.path });
  res.on('finish', () => {
    httpRequestsTotal.inc({ method: req.method, route: req.path, status: res.statusCode });
    end();
  });
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

app.listen(3000);
```

### Java (Micrometer + Spring Boot)

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

```java
@Service
public class OrderService {
    private final Counter ordersTotal;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry registry) {
        this.ordersTotal = Counter.builder("orders_total")
            .description("Total orders processed")
            .tag("type", "standard")
            .register(registry);

        this.orderProcessingTime = Timer.builder("order_processing_seconds")
            .description("Time to process an order")
            .register(registry);
    }

    public Order processOrder(Order order) {
        return orderProcessingTime.record(() -> {
            // ... processing logic ...
            ordersTotal.increment();
            return order;
        });
    }
}
```

---

## 11. Storage & Retention

Prometheus uses its own TSDB (Time Series Database) optimized for time-series data.

### Storage Layout

```
/prometheus/
  01BKGV7JC0RY8A6AVTR00NQSK/     # Block (2 hours of data)
    chunks/
      000001                       # Chunk file (raw samples)
    index                          # Block index
    meta.json                      # Block metadata
    tombstones                     # Deleted data markers
  wal/                             # Write-ahead log (in-memory buffer)
    00000001
    00000002
```

### Configuration Flags

```bash
prometheus \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=15d \     # Keep data for 15 days
  --storage.tsdb.retention.size=50GB \   # OR limit by size
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.max-block-duration=36h
```

### Compaction

Prometheus automatically compacts 2-hour blocks into larger blocks over time:
- Raw blocks: 2h
- After 1 day: ~6h blocks
- After 1 week: ~36h blocks

### Estimating Storage

A rough estimate: **~1-2 bytes per sample per second**

```
storage_bytes ≈ retention_seconds × ingested_samples_per_second × 2
```

Example: 100,000 active series, 15s scrape interval, 30-day retention:
```
samples/sec = 100,000 / 15 = 6,667
storage = 30d × 86400s × 6,667 × 2 bytes ≈ ~34 GB
```

### Snapshot & Backup

```bash
# Create a snapshot (admin API must be enabled)
curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Snapshots appear in:
ls /prometheus/snapshots/
```

---

## 12. Federation & Remote Storage

### Federation

Federate metrics from multiple Prometheus instances into a single "global" Prometheus:

```yaml
# global-prometheus.yml
scrape_configs:
  - job_name: "federate"
    honor_labels: true      # Preserve original labels
    metrics_path: "/federate"
    params:
      match[]:
        - '{job="critical-service"}'
        - '{__name__=~"job:.*"}'    # Only pre-aggregated recording rules
    static_configs:
      - targets:
          - "prometheus-dc1:9090"
          - "prometheus-dc2:9090"
```

### Remote Write / Remote Read

For long-term storage, send data to Thanos, Cortex, VictoriaMetrics, or other backends:

```yaml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      max_samples_per_send: 10000
      capacity: 100000
      max_backoff: 30s

  # With authentication
  - url: "https://cortex:9090/api/prom/push"
    basic_auth:
      username: tenant-id
      password: secret

remote_read:
  - url: "http://thanos-query:10902/api/v1/read"
    read_recent: false    # Only use remote for historical data
```

### Thanos (Overview)

Thanos extends Prometheus with unlimited retention and global querying:

- **Thanos Sidecar** — runs alongside Prometheus, uploads blocks to object storage (S3/GCS)
- **Thanos Store** — serves historical data from object storage
- **Thanos Query** — deduplicates and queries across multiple Prometheus/Store instances
- **Thanos Compactor** — compacts and downsamples historical blocks

---

## 13. Security

### TLS for the Web UI / API

```bash
prometheus \
  --web.config.file=/etc/prometheus/web.yml
```

```yaml
# web.yml
tls_server_config:
  cert_file: /etc/prometheus/certs/server.crt
  key_file:  /etc/prometheus/certs/server.key

basic_auth_users:
  admin: $2y$10$...  # bcrypt hash
```

Generate bcrypt hash:
```bash
htpasswd -nBC 10 admin
```

### Network-Level Security

- Run Prometheus behind a reverse proxy (Nginx, Traefik) with authentication
- Restrict the Prometheus port at the firewall level — it should never be publicly accessible
- Use Kubernetes NetworkPolicies to restrict scrape access
- Never expose the Pushgateway publicly

### RBAC in Kubernetes

```yaml
# ClusterRole for Prometheus to discover targets
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources: [nodes, nodes/metrics, services, endpoints, pods]
    verbs: [get, list, watch]
  - nonResourceURLs: [/metrics, /metrics/cadvisor]
    verbs: [get]
```

---

## 14. Grafana Integration

### Connect Prometheus as a Data Source

1. Open Grafana → **Configuration → Data Sources → Add data source**
2. Select **Prometheus**
3. Set URL: `http://prometheus:9090`
4. Click **Save & Test**

### Useful Dashboard Variables

```
# Variable: instance
Label values query: label_values(up{job="$job"}, instance)

# Variable: job
Label values query: label_values(up, job)

# Variable: interval
Custom: 1m,5m,15m,30m,1h
```

### Essential Grafana Panels

```promql
# CPU usage (Gauge or Time series)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance="$instance"}[5m])) * 100)

# Memory usage %
(1 - node_memory_MemAvailable_bytes{instance="$instance"} / node_memory_MemTotal_bytes{instance="$instance"}) * 100

# Request rate
sum(rate(http_requests_total{job="$job"}[$__rate_interval]))

# Error rate %
sum(rate(http_requests_total{job="$job", status=~"5.."}[$__rate_interval]))
/ sum(rate(http_requests_total{job="$job"}[$__rate_interval])) * 100

# p95 latency
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="$job"}[$__rate_interval])))
```

### Import Pre-built Dashboards

Popular dashboard IDs from grafana.com:

| Dashboard | ID |
|-----------|-----|
| Node Exporter Full | 1860 |
| Kubernetes Cluster | 7249 |
| Kubernetes Pods | 6417 |
| PostgreSQL | 9628 |
| Redis | 11835 |
| NGINX | 9614 |

Go to **Dashboards → Import** and enter the ID.

---

## 15. Production Best Practices

### Capacity & Cardinality

```promql
# Check current series count
prometheus_tsdb_head_series

# Check ingestion rate (samples/sec)
rate(prometheus_tsdb_head_samples_appended_total[5m])

# Find highest cardinality metrics
topk(10, count by (__name__) ({__name__=~".+"}))
```

**Cardinality rules:**
- Keep label cardinality below 10,000 unique values per label
- Never use user IDs, UUIDs, or IP addresses as labels
- Use histograms instead of per-request timing metrics

### High Availability

Run two identical Prometheus instances scraping the same targets. Use Alertmanager clustering for alert deduplication:

```bash
# Alertmanager cluster setup
alertmanager \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-2:9094 \
  --cluster.peer=alertmanager-3:9094
```

### Resource Recommendations

| Scale | CPU | Memory | Disk |
|-------|-----|--------|------|
| < 100k series | 1-2 cores | 4-8 GB | 50 GB |
| 100k-500k series | 4-8 cores | 16-32 GB | 200 GB |
| 500k-1M series | 8-16 cores | 32-64 GB | 500 GB |
| > 1M series | Consider Thanos/Cortex | | |

### Scrape Interval Guidance

| Use Case | Interval |
|----------|---------|
| Critical production | 10–15s |
| Standard services | 15–30s |
| Batch jobs | 60s |
| Long-term trends | 60s+ |

### The Four Golden Signals

Design dashboards and alerts around these:

1. **Latency** — time to service a request (`histogram_quantile`)
2. **Traffic** — requests per second (`rate`)
3. **Errors** — rate of failed requests
4. **Saturation** — how "full" the service is (CPU, memory, queue depth)

### Alert Fatigue Prevention

- Use `for:` duration on all alerts (don't alert on single-point spikes)
- Set appropriate `repeat_interval` in Alertmanager
- Use inhibition rules to suppress child alerts when parent fires
- Group related alerts
- Review and tune alert thresholds regularly

---

## 16. Troubleshooting

### Prometheus Won't Start

```bash
# Check config syntax
./promtool check config prometheus.yml

# Check rules syntax
./promtool check rules rules/alerts.yml
```

### Target is DOWN

1. Check **Status → Targets** in the UI for the error message
2. Confirm the target is reachable: `curl http://<target>:<port>/metrics`
3. Check firewall rules / Kubernetes NetworkPolicies
4. Verify TLS settings match between Prometheus and the target

### Metrics Missing or Stale

```promql
# Check scrape duration
scrape_duration_seconds{job="myapp"}

# Check scrape success
up{job="myapp"}

# Check for failed scrapes
scrape_samples_scraped{job="myapp"}
```

### High Memory Usage

```promql
# Check TSDB head memory
prometheus_tsdb_head_series
prometheus_tsdb_head_chunks_storage_size_bytes

# Check query load
rate(prometheus_engine_query_duration_seconds_count[5m])
```

Possible fixes:
- Reduce scrape interval
- Drop high-cardinality metrics with `metric_relabel_configs`
- Increase retention size limit to trigger compaction

### Drop Unwanted Metrics

```yaml
scrape_configs:
  - job_name: "myapp"
    metric_relabel_configs:
      # Drop all go runtime metrics
      - source_labels: [__name__]
        regex: "go_.*"
        action: drop

      # Drop specific high-cardinality label
      - regex: "user_id"
        action: labeldrop

      # Keep only specific metrics
      - source_labels: [__name__]
        regex: "(http_requests_total|http_request_duration_seconds.*)"
        action: keep
```

### Useful Prometheus Internal Metrics

```promql
# Scrape failures
rate(scrape_samples_post_metric_relabeling[5m])

# Rule evaluation time
rate(prometheus_rule_group_duration_seconds_sum[5m])
  / rate(prometheus_rule_group_duration_seconds_count[5m])

# Query duration
histogram_quantile(0.95, rate(prometheus_engine_query_duration_seconds_bucket[5m]))

# WAL corruption or issues
prometheus_tsdb_wal_corruptions_total

# Reload success
prometheus_config_last_reload_successful
```

---

## Quick Reference Card

```
# Start Prometheus
./prometheus --config.file=prometheus.yml

# Validate config
./promtool check config prometheus.yml

# Validate rules
./promtool check rules rules/*.yml

# Reload config
curl -X POST http://localhost:9090/-/reload

# Query API
curl 'http://localhost:9090/api/v1/query?query=up'
curl 'http://localhost:9090/api/v1/query_range?query=up&start=...&end=...&step=15s'

# Silence alert (Alertmanager)
curl -X POST http://localhost:9093/api/v2/silences -d '{...}'

# Check TSDB status
curl http://localhost:9090/api/v1/status/tsdb

# List all metric names
curl http://localhost:9090/api/v1/label/__name__/values
```

---

*This tutorial covers Prometheus ~v2.50+. Always refer to the [official documentation](https://prometheus.io/docs/) for the most current information.*
