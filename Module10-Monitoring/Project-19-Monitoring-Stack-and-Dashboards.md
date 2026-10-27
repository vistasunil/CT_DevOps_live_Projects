# Project 19 — Monitoring Stack and Service Dashboards

| | |
|---|---|
| **Module** | 10 — Monitoring |
| **Level** | L2 — Engineer |
| **Role you play** | DevOps/SRE building the observability platform |
| **Scope** | `Prometheus-Grafana-hands-on.md` — Prometheus 3.5.0, node_exporter 1.9.1, Grafana, PromQL, exporters |

---

## 1. Business problem

Outages are discovered by customers, not by the team. When something breaks, the first question is
"was the server slow, or the app?" and nobody can answer it, because the only evidence is
`top` run after the fact. There is no history, no capacity data, and no way to tell whether last
week's release made things worse.

Build the **monitoring platform**: Prometheus scraping infrastructure and application metrics,
Grafana dashboards provisioned as code, and a documented metric/dashboard standard the whole
team follows.

## 2. Objectives

1. Install and configure Prometheus 3.5.0 as a systemd service with sane retention.
2. Instrument hosts with node_exporter 1.9.1 and containers/apps with exporters.
3. Instrument a sample application with custom business metrics.
4. Install Grafana and **provision** datasources and dashboards from files, not clicks.
5. Build three dashboards: infrastructure, service (RED), and capacity.
6. Write the metric naming/labelling standard and a PromQL cookbook.

## 3. Scope

**In scope:** Prometheus install/config/`scrape_configs`/relabelling/retention, PromQL
(instant/range vectors, `rate`, `irate`, `increase`, `histogram_quantile`, aggregation,
`by`/`without`), node_exporter, blackbox-style probing basics, cAdvisor/kube metrics,
app instrumentation with a client library, Grafana install via apt keyring, provisioning YAML,
dashboard JSON, variables, recording rules.

**Out of scope:** alerting rules, Alertmanager and on-call (Project 20), long-term storage
(Thanos/Mimir), log aggregation, tracing.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** MON-SOP-019 · **Title:** Metrics, instrumentation and dashboard standard

### 4.1 What every service must expose

| # | Requirement |
|---|---|
| 1 | A `/metrics` endpoint in Prometheus text format |
| 2 | **RED** metrics: **R**ate, **E**rrors, **D**uration for every request path |
| 3 | A `*_build_info` metric carrying version/commit |
| 4 | Latency as a **histogram**, never a pre-computed average |
| 5 | Consistent labels: `job`, `instance`, `env`, `service` |
| 6 | Health endpoint separate from the metrics endpoint |

### 4.2 The four golden signals

| Signal | Typical query shape |
|---|---|
| **Latency** | `histogram_quantile(0.95, sum(rate(..._bucket[5m])) by (le))` |
| **Traffic** | `sum(rate(..._requests_total[5m]))` |
| **Errors** | `sum(rate(..._requests_total{status=~"5.."}[5m])) / sum(rate(..._requests_total[5m]))` |
| **Saturation** | CPU/memory/disk/queue depth utilisation |

### 4.3 Metric naming rules

| Rule | Example |
|---|---|
| `<namespace>_<subsystem>_<name>_<unit>` | `orders_http_request_duration_seconds` |
| Base units only (seconds, bytes, ratios) | not `_ms`, not `_mb` |
| Counters end in `_total` | `orders_http_requests_total` |
| Never put unbounded values in labels | **no** user IDs, order IDs, full URLs, timestamps |
| Keep label cardinality under control | path templates (`/order/:id`), not raw paths |

> High cardinality is the number-one way to kill a Prometheus server. Every unique label
> combination is a separate time series.

### 4.4 Dashboard standard

| # | Rule |
|---|---|
| 1 | Three tiers: **Infrastructure**, **Service (RED)**, **Capacity** |
| 2 | Top row = the four golden signals; detail below |
| 3 | Every dashboard has `env` and `instance`/`service` template variables |
| 4 | Panels state units and thresholds explicitly |
| 5 | Dashboards are **provisioned from Git**, not edited in the UI |
| 6 | Every panel must answer a question someone actually asks during an incident |

