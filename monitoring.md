# Grafana & Prometheus Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Monitoring](#introducing-monitoring)
2. [Prometheus Fundamentals](#prometheus-fundamentals)
3. [Prometheus Installation & Setup](#prometheus-installation--setup)
4. [Prometheus Configuration](#prometheus-configuration)
5. [Metric Types](#metric-types)
6. [PromQL Basics](#promql-basics)
7. [PromQL Advanced Queries](#promql-advanced-queries)
8. [Alerting with Prometheus](#alerting-with-prometheus)
9. [Alertmanager](#alertmanager)
10. [Exporters](#exporters)
11. [Instrumenting Applications](#instrumenting-applications)
12. [Grafana Fundamentals](#grafana-fundamentals)
13. [Grafana Installation & Setup](#grafana-installation--setup)
14. [Data Sources](#data-sources)
15. [Dashboards & Panels](#dashboards--panels)
16. [Grafana Alerting](#grafana-alerting)
17. [Provisioning & Infrastructure as Code](#provisioning--infrastructure-as-code)
18. [Full Stack with Docker Compose](#full-stack-with-docker-compose)
19. [Best Practices](#best-practices)

---

## Introducing Monitoring

Monitoring is the practice of collecting, processing, and visualizing metrics from systems and applications to understand their health, performance, and behavior. A typical monitoring stack has three pillars: metrics (numeric measurements over time), logs (timestamped event records), and traces (request paths through distributed systems). This guide covers the metrics pillar using Prometheus for collection and storage, and Grafana for visualization and alerting.

A modern monitoring stack commonly includes:

- **Prometheus**: A time-series database that scrapes (pulls) metrics from targets at regular intervals and stores them with timestamps.
- **Grafana**: A visualization platform that queries data sources like Prometheus and renders dashboards with graphs, tables, and alerts.
- **Alertmanager**: A component that receives alerts from Prometheus, deduplicates and groups them, and routes notifications to Slack, email, PagerDuty, etc.
- **Exporters**: Agents that expose metrics from third-party systems (databases, hardware, web servers) in a format Prometheus can scrape.

Key concepts:

- **Metric**: A numeric measurement with a name, labels, and a timestamp (e.g., `http_requests_total{method="GET", status="200"} 1042`).
- **Time Series**: A stream of timestamped values for a unique combination of metric name and labels.
- **Scraping**: Prometheus pulls metrics from HTTP endpoints (`/metrics`) at configured intervals.
- **Labels**: Key-value pairs attached to metrics that allow filtering and grouping (e.g., `method="GET"`).
- **Target**: An endpoint Prometheus scrapes (e.g., `localhost:9090/metrics`).
- **Dashboard**: A collection of panels in Grafana that visualize metrics from one or more data sources.

---

## Prometheus Fundamentals

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. It stores all data as time series — streams of timestamped values identified by a metric name and a set of labels. Prometheus uses a pull model: it actively scrapes metrics from configured targets over HTTP, rather than waiting for targets to push data.

### Architecture

```
┌──────────────┐     scrape     ┌──────────────┐
│  Application ├───────────────►│  Prometheus   │
│  /metrics    │                │  Server       │
└──────────────┘                │  (TSDB)       │
                                └──────┬───────┘
┌──────────────┐     scrape            │
│  Node        ├───────────────►       │
│  Exporter    │                       │
└──────────────┘                       │
                                ┌──────▼───────┐     ┌──────────────┐
                                │  Alertmanager │────►│  Slack/Email │
                                └──────────────┘     └──────────────┘
                                       │
                                ┌──────▼───────┐
                                │  Grafana      │
                                │  (Dashboards) │
                                └──────────────┘
```

Prometheus scrapes targets at a configured interval (default 15s), evaluates alerting rules against the collected data, and sends firing alerts to Alertmanager. Grafana queries Prometheus using PromQL to render dashboards.

### Data Model

Every time series is uniquely identified by its metric name and a set of labels:

```
<metric_name>{<label1>=<value1>, <label2>=<value2>, ...}
```

Example:

```
http_requests_total{method="GET", handler="/api/users", status="200"} 1542
http_requests_total{method="POST", handler="/api/users", status="201"} 87
node_cpu_seconds_total{cpu="0", mode="idle"} 78234.56
```

`http_requests_total` is the metric name. `method`, `handler`, and `status` are labels. `1542` is the current value. Each unique combination of metric name and labels is a separate time series.

---

## Prometheus Installation & Setup

### Install Prometheus

**Using Docker** (recommended):

```bash
docker run -d --name prometheus \
  -p 9090:9090 \
  -v ./prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

`-p 9090:9090` exposes the Prometheus web UI and API. The bind mount provides a custom configuration file. Prometheus is now accessible at `http://localhost:9090`.

**Binary Installation (Linux)**:

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvfz prometheus-2.53.0.linux-amd64.tar.gz
cd prometheus-2.53.0.linux-amd64

# Start Prometheus
./prometheus --config.file=prometheus.yml
```

**macOS (Homebrew)**:

```bash
brew install prometheus
brew services start prometheus
```

### Verify Installation

Open `http://localhost:9090` in your browser. The Prometheus web UI lets you execute PromQL queries, view targets, and check configuration. Navigate to **Status → Targets** to see all configured scrape targets and their health.

### Prometheus Flags

```bash
prometheus \
  --config.file=prometheus.yml \
  --storage.tsdb.path=/data \
  --storage.tsdb.retention.time=30d \
  --web.listen-address=:9090 \
  --web.enable-lifecycle \
  --web.enable-admin-api
```

`--storage.tsdb.path` sets the directory for time-series data. `--storage.tsdb.retention.time` controls how long data is kept (default 15d). `--web.enable-lifecycle` allows reloading config via HTTP POST to `/-/reload`. `--web.enable-admin-api` enables admin endpoints like snapshot and deletion.

---

## Prometheus Configuration

Prometheus is configured via a YAML file, typically `prometheus.yml`.

### Minimal Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s                 # How often to scrape targets
  evaluation_interval: 15s             # How often to evaluate alerting rules
  scrape_timeout: 10s                  # Timeout per scrape request

scrape_configs:
  - job_name: "prometheus"             # Scrape Prometheus itself
    static_configs:
      - targets: ["localhost:9090"]
```

`global` sets defaults for all scrape jobs. `scrape_interval` is how frequently Prometheus scrapes each target. `evaluation_interval` is how frequently alerting rules are evaluated. `scrape_configs` defines the list of targets to scrape.

### Multiple Scrape Jobs

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "app"
    metrics_path: "/metrics"           # Default path
    scheme: "http"                     # Default scheme
    scrape_interval: 10s               # Override global interval
    static_configs:
      - targets: ["app1:8080", "app2:8080"]
        labels:
          environment: "production"
          team: "backend"

  - job_name: "redis"
    static_configs:
      - targets: ["redis-exporter:9121"]
```

Each `job_name` groups related targets. `metrics_path` defaults to `/metrics`. `scheme` defaults to `http`. Labels added under `static_configs` are attached to all metrics from those targets. `scrape_interval` can be overridden per job.

### Relabeling

Relabeling lets you rewrite, filter, or drop labels and targets before or after scraping.

```yaml
scrape_configs:
  - job_name: "app"
    static_configs:
      - targets: ["app1:8080", "app2:8080"]
    relabel_configs:
      # Add a custom label from an existing label
      - source_labels: [__address__]
        regex: "(.+):.*"
        target_label: "hostname"
        replacement: "${1}"

      # Drop targets matching a pattern
      - source_labels: [__address__]
        regex: "internal.*"
        action: drop

    metric_relabel_configs:
      # Drop specific metrics after scraping
      - source_labels: [__name__]
        regex: "go_.*"
        action: drop
```

`relabel_configs` apply before the scrape (to targets and their labels). `metric_relabel_configs` apply after the scrape (to the collected metrics). `action: drop` discards matching targets or metrics. `action: keep` keeps only matching items. This is useful for reducing cardinality and storage.

### Service Discovery

Instead of hardcoding targets, Prometheus can discover them automatically:

```yaml
scrape_configs:
  # Kubernetes service discovery
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: "${1}"

  # File-based service discovery
  - job_name: "file-targets"
    file_sd_configs:
      - files:
          - "targets/*.json"
        refresh_interval: 5m

  # EC2 service discovery
  - job_name: "ec2"
    ec2_sd_configs:
      - region: "us-east-1"
        port: 9100
```

Service discovery eliminates manual target management. Kubernetes SD discovers pods with specific annotations. File SD reads targets from JSON or YAML files that can be updated externally. EC2 SD discovers instances by region, tags, or other metadata.

### File-Based Targets

```json
// targets/app.json
[
  {
    "targets": ["app1:8080", "app2:8080"],
    "labels": {
      "env": "production",
      "team": "backend"
    }
  }
]
```

Prometheus watches these files and picks up changes automatically without a restart.

### Reload Configuration

```bash
# Send SIGHUP
kill -HUP $(pgrep prometheus)

# HTTP POST (requires --web.enable-lifecycle)
curl -X POST http://localhost:9090/-/reload
```

Both methods reload the configuration without restarting Prometheus or losing any scraped data.

---

## Metric Types

Prometheus defines four core metric types. Each serves a different purpose.

### Counter

A counter is a cumulative metric that only goes up (or resets to zero on restart). Use it for counting events.

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200"} 1542
http_requests_total{method="POST", status="201"} 87
http_requests_total{method="GET", status="500"} 12
```

Counters are used for request counts, errors, bytes sent, tasks completed, etc. Never use a counter for values that can decrease. To get the rate of events, use the `rate()` function in PromQL.

### Gauge

A gauge is a metric that can go up and down. Use it for current values.

```
# HELP node_memory_available_bytes Available memory in bytes
# TYPE node_memory_available_bytes gauge
node_memory_available_bytes 2147483648

# HELP temperature_celsius Current temperature
# TYPE temperature_celsius gauge
temperature_celsius{location="server_room"} 22.5
```

Gauges are used for temperatures, memory usage, CPU usage, queue sizes, active connections, disk space, etc. Unlike counters, gauges represent a snapshot of a current value.

### Histogram

A histogram samples observations (e.g., request durations) and counts them in configurable buckets. It also provides a sum and count of observations.

```
# HELP http_request_duration_seconds Request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 2400
http_request_duration_seconds_bucket{le="0.01"} 2850
http_request_duration_seconds_bucket{le="0.025"} 3100
http_request_duration_seconds_bucket{le="0.05"} 3200
http_request_duration_seconds_bucket{le="0.1"} 3280
http_request_duration_seconds_bucket{le="0.25"} 3290
http_request_duration_seconds_bucket{le="0.5"} 3295
http_request_duration_seconds_bucket{le="1"} 3298
http_request_duration_seconds_bucket{le="+Inf"} 3300
http_request_duration_seconds_sum 245.67
http_request_duration_seconds_count 3300
```

Each `_bucket{le="X"}` counts requests that took ≤ X seconds. Buckets are cumulative — the `le="+Inf"` bucket equals `_count`. `_sum` is the total of all observed values. `_count` is the total number of observations. Use `histogram_quantile()` in PromQL to calculate percentiles.

### Summary

A summary is similar to a histogram but calculates quantiles on the client side.

```
# HELP rpc_duration_seconds RPC latency in seconds
# TYPE rpc_duration_seconds summary
rpc_duration_seconds{quantile="0.5"} 0.042
rpc_duration_seconds{quantile="0.9"} 0.087
rpc_duration_seconds{quantile="0.99"} 0.155
rpc_duration_seconds_sum 1234.56
rpc_duration_seconds_count 15000
```

Summaries provide pre-calculated quantiles (p50, p90, p99). Unlike histograms, summary quantiles cannot be aggregated across instances. Prefer histograms in most cases because they allow server-side percentile calculation and aggregation.

### Histogram vs Summary

| Feature               | Histogram                             | Summary                          |
| --------------------- | ------------------------------------- | -------------------------------- |
| Quantile Calculation  | Server-side (PromQL)                  | Client-side (application)        |
| Aggregation           | Can aggregate across instances        | Cannot aggregate quantiles       |
| Configuration         | Define bucket boundaries              | Define quantile targets          |
| Cost                  | Fixed number of time series (buckets) | Fixed number (quantiles)         |
| Accuracy              | Depends on bucket boundaries          | Configurable error               |
| Recommendation        | Preferred in most cases               | Use when exact quantiles needed  |

---

## PromQL Basics

PromQL (Prometheus Query Language) is a powerful functional query language for selecting, aggregating, and transforming time-series data. You use it in the Prometheus UI, Grafana dashboards, and alerting rules.

### Selectors

```promql
# Instant vector: current value of all time series with this name
http_requests_total

# Label matching: equals
http_requests_total{method="GET"}

# Label matching: not equals
http_requests_total{method!="GET"}

# Label matching: regex match
http_requests_total{handler=~"/api/.*"}

# Label matching: negative regex match
http_requests_total{handler!~"/health|/ready"}

# Multiple labels
http_requests_total{method="GET", status="200", job="api"}

# Match any value for a label
http_requests_total{method=~".+"}
```

An instant vector returns the latest value for each matching time series. `=~` matches against a regular expression (RE2 syntax). `!~` negates the regex match. At least one label matcher must not match the empty string.

### Range Vectors

```promql
# Values over the last 5 minutes
http_requests_total{method="GET"}[5m]

# Supported time units:
# ms (milliseconds), s (seconds), m (minutes),
# h (hours), d (days), w (weeks), y (years)

http_requests_total[1h]                # Last 1 hour
http_requests_total[30m]               # Last 30 minutes
node_cpu_seconds_total[5m]             # Last 5 minutes
```

Range vectors return all values within the specified time window. They are used as input to functions like `rate()`, `increase()`, and `avg_over_time()`. You cannot graph a range vector directly — it must be passed through a function first.

### Offset & @ Modifier

```promql
# Value 1 hour ago
http_requests_total offset 1h

# Rate 1 day ago
rate(http_requests_total[5m] offset 1d)

# Value at a specific Unix timestamp
http_requests_total @ 1609459200
```

`offset` shifts the query back in time. `@` evaluates at a specific Unix timestamp. These are useful for comparing current values to historical data.

### Arithmetic Operators

```promql
# Basic math
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Division
http_request_duration_seconds_sum / http_request_duration_seconds_count

# Supported operators: + - * / % ^
```

Arithmetic operators work element-wise on matching time series. Prometheus matches series by their label sets — two series operate together only if they share the same labels (unless modified with `on()` or `ignoring()`).

### Comparison Operators

```promql
# Filter: only show values > 90
node_cpu_usage_percent > 90

# Return 1 or 0 (bool comparison)
node_cpu_usage_percent > bool 90

# Supported operators: == != > < >= <=
```

Without `bool`, comparison operators filter — they drop series that do not match. With `bool`, they return 1 (true) or 0 (false) for each series.

### Logical Operators

```promql
# Union: combine two vectors
metric_a or metric_b

# Intersection: keep series present in both
metric_a and metric_b

# Complement: keep series from left not in right
metric_a unless metric_b
```

`or` returns all series from both vectors (left takes precedence for duplicates). `and` keeps only series from the left that have matching label sets in the right. `unless` keeps only series from the left that do not have matching label sets in the right.

---

## PromQL Advanced Queries

### Rate & Increase

```promql
# Per-second rate over 5 minutes (for counters)
rate(http_requests_total[5m])

# Per-second rate, handles counter resets at range boundaries
irate(http_requests_total[5m])

# Total increase over 1 hour
increase(http_requests_total[1h])
```

`rate()` calculates the per-second average rate of increase over the range window. It handles counter resets automatically. Always use `rate()` on counters — never graph a raw counter. `irate()` uses only the last two data points, making it more sensitive to spikes but noisier. `increase()` returns the total increase over the time window (equivalent to `rate() * seconds`).

### Aggregation Operators

```promql
# Sum all request rates
sum(rate(http_requests_total[5m]))

# Sum grouped by method
sum by (method) (rate(http_requests_total[5m]))

# Sum ignoring specific labels
sum without (instance) (rate(http_requests_total[5m]))

# Average CPU usage across all instances
avg(node_cpu_usage_percent)

# Maximum memory usage per job
max by (job) (node_memory_usage_bytes)

# Count number of targets
count(up)

# Count by status
count by (status) (rate(http_requests_total[5m]))

# Standard deviation
stddev by (job) (rate(http_requests_total[5m]))

# Bottom/top K
topk(5, rate(http_requests_total[5m]))
bottomk(5, rate(http_requests_total[5m]))

# Quantile across instances
quantile(0.95, rate(http_requests_total[5m]))

# Count distinct label values
count(count by (handler) (http_requests_total))
```

`sum`, `avg`, `max`, `min`, `count`, `stddev`, `stdvar`, `topk`, `bottomk`, `quantile`, and `count_values` are aggregation operators. `by (label)` groups results by the specified labels, dropping all others. `without (label)` keeps all labels except the specified ones. These are essential for rolling up metrics across many instances.

### Histogram Quantiles

```promql
# 95th percentile request duration
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# 99th percentile grouped by handler
histogram_quantile(0.99,
  sum by (handler, le) (rate(http_request_duration_seconds_bucket[5m]))
)

# 50th percentile (median)
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))

# Average request duration
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

`histogram_quantile()` calculates quantiles from histogram buckets. The `le` label (less-than-or-equal) must be preserved in any aggregation — always include `le` in the `by` clause. Dividing `_sum` by `_count` gives the average.

### Over-Time Functions

```promql
# Average value over the last hour
avg_over_time(node_cpu_usage_percent[1h])

# Max value over the last day
max_over_time(node_memory_usage_bytes[1d])

# Min value over the last hour
min_over_time(temperature_celsius[1h])

# Sum of values over time
sum_over_time(http_request_duration_seconds_sum[5m])

# Count of samples
count_over_time(up[1h])

# Standard deviation over time
stddev_over_time(latency_seconds[1h])

# Last value in range
last_over_time(up[5m])

# Predict value 4 hours from now using linear regression
predict_linear(node_filesystem_avail_bytes[6h], 4 * 3600)

# Rate of change
deriv(node_network_receive_bytes_total[5m])
```

`*_over_time()` functions operate on range vectors and return an instant vector. `predict_linear()` uses simple linear regression to predict a future value — ideal for alerting on resources that are trending toward exhaustion. `deriv()` calculates the per-second derivative using linear regression.

### Label Functions

```promql
# Replace or add labels
label_replace(up, "hostname", "$1", "instance", "(.+):.*")

# Join labels from different metrics
label_join(up, "host_port", ":", "instance", "job")
```

`label_replace()` creates or overwrites a label using a regex substitution on another label's value. `label_join()` concatenates multiple label values into a new label.

### Vector Matching

```promql
# Ignore specific labels when matching
http_errors_total / ignoring(status) http_requests_total

# Match on specific labels only
http_errors_total / on(method, handler) http_requests_total

# One-to-many matching
http_errors_total / on(method) group_left http_requests_total
```

`ignoring()` excludes labels from matching. `on()` includes only specified labels for matching. `group_left` / `group_right` enable one-to-many matching — the "group" side can have multiple series matching one series on the other side.

### Common Patterns

```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# Availability (% of successful requests)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
) * 100

# Saturation: memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
  / node_memory_MemTotal_bytes * 100

# Disk will be full in less than 24 hours
predict_linear(node_filesystem_avail_bytes[6h], 24 * 3600) < 0

# Requests per second per instance
sum by (instance) (rate(http_requests_total[5m]))

# Top 5 busiest endpoints
topk(5, sum by (handler) (rate(http_requests_total[5m])))

# Uptime: fraction of time target was up
avg_over_time(up[24h])
```

---

## Alerting with Prometheus

Prometheus evaluates alerting rules at regular intervals and sends firing alerts to Alertmanager.

### Alert Rule File

```yaml
# prometheus.yml — reference the rule file
rule_files:
  - "alerts/*.yml"
```

```yaml
# alerts/app_alerts.yml
groups:
  - name: app-alerts
    interval: 30s                      # Override evaluation_interval for this group
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
            / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes."
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      - alert: HighMemoryUsage
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
            / node_memory_MemTotal_bytes > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 90%"
          description: "Instance {{ $labels.instance }} memory is at {{ $value | humanizePercentage }}."

      - alert: DiskSpaceRunningLow
        expr: predict_linear(node_filesystem_avail_bytes[6h], 24 * 3600) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Disk space predicted to run out within 24 hours"
          description: "Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }}."

      - alert: TargetDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"
          description: "Job {{ $labels.job }} target {{ $labels.instance }} has been down for more than 2 minutes."
```

`alert` names the alert. `expr` is the PromQL expression; if it returns any results, the alert fires. `for` is the duration the condition must be true before the alert transitions from `pending` to `firing`. `labels` attach metadata to the alert for routing and grouping. `annotations` provide human-readable details. `{{ $value }}` is the expression's current value. `{{ $labels.instance }}` accesses the metric's labels.

### Recording Rules

Recording rules pre-compute expensive PromQL expressions and save the result as a new time series. This speeds up dashboard queries and alerting.

```yaml
# alerts/recording_rules.yml
groups:
  - name: recording-rules
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: instance:node_cpu_usage:avg
        expr: |
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      - record: job:http_request_duration_seconds:p95
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

Recording rule names follow the convention `level:metric:operations` — for example, `job:http_requests_total:rate5m`. Use recording rules for expressions that appear in multiple dashboards or alerts, or for expressions that are computationally expensive.

### Check Rules

```bash
# Validate rule files
promtool check rules alerts/app_alerts.yml

# Test rules against sample data
promtool test rules tests/alert_tests.yml

# Validate prometheus.yml
promtool check config prometheus.yml
```

`promtool` is included with Prometheus and validates rule syntax, config files, and can run unit tests against rules.

---

## Alertmanager

Alertmanager handles alerts sent by Prometheus. It deduplicates, groups, silences, inhibits, and routes alerts to notification channels.

### Installation

```bash
docker run -d --name alertmanager \
  -p 9093:9093 \
  -v ./alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager
```

Alertmanager web UI is available at `http://localhost:9093`.

### Connect Prometheus to Alertmanager

```yaml
# prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m                 # Mark alert resolved after silence
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alerts@example.com"
  smtp_auth_username: "alerts@example.com"
  smtp_auth_password: "app-password"
  slack_api_url: "https://hooks.slack.com/services/T00/B00/XXXX"

templates:
  - "/etc/alertmanager/templates/*.tmpl"

route:
  receiver: "slack-default"            # Default receiver
  group_by: ["alertname", "job"]       # Group alerts with same labels
  group_wait: 30s                      # Wait before sending first notification
  group_interval: 5m                   # Wait before sending updates
  repeat_interval: 4h                  # Resend if alert is still firing

  routes:
    - match:
        severity: critical
      receiver: "pagerduty"
      group_wait: 10s
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: "slack-warnings"
      repeat_interval: 12h

    - match_re:
        team: "^(backend|frontend)$"
      receiver: "slack-engineering"

receivers:
  - name: "slack-default"
    slack_configs:
      - channel: "#alerts"
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

  - name: "slack-warnings"
    slack_configs:
      - channel: "#alerts-warnings"
        send_resolved: true

  - name: "pagerduty"
    pagerduty_configs:
      - service_key: "your-pagerduty-key"
        severity: '{{ .GroupLabels.severity }}'

  - name: "email"
    email_configs:
      - to: "oncall@example.com"
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["alertname", "instance"]
```

`route` defines a routing tree. Alerts enter at the top and are matched against child routes. `group_by` groups alerts with the same labels into a single notification. `group_wait` delays the first notification to batch initial alerts. `group_interval` sets the minimum time between updates for the same group. `repeat_interval` controls how often to re-notify if an alert is still firing. `match` and `match_re` filter alerts by label values. `receivers` define notification channels. `send_resolved: true` sends a notification when the alert resolves. `inhibit_rules` suppress lower-severity alerts when a higher-severity alert is firing for the same target.

### Silences

Silences temporarily mute alerts. Create them via the Alertmanager web UI (`http://localhost:9093/#/silences`) or the API:

```bash
# Create a silence via API
curl -X POST http://localhost:9093/api/v2/silences -H "Content-Type: application/json" -d '{
  "matchers": [
    {"name": "alertname", "value": "HighMemoryUsage", "isRegex": false}
  ],
  "startsAt": "2025-01-15T00:00:00Z",
  "endsAt": "2025-01-15T06:00:00Z",
  "createdBy": "bidur",
  "comment": "Planned maintenance window"
}'
```

Use silences during planned maintenance windows to avoid noise.

---

## Exporters

Exporters are agents that expose metrics from third-party systems in Prometheus format.

### Node Exporter (System Metrics)

Exposes hardware and OS metrics: CPU, memory, disk, network, filesystem.

```bash
docker run -d --name node-exporter \
  -p 9100:9100 \
  --pid="host" \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  prom/node-exporter \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/rootfs
```

Key metrics exposed:

```promql
node_cpu_seconds_total                 # CPU time per mode (idle, user, system)
node_memory_MemTotal_bytes             # Total memory
node_memory_MemAvailable_bytes         # Available memory
node_filesystem_avail_bytes            # Available disk space
node_filesystem_size_bytes             # Total disk space
node_network_receive_bytes_total       # Network bytes received
node_network_transmit_bytes_total      # Network bytes transmitted
node_load1                             # 1-minute load average
node_disk_read_bytes_total             # Disk read bytes
node_disk_written_bytes_total          # Disk write bytes
```

Add to Prometheus configuration:

```yaml
scrape_configs:
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### cAdvisor (Container Metrics)

Exposes Docker container resource usage metrics.

```bash
docker run -d --name cadvisor \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor
```

Key metrics:

```promql
container_cpu_usage_seconds_total      # Container CPU usage
container_memory_usage_bytes           # Container memory usage
container_network_receive_bytes_total  # Container network in
container_network_transmit_bytes_total # Container network out
container_fs_usage_bytes               # Container filesystem usage
```

### Common Exporters

| Exporter              | Default Port | Metrics                            |
| --------------------- | ------------ | ---------------------------------- |
| Node Exporter         | 9100         | CPU, memory, disk, network         |
| cAdvisor              | 8080         | Docker container resources         |
| Blackbox Exporter     | 9115         | HTTP, TCP, ICMP, DNS probes        |
| MySQL Exporter        | 9104         | MySQL server metrics               |
| PostgreSQL Exporter   | 9187         | PostgreSQL server metrics          |
| Redis Exporter        | 9121         | Redis server metrics               |
| MongoDB Exporter      | 9216         | MongoDB server metrics             |
| Nginx Exporter        | 9113         | Nginx connection and request stats |
| Kafka Exporter        | 9308         | Kafka broker and consumer metrics  |

### Blackbox Exporter (Probing)

Blackbox Exporter probes endpoints over HTTP, TCP, ICMP, and DNS.

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      follow_redirects: true

  tcp_connect:
    prober: tcp
    timeout: 5s
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "blackbox-http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

The relabeling rewrites targets so that Prometheus scrapes the Blackbox Exporter while passing the actual URL as a parameter. The Blackbox Exporter probes the target and returns probe metrics.

---

## Instrumenting Applications

To expose custom metrics from your application, use a Prometheus client library.

### Python (FastAPI Example)

```bash
pip install prometheus-client
```

```python
from fastapi import FastAPI, Request
from prometheus_client import (
    Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
)
from starlette.responses import Response
import time

app = FastAPI()

# Define metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "handler", "status"]
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "Request duration in seconds",
    ["method", "handler"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

ACTIVE_REQUESTS = Gauge(
    "http_active_requests",
    "Currently active requests"
)

# Middleware to track metrics
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    ACTIVE_REQUESTS.inc()
    start_time = time.time()

    response = await call_next(request)

    duration = time.time() - start_time
    REQUEST_COUNT.labels(
        method=request.method,
        handler=request.url.path,
        status=response.status_code
    ).inc()
    REQUEST_DURATION.labels(
        method=request.method,
        handler=request.url.path
    ).observe(duration)
    ACTIVE_REQUESTS.dec()

    return response

# Metrics endpoint
@app.get("/metrics")
def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/")
def read_root():
    return {"message": "Hello, Monitoring!"}
```

`Counter` increments only. `Histogram` observes values into buckets. `Gauge` goes up and down. `.labels()` adds label values. `.inc()` increments by 1. `.observe()` records a value. `generate_latest()` serializes all metrics in Prometheus text format for the `/metrics` endpoint.

### Node.js Example

```bash
npm install prom-client
```

```javascript
const express = require("express");
const client = require("prom-client");

const app = express();

// Collect default Node.js metrics (CPU, memory, event loop, etc.)
client.collectDefaultMetrics({ prefix: "app_" });

// Custom metrics
const httpRequestCounter = new client.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
});

const httpRequestDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "Request duration in seconds",
  labelNames: ["method", "route"],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

// Middleware
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({ method: req.method, route: req.path });
  res.on("finish", () => {
    httpRequestCounter.inc({ method: req.method, route: req.path, status: res.statusCode });
    end();
  });
  next();
});

app.get("/", (req, res) => res.json({ message: "Hello, Monitoring!" }));

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(8080, () => console.log("Listening on :8080"));
```

`collectDefaultMetrics()` automatically tracks Node.js runtime metrics. `startTimer()` returns a function that records the duration when called.

### Metric Naming Conventions

- Use `snake_case` for metric names.
- Include a unit suffix: `_seconds`, `_bytes`, `_total`, `_info`.
- Counters must end with `_total`.
- Use the base unit (seconds not milliseconds, bytes not megabytes).
- Prefix with a namespace: `http_`, `node_`, `app_`.

```
# Good
http_requests_total
http_request_duration_seconds
node_memory_available_bytes

# Bad
httpRequests
request_duration_ms
memory_available_mb
```

---

## Grafana Fundamentals

Grafana is an open-source platform for monitoring and observability. It connects to data sources like Prometheus, Loki, Elasticsearch, PostgreSQL, and many others, and lets you create interactive dashboards with rich visualizations.

### Key Concepts

- **Data Source**: A connection to a database or service (Prometheus, MySQL, Loki, etc.).
- **Dashboard**: A collection of panels arranged on a grid.
- **Panel**: A single visualization (graph, gauge, table, stat, heatmap, etc.).
- **Variable**: A template parameter that lets users filter dashboards interactively (e.g., select a specific instance or job).
- **Alert**: A condition-based rule that triggers notifications when thresholds are breached.
- **Organization**: A tenant boundary for users, dashboards, and data sources.
- **Folder**: A way to organize dashboards with access control.

---

## Grafana Installation & Setup

### Install Grafana

**Using Docker** (recommended):

```bash
docker run -d --name grafana \
  -p 3000:3000 \
  grafana/grafana-oss
```

Grafana is now accessible at `http://localhost:3000`. Default credentials are `admin` / `admin`. You will be prompted to change the password on first login.

**With Persistent Storage**:

```bash
docker run -d --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_PASSWORD=strongpassword \
  grafana/grafana-oss
```

`-v grafana-data:/var/lib/grafana` persists dashboards, data sources, and settings. `GF_SECURITY_ADMIN_PASSWORD` sets the admin password via environment variable.

**macOS (Homebrew)**:

```bash
brew install grafana
brew services start grafana
```

**Linux (Ubuntu/Debian)**:

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install grafana -y
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### Grafana Configuration

Grafana is configured via `grafana.ini` or environment variables. Environment variables use the format `GF_<SECTION>_<KEY>` (uppercase).

```ini
# /etc/grafana/grafana.ini

[server]
http_port = 3000
domain = grafana.example.com
root_url = https://grafana.example.com

[security]
admin_user = admin
admin_password = strongpassword

[auth.anonymous]
enabled = false

[smtp]
enabled = true
host = smtp.gmail.com:587
user = alerts@example.com
password = app-password
from_address = alerts@example.com

[log]
level = info
```

Equivalent environment variables:

```bash
GF_SERVER_HTTP_PORT=3000
GF_SECURITY_ADMIN_PASSWORD=strongpassword
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
```

---

## Data Sources

Data sources are the backends Grafana queries to populate dashboards.

### Adding Prometheus Data Source

**Via UI**: Navigate to **Connections → Data sources → Add data source → Prometheus**. Set the URL to `http://prometheus:9090` (or `http://localhost:9090` if running locally).

**Via API**:

```bash
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

`access: "proxy"` means Grafana's backend makes the request (recommended). `access: "direct"` means the browser makes the request directly.

### Common Data Sources

| Data Source     | Type         | Use Case                          |
| --------------- | ------------ | --------------------------------- |
| Prometheus      | Time series  | Metrics monitoring                |
| Loki            | Logs         | Log aggregation and querying      |
| Elasticsearch   | Logs/Search  | Log analysis, full-text search    |
| InfluxDB        | Time series  | IoT, time-series data             |
| PostgreSQL      | SQL          | Application data, business metrics|
| MySQL           | SQL          | Application data                  |
| Tempo           | Traces       | Distributed tracing               |
| CloudWatch      | AWS metrics  | AWS service monitoring            |
| Jaeger          | Traces       | Distributed tracing               |

---

## Dashboards & Panels

### Creating a Dashboard

**Via UI**: Click **Dashboards → New → New Dashboard → Add visualization**. Select your data source and write a query.

### Panel Types

| Panel Type   | Use Case                                          |
| ------------ | ------------------------------------------------- |
| Time Series  | Metrics over time (line, area, bar charts)         |
| Stat         | Single large value with optional sparkline        |
| Gauge        | Current value against min/max range               |
| Bar Gauge    | Horizontal/vertical bars for comparing values     |
| Table        | Tabular data with sorting and filtering           |
| Heatmap      | Distribution of values over time (histogram bins) |
| Logs         | Log lines from Loki or Elasticsearch              |
| Node Graph   | Network topology and relationships                |
| Pie Chart    | Proportional breakdown                            |
| Alert List   | List of firing and pending alerts                 |

### Building Panels

For each panel, write a PromQL query and configure the visualization.

**CPU Usage Panel**:

```promql
# Query
100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# Legend: {{ instance }}
```

**Memory Usage Panel**:

```promql
# Query
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Legend: {{ instance }}
```

**Request Rate Panel**:

```promql
# Query
sum by (handler) (rate(http_requests_total[5m]))

# Legend: {{ handler }}
```

**Error Rate Panel**:

```promql
# Query A (errors)
sum(rate(http_requests_total{status=~"5.."}[5m]))

# Query B (total)
sum(rate(http_requests_total[5m]))

# Transform: use expression A / B * 100
```

**Disk Usage Panel**:

```promql
# Query
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

**p95 Latency Panel**:

```promql
# Query
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Template Variables

Variables make dashboards dynamic. Users select values from a dropdown that filters all panels.

**Create a variable**: Dashboard settings → **Variables → New variable**.

| Field       | Example                                               |
| ----------- | ----------------------------------------------------- |
| Name        | `instance`                                            |
| Type        | Query                                                 |
| Data Source  | Prometheus                                           |
| Query       | `label_values(up, instance)`                         |
| Multi-value | Enable to allow selecting multiple values             |
| Include All | Add an "All" option                                  |

Use the variable in queries:

```promql
# Single value
rate(http_requests_total{instance="$instance"}[5m])

# Multi-value (regex)
rate(http_requests_total{instance=~"$instance"}[5m])
```

Common variable queries:

```promql
label_values(up, job)                  # All job names
label_values(up, instance)             # All instances
label_values(node_cpu_seconds_total, cpu)  # All CPUs
label_values(http_requests_total{job="$job"}, handler)  # Handlers for selected job
query_result(count by (status) (http_requests_total))   # Dynamic values from query
```

`$variable` substitutes the selected value. `=~"$variable"` uses regex matching, which works correctly when multiple values are selected (Grafana joins them with `|`).

### Dashboard JSON Model

Dashboards can be exported and imported as JSON. Go to **Dashboard settings → JSON Model** to view or copy the full JSON.

```bash
# Export dashboard via API
curl -s http://admin:admin@localhost:3000/api/dashboards/uid/<dashboard-uid> | jq .

# Import dashboard via API
curl -X POST http://admin:admin@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

### Community Dashboards

Grafana.com hosts thousands of pre-built dashboards. Import them by ID:

| Dashboard                | ID    | Use Case                        |
| ------------------------ | ----- | ------------------------------- |
| Node Exporter Full       | 1860  | System metrics (CPU, RAM, disk) |
| Docker Container         | 893   | Container resource usage        |
| Kubernetes Cluster       | 6417  | K8s cluster overview            |
| Prometheus 2.0 Stats     | 3662  | Prometheus self-monitoring      |
| Nginx                    | 12708 | Nginx request metrics           |
| PostgreSQL               | 9628  | PostgreSQL database metrics     |
| Redis                    | 11835 | Redis server metrics            |

**Import via UI**: Dashboards → New → Import → Enter the dashboard ID → Select data source → Import.

---

## Grafana Alerting

Grafana has a built-in alerting system (unified alerting, introduced in Grafana 8+) that can replace or complement Prometheus Alertmanager.

### Creating Alert Rules

**Via UI**: Alerting → Alert rules → New alert rule.

1. **Define query**: Write the PromQL expression.
2. **Set condition**: Define threshold (e.g., value > 90 for 5 minutes).
3. **Set evaluation**: Choose folder, evaluation group, and interval.
4. **Add labels**: For routing (e.g., `severity=critical`).
5. **Add annotations**: Summary and description with template variables.

### Alert Rule Example (YAML provisioning)

```yaml
# alert-rules.yml
apiVersion: 1
groups:
  - orgId: 1
    name: app-alerts
    folder: Application
    interval: 1m
    rules:
      - uid: high-cpu-alert
        title: High CPU Usage
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: prometheus
            model:
              expr: 100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
              intervalMs: 1000
              maxDataPoints: 43200
          - refId: C
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: __expr__
            model:
              type: threshold
              expression: A
              conditions:
                - evaluator:
                    type: gt
                    params: [90]
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "CPU usage is above 90%"
```

### Contact Points

Contact points define where notifications are sent.

**Via UI**: Alerting → Contact points → New contact point.

Supported integrations: Slack, Email, PagerDuty, OpsGenie, Microsoft Teams, Webhook, Discord, Telegram, and many more.

**Slack Example** (provisioning):

```yaml
# contactpoints.yml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: slack-alerts
    receivers:
      - uid: slack-receiver
        type: slack
        settings:
          url: "https://hooks.slack.com/services/T00/B00/XXXX"
          recipient: "#alerts"
          title: '{{ template "default.title" . }}'
          text: '{{ template "default.message" . }}'
```

### Notification Policies

Notification policies route alerts to contact points based on label matching.

**Via UI**: Alerting → Notification policies.

```yaml
# notification-policies.yml
apiVersion: 1
policies:
  - orgId: 1
    receiver: slack-alerts
    group_by: ["alertname", "grafana_folder"]
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
    routes:
      - receiver: pagerduty-critical
        matchers:
          - severity = critical
        group_wait: 10s
        repeat_interval: 1h
      - receiver: slack-warnings
        matchers:
          - severity = warning
        repeat_interval: 12h
```

### Silences & Mute Timings

**Silences** temporarily suppress alerts (via UI: Alerting → Silences → Create silence).

**Mute timings** define recurring windows when alerts should not fire:

```yaml
# mute-timings.yml
apiVersion: 1
muteTimes:
  - orgId: 1
    name: maintenance-window
    time_intervals:
      - times:
          - start_time: "02:00"
            end_time: "04:00"
        weekdays: ["saturday", "sunday"]
```

---

## Provisioning & Infrastructure as Code

Grafana supports file-based provisioning — you define data sources, dashboards, and alert rules in YAML files, and Grafana loads them automatically on startup.

### Directory Structure

```
grafana/
├── provisioning/
│   ├── datasources/
│   │   └── datasource.yml
│   ├── dashboards/
│   │   ├── dashboard.yml          # Dashboard provider config
│   │   └── json/
│   │       ├── node-overview.json
│   │       └── app-metrics.json
│   ├── alerting/
│   │   ├── alert-rules.yml
│   │   ├── contactpoints.yml
│   │   └── notification-policies.yml
│   └── plugins/
│       └── plugin.yml
└── grafana.ini
```

### Provisioning Data Sources

```yaml
# provisioning/datasources/datasource.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
      httpMethod: POST

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

`editable: false` prevents manual changes in the UI, enforcing config-as-code. `jsonData.timeInterval` sets the minimum scrape interval hint.

### Provisioning Dashboards

```yaml
# provisioning/dashboards/dashboard.yml
apiVersion: 1

providers:
  - name: "default"
    orgId: 1
    folder: "Provisioned"
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/provisioning/dashboards/json
      foldersFromFilesStructure: true
```

`disableDeletion: true` prevents dashboards from being deleted through the UI. `updateIntervalSeconds` controls how often Grafana scans the directory for changes. Place dashboard JSON files in the `json/` directory.

### Grafana Docker with Provisioning

```bash
docker run -d --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -v ./provisioning:/etc/grafana/provisioning \
  -e GF_SECURITY_ADMIN_PASSWORD=strongpassword \
  grafana/grafana-oss
```

Grafana reads the provisioning directory on startup and configures everything automatically.

---

## Full Stack with Docker Compose

This section brings everything together: Prometheus, Grafana, Alertmanager, Node Exporter, and cAdvisor in a single `compose.yaml`.

### Project Structure

```
monitoring/
├── compose.yaml
├── prometheus/
│   ├── prometheus.yml
│   └── alerts/
│       └── rules.yml
├── alertmanager/
│   └── alertmanager.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasource.yml
        └── dashboards/
            ├── dashboard.yml
            └── json/
                └── node-overview.json
```

### compose.yaml

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts:/etc/prometheus/alerts
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=30d"
      - "--web.enable-lifecycle"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    depends_on:
      - prometheus
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "grafana"
    static_configs:
      - targets: ["grafana:3000"]
```

### alerts/rules.yml

```yaml
groups:
  - name: system-alerts
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"

      - alert: HighCPUUsage
        expr: |
          100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage above 85% on {{ $labels.instance }}"
          description: "Current value: {{ $value | printf \"%.1f\" }}%"

      - alert: HighMemoryUsage
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
            / node_memory_MemTotal_bytes * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 90% on {{ $labels.instance }}"

      - alert: DiskSpaceLow
        expr: |
          100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage above 85% on {{ $labels.instance }}"

      - alert: ContainerHighCPU
        expr: |
          sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m])) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} CPU above 80%"

      - alert: ContainerHighMemory
        expr: |
          sum by (name) (container_memory_usage_bytes{name!=""}) > 1073741824
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} using more than 1GB memory"
```

### alertmanager.yml

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: "slack"
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: "slack"
    slack_configs:
      - channel: "#alerts"
        api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
```

### provisioning/datasources/datasource.yml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### provisioning/dashboards/dashboard.yml

```yaml
apiVersion: 1

providers:
  - name: "default"
    orgId: 1
    folder: "Provisioned"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/provisioning/dashboards/json
```

### Start the Stack

```bash
docker compose up -d                   # Start all services
docker compose ps                      # Verify everything is running
docker compose logs -f prometheus      # Follow Prometheus logs
```

After starting, access:

- **Prometheus**: `http://localhost:9090` — query metrics, view targets
- **Grafana**: `http://localhost:3000` — dashboards and visualizations
- **Alertmanager**: `http://localhost:9093` — alert routing and silences
- **Node Exporter**: `http://localhost:9100/metrics` — raw system metrics
- **cAdvisor**: `http://localhost:8080` — container metrics UI

Import community dashboards in Grafana (Dashboard ID `1860` for Node Exporter Full, `893` for Docker containers) to get production-ready visualizations instantly.

---

## Best Practices

### Prometheus

- **Use recording rules** for frequently queried or expensive PromQL expressions.
- **Set appropriate scrape intervals**: 15s is a good default. Do not scrape faster than necessary.
- **Use `rate()` on counters**: Never graph raw counters; always wrap them in `rate()` or `increase()`.
- **Limit label cardinality**: Avoid labels with unbounded values (user IDs, request IDs, email addresses). High cardinality explodes storage and slows queries.
- **Use meaningful metric names**: Follow the convention `namespace_subsystem_name_unit` (e.g., `http_request_duration_seconds`).
- **Set retention appropriately**: `--storage.tsdb.retention.time=30d` is common. Longer retention requires more disk.
- **Use federation or remote write** for multi-cluster setups.
- **Monitor Prometheus itself**: Scrape `localhost:9090/metrics` and alert on `prometheus_tsdb_head_series` growing unexpectedly.
- **Use `metric_relabel_configs`** to drop unneeded metrics and reduce cardinality at ingest time.

### Grafana

- **Use template variables** for all dashboards to make them reusable across environments.
- **Provision dashboards as code**: Store JSON files in version control and use file-based provisioning.
- **Use folders and permissions** to organize dashboards by team or service.
- **Import community dashboards** as starting points and customize them.
- **Set meaningful panel titles and descriptions** for clarity.
- **Use dashboard links** to connect related dashboards.
- **Set appropriate time ranges** and refresh intervals per dashboard.
- **Use annotations** to mark deployments, incidents, and configuration changes on graphs.

### Alerting

- **Alert on symptoms, not causes**: Alert on "error rate > 5%" rather than "CPU > 90%".
- **Use `for` duration**: Avoid alert storms by requiring conditions to persist before firing.
- **Set appropriate severity levels**: Critical for pages, warning for investigation.
- **Include runbook URLs** in alert annotations.
- **Group related alerts** to reduce notification noise.
- **Test alert rules** with `promtool test rules` before deploying.
- **Use inhibition rules** to suppress lower-priority alerts when critical ones are firing.
- **Silence alerts during maintenance** to avoid false alarms.
- **Keep alert rules simple**: Complex expressions are harder to debug and maintain.

### General

- **Use labels consistently** across all services and exporters.
- **Document your monitoring setup** — what is monitored, where dashboards live, who gets alerted.
- **Store all config in version control**: `prometheus.yml`, alert rules, Grafana provisioning, `compose.yaml`.
- **Separate monitoring from application infrastructure** when possible.
- **Use the RED method** for services: Rate (requests/sec), Errors (error rate), Duration (latency).
- **Use the USE method** for resources: Utilization (%), Saturation (queue depth), Errors (error count).
- **Review and prune dashboards** regularly — unused dashboards add confusion.
