---
title: "Ray Observability: How Ray, Prometheus & Grafana Fit Together"
date: 2026-02-19
draft: false
summary: "Quick notes on how Ray exports metrics, how Prometheus scrapes them, and where things break on shared clusters."
---

> These notes are based on **Ray 2.52.1**. Code links point to that tag. Things may have changed in newer versions.

**Contents:**

1. [The 3 Pieces (TLDR)](#the-3-pieces-tldr)
    1. [Ray: Metrics Export](#ray-metrics-export)
    2. [Prometheus](#prometheus-service-discovery)
        1. [Service Discovery](#prometheus-service-discovery)
        2. [Configuration](#prometheus-configuration)
    3. [Grafana: Dashboards](#grafana-dashboards)
2. [Key Files & Ports](#the-key-files--ports)
3. [Where This Breaks: Shared Clusters](#where-this-breaks-shared--multi-user-clusters)
4. [A Fix: Decouple Prometheus from Ray's Temp Dir in NeMo Curator](#a-fix-decouple-prometheus-from-rays-temp-dir-in-nemo-curator)

---

## The 3 Pieces (TLDR)

**Ray** exports metrics. **Prometheus** scrapes and stores them. **Grafana** visualizes them.

That's it. The plumbing between them:

1. Ray Head Node writes `PROMETHEUS_SERVICE_DISCOVERY_FILE` — a list of all nodes' metrics endpoints.
2. `prometheus.yml` references this file so Prometheus knows where to scrape.
3. Grafana uses Prometheus as a data source and displays the metrics.

```mermaid
flowchart LR
    RC["Ray Cluster (head node)"] -->|"writes every 5s"| SD["PROMETHEUS_SERVICE_DISCOVERY_FILE<br>.json"] <-->|"referenced in"| YML["prometheus.yml"]
    YML -->|"configures"| P["Prometheus :9090"]
    P -->|"data source"| G["Grafana :3000"]
```

---

### Ray: Metrics Export

Every Ray node runs a **metrics agent** (part of the dashboard agent) that serves metrics over HTTP in Prometheus format.

- The port is `metrics_export_port` (default `8080`)
- One agent per node, one port per agent
- Ray components (raylet, workers, GCS) push metrics to this agent via gRPC
- The agent translates OpenCensus/OpenTelemetry data into Prometheus gauges, counters, and histograms

4 nodes = 4 HTTP endpoints serving metrics.

<details open>
<summary>Code</summary>

```python
# https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/reporter/reporter_agent.py#L448-L465
stats_exporter = prometheus_exporter.new_stats_exporter(
    prometheus_exporter.Options(
        namespace="ray",
        port=dashboard_agent.metrics_export_port,
        address="127.0.0.1" if self._ip == "127.0.0.1" else "",
    )
)
```

- [`MetricsAgent`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L628) — records and proxies metrics
- [`ReporterAgent`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/reporter/reporter_agent.py#L448-L465) — initializes the Prometheus exporter and MetricsAgent
- [`start_raylet`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/services.py#L1767) — passes `--metrics-export-port` to the dashboard agent

</details>

---

### Prometheus: Service Discovery

Prometheus needs to know _where_ to scrape. Ray handles this via **file-based service discovery**.

The **head node** runs a [`PrometheusServiceDiscoveryWriter`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L757) thread that:

1. Calls `ray.nodes()` to get all alive nodes
2. Collects each node's `MetricsExportPort`
3. Writes the list to a JSON file

**The file:** `{RAY_TEMP_DIR}/prom_metrics_service_discovery.json`

Default: `/tmp/ray/prom_metrics_service_discovery.json` ([constant](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/ray_constants.py#L172))

**Format:**

```json
[{
  "labels": {"job": "ray"},
  "targets": ["10.0.0.1:8080", "10.0.0.2:8080", "10.0.0.3:8080"]
}]
```

Rewritten [**every 5 seconds**](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L777). New nodes appear automatically, dead nodes get dropped.

<details open>
<summary>Code</summary>

```python
# https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L792-L808
nodes = ray.nodes()
metrics_export_addresses = [
    build_address(node["NodeManagerAddress"], node["MetricsExportPort"])
    for node in nodes
    if node["alive"] is True
]
# some code skipped
content = [{"labels": {"job": "ray"}, "targets": metrics_export_addresses}]
with self._content_lock:
    self.latest_service_discovery_content = content
return json.dumps(content)
```

- [`get_file_discovery_content`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L790) — builds the targets list from alive nodes
- [`write`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/_private/metrics_agent.py#L810) — atomic write to disk
- [`ReportHead`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/reporter/reporter_head.py#L70) — instantiates the writer and [starts it as a daemon thread](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/reporter/reporter_head.py#L913-L914)

</details>

---

### Prometheus: Configuration

Prometheus needs a `prometheus.yml` that defines:

- **Where** to find the discovery file
- **How often** to scrape (default: every 10s)

```yaml
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: 'ray'
    file_sd_configs:
      - files:
          - '/tmp/ray/prom_metrics_service_discovery.json'
```

Ray generates this from a [template](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/metrics/templates.py#L49-L63) and writes it to `{SESSION_DIR}/metrics/prometheus/prometheus.yml` in [`_create_default_prometheus_configs`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/metrics/metrics_head.py#L379-L406).

Prometheus watches the discovery file for changes — no restart needed when nodes join or leave.

**Watch out:** Ray also ships a [static `prometheus.yml`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/metrics/export/prometheus/prometheus.yml#L11) that hardcodes `/tmp/ray/prom_metrics_service_discovery.json`. If you use a custom temp dir, make sure you're using the _generated_ config, not the static one.

---

### Grafana: Dashboards

Grafana points to Prometheus as a data source (`http://localhost:9090`) and loads pre-built dashboards.

Ray bundles several dashboard generators:

- **Default** — cluster-wide metrics (CPU, memory, object store, etc.)
- **Data** — Ray Data specific metrics
- **Serve** — Ray Serve metrics
- **Serve Deployment** — per-deployment metrics

These are JSON files that Grafana auto-provisions from a dashboards directory.

---

## The Key Files & Ports

```text
Port / File                                          Who uses it
─────────────────────────────────────────────────────────────────
metrics_export_port (default 8080)                   Ray → Prometheus
  Each node's HTTP endpoint serving metrics

prom_metrics_service_discovery.json                  Ray → Prometheus
  {RAY_TEMP_DIR}/prom_metrics_service_discovery.json
  List of all nodes' metrics endpoints, updated every 5s

prometheus.yml                                       Prometheus
  {SESSION_DIR}/metrics/prometheus/prometheus.yml
  Tells Prometheus where the discovery file is

Prometheus web port (default 9090)                   Prometheus → Grafana
  Where Prometheus serves its query API

Grafana web port (default 3000)                      Grafana → You
  Where you see the dashboards
```

---

## Where This Breaks: Shared / Multi-User Clusters

The whole setup assumes **one head node, one temp dir, one Prometheus instance**. That falls apart on shared machines.

### Problem 1: Temp Dir Collision

Default `RAY_TEMP_DIR` is `/tmp/ray`. Multiple users starting Ray clusters on the same machine means every head node writes to the _same_ `prom_metrics_service_discovery.json` every 5 seconds. Last writer wins. Metrics endpoints keep appearing and disappearing.

### Problem 2: Custom Temp Dir Breaks Prometheus

Setting `--temp-dir /tmp/ray_alice` makes Ray write the discovery file to `/tmp/ray_alice/prom_metrics_service_discovery.json`.

But the [**static** `prometheus.yml`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/metrics/export/prometheus/prometheus.yml#L11) that ships with Ray hardcodes:

```yaml
- files:
    - '/tmp/ray/prom_metrics_service_discovery.json'
```

Prometheus reads from `/tmp/ray/`, Ray writes to `/tmp/ray_alice/`. Nothing matches. No metrics.

Ray _does_ generate a [dynamic `prometheus.yml`](https://github.com/ray-project/ray/blob/ray-2.52.1/python/ray/dashboard/modules/metrics/metrics_head.py#L379-L406) with the correct temp dir at `{session_dir}/metrics/prometheus/prometheus.yml` — but you need to know to point Prometheus at that one instead.

### Problem 3: Port Conflicts

Default Prometheus port is `9090`, Grafana is `3000`. Two users starting monitoring on the same machine will collide.

## A Fix: Decouple Prometheus from Ray's Temp Dir in NeMo Curator

The approach we take in [NeMo Curator #1523](https://github.com/NVIDIA/NeMo-Curator/pull/1523):

1. Start Prometheus with an **empty** `file_sd_configs: []` and `--web.enable-lifecycle`
2. When a Ray cluster starts, **dynamically append** its discovery file path to the Prometheus config
3. Hot-reload Prometheus via `POST /-/reload`
4. On shutdown, remove the entry and reload again

This avoids hardcoded paths — Prometheus always points at the right discovery file regardless of the temp dir.