### 4.5 Retention and sizing
Start with 15 days retention and a size cap. Review disk usage weekly:
`prometheus_tsdb_head_series` and `rate(prometheus_tsdb_head_samples_appended_total[5m])` tell
you whether cardinality is growing.

---

## 5. Hands-on implementation

### Step 1 — Install Prometheus 3.5.0

```bash
PROM_VERSION=3.5.0

sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

cd /tmp
curl -fsSLO https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar xzf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64

sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo cp -r consoles console_libraries /etc/prometheus/ 2>/dev/null || true
sudo chown -R prometheus:prometheus /etc/prometheus

prometheus --version
```

`/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=10GB \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

> Prometheus 3.x removed the `--web.console.*` flags that older guides use. Do not add them —
> the service will refuse to start.

### Step 2 — node_exporter 1.9.1 on every monitored host

```bash
NE_VERSION=1.9.1

sudo useradd --no-create-home --shell /bin/false node_exporter
cd /tmp
curl -fsSLO https://github.com/prometheus/node_exporter/releases/download/v${NE_VERSION}/node_exporter-${NE_VERSION}.linux-amd64.tar.gz
tar xzf node_exporter-${NE_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NE_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

`/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes \
  --web.listen-address=0.0.0.0:9100

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s http://localhost:9100/metrics | head -20
```

### Step 3 — Prometheus configuration

`/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  external_labels:
    cluster: "acme-dev"
    env:     "dev"

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels: { service: "prometheus", tier: "platform" }

  - job_name: "node"
    static_configs:
      - targets: ["10.128.0.11:9100"]
        labels: { service: "app-server", tier: "compute", role: "web" }
      - targets: ["10.128.0.12:9100"]
        labels: { service: "app-server", tier: "compute", role: "app" }
      - targets: ["10.128.0.13:9100"]
        labels: { service: "db-server",  tier: "data",    role: "db" }
    relabel_configs:
      # Use a readable instance name instead of ip:port
      - source_labels: [__address__]
        regex: "([^:]+):.*"
        target_label: instance
        replacement: "${1}"

  - job_name: "orders-api"
    metrics_path: /metrics
    static_configs:
      - targets: ["10.128.0.12:8000"]
        labels: { service: "orders-api", tier: "application" }

  - job_name: "cadvisor"
    static_configs:
      - targets: ["10.128.0.12:8080"]
        labels: { service: "containers", tier: "platform" }

  - job_name: "nginx"
    static_configs:
      - targets: ["10.128.0.11:9113"]
        labels: { service: "nginx", tier: "web" }
```

```bash
sudo mkdir -p /etc/prometheus/rules
sudo chown -R prometheus:prometheus /etc/prometheus
promtool check config /etc/prometheus/prometheus.yml     # ALWAYS before reloading

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus --no-pager
```

```bash
# Firewall: your IP only
gcloud compute firewall-rules create allow-monitoring \
  --allow tcp:9090,tcp:3000 --source-ranges <your-ip>/32
```

Browse `http://<vm-ip>:9090/targets` — every target must be **UP**.

### Step 4 — Container metrics with cAdvisor

```bash
docker run -d \
  --name cadvisor \
  --restart unless-stopped \
  -p 8080:8080 \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:v0.52.1

curl -s http://localhost:8080/metrics | grep container_cpu_usage_seconds_total | head -3
```

### Step 5 — Instrument the application

`app/app.py` (Flask + `prometheus_client`):

```python
#!/usr/bin/env python3
"""Sample service exposing RED metrics per MON-SOP-019."""
import os
import random
import time

from flask import Flask, Response, request
from prometheus_client import (
    Counter, Histogram, Gauge, Info,
    CONTENT_TYPE_LATEST, generate_latest,
)

app = Flask(__name__)

SERVICE = "orders"
VERSION = os.getenv("APP_VERSION", "1.0.0")
COMMIT = os.getenv("GIT_SHA", "unknown")

# --- metric definitions (naming per SOP 4.3) -------------------------------
REQUESTS = Counter(
    "orders_http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"],      # endpoint is a TEMPLATE, never a raw path
)

LATENCY = Histogram(
    "orders_http_request_duration_seconds",
    "Request duration in seconds",
    ["method", "endpoint"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10),
)

IN_PROGRESS = Gauge(
    "orders_http_requests_in_progress",
    "In-flight HTTP requests",
    ["endpoint"],
)

QUEUE_DEPTH = Gauge(
    "orders_queue_depth",
    "Pending orders waiting to be processed",
)

ORDERS_VALUE = Counter(
    "orders_value_total",
    "Cumulative monetary value of accepted orders",
    ["currency"],
)

BUILD = Info("orders_build", "Build information")
BUILD.info({"version": VERSION, "commit": COMMIT, "service": SERVICE})


@app.before_request
def _start_timer():
    request._start = time.perf_counter()
    IN_PROGRESS.labels(endpoint=request.endpoint or "unknown").inc()


@app.after_request
def _record(response):
    endpoint = request.endpoint or "unknown"
    elapsed = time.perf_counter() - getattr(request, "_start", time.perf_counter())
    LATENCY.labels(request.method, endpoint).observe(elapsed)
    REQUESTS.labels(request.method, endpoint, response.status_code).inc()
    IN_PROGRESS.labels(endpoint=endpoint).dec()
    return response


@app.route("/health")
def health():
    return {"status": "ok", "version": VERSION}


@app.route("/orders", methods=["GET"])
def list_orders():
    time.sleep(random.uniform(0.01, 0.12))
    QUEUE_DEPTH.set(random.randint(0, 25))
    return {"orders": [], "count": 0}


@app.route("/orders", methods=["POST"])
def create_order():
    time.sleep(random.uniform(0.05, 0.4))
    if random.random() < 0.05:                       # simulated 5% failure rate
        return {"error": "downstream unavailable"}, 503
    ORDERS_VALUE.labels(currency="GBP").inc(random.uniform(10, 500))
    return {"status": "accepted"}, 201


@app.route("/slow")
def slow():
    time.sleep(random.uniform(1.0, 3.0))
    return {"status": "eventually"}


@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

```bash
sudo apt-get install -y python3-pip python3-venv
python3 -m venv ~/appenv
~/appenv/bin/pip install flask prometheus_client gunicorn
~/appenv/bin/gunicorn -w 2 -b 0.0.0.0:8000 app:app &

curl -s http://localhost:8000/metrics | grep orders_ | head -20
```

Reload Prometheus and confirm the new target:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

### Step 6 — Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server --no-pager
```

Change the default password immediately:

```bash
sudo grafana-cli admin reset-admin-password '<STRONG_PASSWORD_FROM_YOUR_VAULT>'
```

### Step 7 — Provision the datasource and dashboards as code

`/etc/grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
```

`/etc/grafana/provisioning/dashboards/acme.yml`:

```yaml
apiVersion: 1
providers:
  - name: 'acme-dashboards'
    orgId: 1
    folder: 'ACME'
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    allowUiUpdates: false        # dashboards come from Git, not the UI
    options:
      path: /var/lib/grafana/dashboards
```

```bash
sudo mkdir -p /var/lib/grafana/dashboards
sudo chown -R grafana:grafana /var/lib/grafana/dashboards
```

Import the community node_exporter dashboard as a starting point:

```bash
curl -fsSL -o /tmp/node.json \
  https://grafana.com/api/dashboards/1860/revisions/latest/download
sudo mv /tmp/node.json /var/lib/grafana/dashboards/node-exporter-full.json
sudo chown grafana:grafana /var/lib/grafana/dashboards/node-exporter-full.json
```

`/var/lib/grafana/dashboards/orders-service-red.json` (own dashboard, abbreviated to the key panels):

```json
{
  "title": "Orders Service - RED",
  "uid": "orders-red",
  "tags": ["acme", "service"],
  "timezone": "browser",
  "refresh": "30s",
  "templating": {
    "list": [
      {
        "name": "env", "type": "query", "datasource": "Prometheus",
        "query": "label_values(orders_http_requests_total, env)",
        "refresh": 1, "includeAll": false
      },
      {
        "name": "endpoint", "type": "query", "datasource": "Prometheus",
        "query": "label_values(orders_http_requests_total{env=\"$env\"}, endpoint)",
        "refresh": 1, "includeAll": true, "multi": true
      }
    ]
  },
  "panels": [
    {
      "type": "timeseries", "title": "Request rate (req/s)",
      "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
      "fieldConfig": {"defaults": {"unit": "reqps"}},
      "targets": [{
        "expr": "sum by (endpoint) (rate(orders_http_requests_total{env=\"$env\",endpoint=~\"$endpoint\"}[5m]))",
        "legendFormat": "{{endpoint}}"
      }]
    },
    {
      "type": "timeseries", "title": "Error ratio (5xx)",
      "gridPos": {"h": 8, "w": 8, "x": 8, "y": 0},
      "fieldConfig": {"defaults": {"unit": "percentunit",
        "thresholds": {"mode": "absolute", "steps": [
          {"color": "green", "value": null},
          {"color": "orange", "value": 0.01},
          {"color": "red", "value": 0.05}]}}},
      "targets": [{
        "expr": "sum(rate(orders_http_requests_total{env=\"$env\",status=~\"5..\"}[5m])) / clamp_min(sum(rate(orders_http_requests_total{env=\"$env\"}[5m])), 0.001)",
        "legendFormat": "error ratio"
      }]
    },
    {
      "type": "timeseries", "title": "Latency p50 / p95 / p99",
      "gridPos": {"h": 8, "w": 8, "x": 16, "y": 0},
      "fieldConfig": {"defaults": {"unit": "s"}},
      "targets": [
        {"expr": "histogram_quantile(0.50, sum by (le) (rate(orders_http_request_duration_seconds_bucket{env=\"$env\"}[5m])))", "legendFormat": "p50"},
        {"expr": "histogram_quantile(0.95, sum by (le) (rate(orders_http_request_duration_seconds_bucket{env=\"$env\"}[5m])))", "legendFormat": "p95"},
        {"expr": "histogram_quantile(0.99, sum by (le) (rate(orders_http_request_duration_seconds_bucket{env=\"$env\"}[5m])))", "legendFormat": "p99"}
      ]
    },
    {
      "type": "stat", "title": "In-flight requests",
      "gridPos": {"h": 6, "w": 6, "x": 0, "y": 8},
      "targets": [{"expr": "sum(orders_http_requests_in_progress{env=\"$env\"})"}]
    },
    {
      "type": "stat", "title": "Queue depth",
      "gridPos": {"h": 6, "w": 6, "x": 6, "y": 8},
      "targets": [{"expr": "max(orders_queue_depth{env=\"$env\"})"}]
    },
    {
      "type": "table", "title": "Running version",
      "gridPos": {"h": 6, "w": 12, "x": 12, "y": 8},
      "targets": [{"expr": "orders_build_info", "format": "table", "instant": true}]
    }
  ],
  "schemaVersion": 39,
  "version": 1
}
```

```bash
sudo systemctl restart grafana-server
```

Browse `http://<vm-ip>:3000` → folder **ACME** → the dashboards appear automatically.

### Step 8 — PromQL cookbook (save this as `docs/promql-cookbook.md`)

```promql
# --- Infrastructure --------------------------------------------------------

# CPU utilisation % per host
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory used %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Root filesystem used %
100 - (node_filesystem_avail_bytes{mountpoint="/",fstype!="tmpfs"}
       / node_filesystem_size_bytes{mountpoint="/",fstype!="tmpfs"} * 100)

# Predicted hours until the disk fills (linear projection over 6h)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*3600) < 0

# Disk I/O saturation
rate(node_disk_io_time_seconds_total[5m])

# Network throughput (bytes/s), excluding loopback
sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[5m]))

# Host up/down
up{job="node"} == 0

# Time since last boot
node_time_seconds - node_boot_time_seconds

# Load average per core
node_load5 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})

# --- Service (RED) ---------------------------------------------------------

# Throughput
sum(rate(orders_http_requests_total[5m]))

# Error ratio
sum(rate(orders_http_requests_total{status=~"5.."}[5m]))
  / sum(rate(orders_http_requests_total[5m]))

# p95 latency overall
histogram_quantile(0.95,
  sum by (le) (rate(orders_http_request_duration_seconds_bucket[5m])))

# p95 latency per endpoint
histogram_quantile(0.95,
  sum by (le, endpoint) (rate(orders_http_request_duration_seconds_bucket[5m])))

# Apdex-style: fraction of requests under 300ms
sum(rate(orders_http_request_duration_seconds_bucket{le="0.25"}[5m]))
  / sum(rate(orders_http_request_duration_seconds_count[5m]))

# Requests in the last hour (counters: use increase, not rate)
sum(increase(orders_http_requests_total[1h]))

# Top 5 slowest endpoints
topk(5, histogram_quantile(0.95,
  sum by (le, endpoint) (rate(orders_http_request_duration_seconds_bucket[5m]))))

# --- Containers ------------------------------------------------------------

container_memory_usage_bytes{name!=""}
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# --- Prometheus self-health / cardinality ----------------------------------

prometheus_tsdb_head_series                                   # total active series
topk(10, count by (__name__)({__name__=~".+"}))               # worst metric names
rate(prometheus_tsdb_head_samples_appended_total[5m])         # ingestion rate
scrape_duration_seconds > 5                                   # slow targets
```

### Step 9 — Recording rules for expensive queries

`/etc/prometheus/rules/recording.yml`:

```yaml
groups:
  - name: orders-red
    interval: 30s
    rules:
      - record: job:orders_requests:rate5m
        expr: sum by (job) (rate(orders_http_requests_total[5m]))

      - record: job:orders_errors:ratio5m
        expr: |
          sum by (job) (rate(orders_http_requests_total{status=~"5.."}[5m]))
            / clamp_min(sum by (job) (rate(orders_http_requests_total[5m])), 0.001)

      - record: job:orders_latency_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (rate(orders_http_request_duration_seconds_bucket[5m])))

  - name: node-capacity
    interval: 30s
    rules:
      - record: instance:node_cpu_utilisation:rate5m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: instance:node_memory_utilisation:ratio
        expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

```bash
promtool check rules /etc/prometheus/rules/recording.yml
curl -X POST http://localhost:9090/-/reload
# verify the new series exist
curl -s 'http://localhost:9090/api/v1/query?query=job:orders_requests:rate5m' | head -c 400
```

### Step 10 — Generate load and validate the dashboards

```bash
sudo apt-get install -y apache2-utils

# Steady traffic
ab -n 5000 -c 10 http://10.128.0.12:8000/orders

# POST traffic (will produce ~5% 503s by design)
ab -n 2000 -c 20 -p /dev/null -T application/json http://10.128.0.12:8000/orders

# Latency spike
for i in $(seq 1 50); do curl -s http://10.128.0.12:8000/slow > /dev/null & done

# CPU saturation on a host
sudo apt-get install -y stress-ng
stress-ng --cpu 2 --timeout 180s

# Disk fill (watch predict_linear react), then clean up
fallocate -l 2G /tmp/filler && sleep 120 && rm /tmp/filler
```

Confirm on the dashboard: request rate rises, error ratio shows ~5 %, p95 jumps during the `/slow`
burst, CPU utilisation panel reacts, disk prediction changes.

### Step 11 — Capacity dashboard queries

Create `/var/lib/grafana/dashboards/capacity.json` with panels for:

```promql
# Headroom per host
100 - instance:node_cpu_utilisation:rate5m
(1 - instance:node_memory_utilisation:ratio) * 100

# Growth trend: series count over time (cardinality budget)
prometheus_tsdb_head_series

# Busiest hour of the day, last 7 days
max_over_time(job:orders_requests:rate5m[7d])

# Week-on-week traffic change
sum(rate(orders_http_requests_total[1h]))
  / sum(rate(orders_http_requests_total[1h] offset 7d))
```

---

## 6. Acceptance criteria

- [ ] Prometheus 3.5.0 running under systemd with retention limits and **no** deprecated console flags
- [ ] `promtool check config` and `check rules` pass before every reload
- [ ] All targets **UP** on `/targets`, with readable `instance` labels from relabelling
- [ ] node_exporter on every host; cAdvisor collecting container metrics
- [ ] Application exposes RED metrics plus a `build_info` metric; latency is a histogram
- [ ] No unbounded-cardinality labels (verified with the `topk` cardinality query)
- [ ] Grafana installed via keyring; default password changed
- [ ] Datasource and dashboards **provisioned from files**, `allowUiUpdates: false`
- [ ] Three dashboards exist: infrastructure, service RED, capacity
- [ ] Dashboards use `env`/`endpoint` template variables
- [ ] Recording rules exist for the expensive queries and are used by panels
- [ ] Load tests visibly and correctly move every panel
- [ ] PromQL cookbook committed to the repo

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Prometheus won't start | Deprecated `--web.console.*` flags (3.x) or bad YAML | Remove the flags; `promtool check config`; `journalctl -u prometheus` |
| Target `DOWN` — connection refused | Exporter not running or bound to localhost | `systemctl status node_exporter`; bind `0.0.0.0`; check firewall |
| Target `DOWN` — context deadline exceeded | Scrape slower than `scrape_timeout` | Raise the timeout or reduce what the exporter collects |
| No app metrics | Wrong `metrics_path`/port, or `/metrics` not registered | `curl` the endpoint directly first |
| Grafana shows "No data" | Wrong datasource URL, time range, or a label that doesn't exist | Test the query in the Prometheus UI first |
| Prometheus memory keeps growing | Cardinality explosion | `topk(10, count by (__name__)({__name__=~".+"}))`; drop offending labels with `metric_relabel_configs` |
| Disk fills up | No retention limits | `--storage.tsdb.retention.time/.size` |
| `rate()` returns nothing | Range shorter than 2 scrape intervals, or a gauge used with `rate` | Use ≥4× scrape interval; `rate` is for counters only |
| Latency percentiles look wrong | Averaging an average, or buckets too coarse | `histogram_quantile` over summed `_bucket` rates; tune buckets |
| Dashboard edits vanish after restart | Provisioned dashboards are read-only | Edit the JSON in Git — that is the intent |
| Counter resets cause spikes | App restarted | `rate`/`increase` handle resets; do not use raw `delta` on counters |

---

## 8. Cleanup

```bash
sudo systemctl disable --now prometheus node_exporter grafana-server
docker rm -f cadvisor
sudo rm -rf /var/lib/prometheus /etc/prometheus
gcloud compute firewall-rules delete allow-monitoring
```

---

## 9. Deliverables

1. `monitoring/` in Git: `prometheus.yml`, `rules/recording.yml`, systemd units, Grafana
   provisioning YAML, dashboard JSON files.
2. `MON-SOP-019.md` — instrumentation requirements, naming rules, dashboard standard.
3. `docs/promql-cookbook.md`.
4. Instrumented `app.py` demonstrating RED metrics.
5. Screenshots: Prometheus `/targets` all green; the three dashboards during the load test.
6. A short cardinality report: current `prometheus_tsdb_head_series` and the top 10 metrics.

---

## 10. Interview talking points

* *"What do you monitor first on a new service?"* — The four golden signals / RED: rate, errors,
  duration, plus saturation. Everything else is detail.
* *"Counter vs gauge vs histogram?"* — Counters only increase (use `rate`/`increase`); gauges go
  up and down; histograms bucket observations so you can compute percentiles **after the fact**
  across instances — an average latency hides every outlier.
* *"Why not store the 95th percentile directly?"* — Percentiles cannot be averaged or aggregated
  across instances. Histograms can.
* *"How do you kill a Prometheus server?"* — Unbounded label cardinality: user IDs, raw URLs,
  timestamps in labels. I template paths and audit with the cardinality query.
* *"Push vs pull?"* — Prometheus pulls, which gives free target-health (`up`) and avoids clients
  overwhelming the server; Pushgateway is only for short-lived batch jobs.
* *"Why provision dashboards as code?"* — They are reviewable, versioned and reproducible; UI
  edits are lost and undocumented.
* *"What's missing from this build?"* — Alerting and routing, which is the next piece of work.

**CV bullet:** *Built the organisation's metrics platform — Prometheus 3.5 with relabelling,
recording rules and retention governance, node/cAdvisor exporters, RED-instrumented application
code, and Grafana dashboards provisioned as code — giving the team golden-signal visibility and
capacity trending where previously outages were reported by customers.*
