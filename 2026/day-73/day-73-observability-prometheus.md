# Introduction to Observability and Prometheus
> Infrastructure is built. Servers are configured. Containers are running. Now - how do we know it's all healthy? And when something breaks at 3 AM, how do we find out why?
 
That's the problem observability solves. This guide covers the three pillars of observability - metrics, logs, and traces - and walks through setting up Prometheus, the most widely used metrics collection tool in the DevOps ecosystem.
 
## Table of Contents
1. [Understanding Observability](#1-understand-observability)
2. [Set Up Prometheus with Docker](#2-set-up-prometheus-with-docker)
3. [Core Prometheus Concepts](#3-understand-prometheus-concepts)
4. [PromQL Basics](#4-learn-promql-basics)
5. [Add a Sample Application as a Scrape Target](#5-add-a-sample-application-as-a-scrape-target)
6. [Data Retention and Storage](#6-explore-data-retention-and-storage)

## 1. Understand Observability
Research and write short notes on:

### What is observability? How is it different from traditional monitoring?
**Ans.** These terms are often used interchangeably, but they answer different questions:
| | Monitoring | Observability |
|---|---|---|
| **Question** | *When* is something wrong? | *Why* is something wrong? |
| **Approach** | Alerts and thresholds | Exploration, querying, correlation |
| **Example** | "Error rate exceeded 5%" | "The payment service timed out calling the DB" |

### The Three Pillars of Observability:
**Metrics** - Numerical measurements over time: CPU usage, request count, error rate.<br>
**Tools**: Prometheus, Datadog, CloudWatch
 
**Logs** - Timestamped text records of events: application output, error messages, stack traces.<br>
**Tools**: Loki, ELK Stack, Fluentd
 
**Traces** - The journey of a single request across multiple services.<br>
**Tools**: OpenTelemetry, Jaeger, Zipkin

### Why do DevOps engineers need all three?
**Ans.** Each pillar answers a different part of the same incident:
- **Metrics** tells *what* is broken → high error rate on `/api/users`
- **Logs** tells *why* it broke → stack trace showing a database timeout
- **Traces** tells *where* it broke → the payment service call took 12 seconds


### The Architecture We're Building
 Over the next five days, this is the full observability stack:
```
[Your App]  ──── metrics ──► [Prometheus]       ──► [Grafana Dashboards]
[Your App]  ──── logs    ──► [Promtail]         ──► [Loki] ──► [Grafana]
[Your App]  ──── traces  ──► [OTEL Collector]   ──► [Grafana / Debug]
[Host]      ──── metrics ──► [Node Exporter]    ──► [Prometheus]
[Docker]    ──── metrics ──► [cAdvisor]         ──► [Prometheus]
```

## 2. Set Up Prometheus with Docker
Create a project directory for this entire observability block - we will keep adding to it over the next 5 days.

### Create Project Directory
We’ll keep all observability files in one place for the next 5 days.
```bash
mkdir observability-stack && cd observability-stack
```

### Create Prometheus Configuration
Prometheus needs a config file (`prometheus.yml`) to know what endpoints to scrape.
`prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```
| Field | Purpose |
|---|---|
| `scrape_interval` | How often Prometheus pulls metrics from targets (15s) |
| `evaluation_interval` | How often Prometheus evaluates alerting rules (15s) |
| `job_name` | Logical name for a group of targets |
| `targets` | Endpoints Prometheus scrapes (here, itself at `localhost:9090`) |

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4409).png)

### Create Docker Compose File
Now let’s define how Prometheus runs inside Docker (`docker-compose.yml`).
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

volumes:
  prometheus_data:
```
- **image** → official Prometheus Docker image.
- **ports** → maps container port 9090 to host port 9090.
- **volumes** → mounts config + persistent data storage.
- **restart: unless-stopped** → ensures Prometheus auto-restarts if it crashes.

The named volume `prometheus_data` persists metric history across container restarts. Without it, all historical data is lost whenever the container is recreated.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4411).png)

### Start Prometheus
```bash
docker compose up -d
```
- `-d` → detached mode (runs in background).
- Docker will pull the Prometheus image if not already present.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4413).png)

Verify the container is running:
```bash
docker ps
# Expected: container named 'prometheus' with port 9090 mapped
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4416).png)

### Access Prometheus UI
Open our browser:
```Code
http://localhost:9090
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4420).png)

We should see the **Prometheus web UI**.
- Navigate to **Status > Targets**.
- We should see **one target (prometheus)**.
- State should be **UP** (green).

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4424).png)

## 3. Understand Prometheus Concepts
### How Prometheus Works
Prometheus uses a **pull-based model** - it reaches out to scrape targets at regular intervals, rather than waiting for targets to push data to it. Each scraped endpoint is called a **scrape target**.
 
### Metric Types
| Type | Behaviour | Example |
|---|---|---|
| **Counter** | Only goes up (total requests served, total errors) | `http_requests_total`, `errors_total` |
| **Gauge** | Increases and decreases (current CPU usage, memory in use, active connections) | `memory_usage_bytes`, `active_connections` |
| **Histogram** | Distribution of values in configurable buckets | Request duration bucketed as <100ms, <500ms, <1s |
| **Summary** | Similar to histogram, but percentiles calculated client-side | p50, p95, p99 latency |

### Labels and Time Series 
**Labels** are key-value pairs that add dimensions to a metric:
```
http_requests_total{method="GET", status="200", handler="/api/users"}
```
A **time series** is a unique combination of a metric name and a set of labels. Labels let us slice and filter data without creating separate metrics.

### Open Prometheus Graph UI
- Go to: http://localhost:9090/graph
- This is the query interface where we can run PromQL queries and visualize results.

We should see a text box for queries and options to view results as a graph or table.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4427).png)

### Run Queries in Prometheus
Run these queries one by one in the Graph UI:

```
# How many metrics is Prometheus collecting about itself?
count({__name__=~".+"})
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4430).png)

```
# How much memory is Prometheus using?
process_resident_memory_bytes
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4434).png)

```
# Total HTTP requests to the Prometheus server
prometheus_http_requests_total
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4437).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4438).png)

```
# Break it down by handler
prometheus_http_requests_total{handler="/api/v1/query"}
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4440).png)

We should see numeric results in the **Console tab** and graphs in the **Graph tab**.

### What is the difference between a counter and a gauge? Give one real-world example of each.
**Counter** → only increases.
- Example: `http_requests_total` → counts total requests served.
- Real-world analogy: Odometer in a car (always goes up).

**Gauge** → goes up and down.
- Example: `process_resident_memory_bytes` → memory usage fluctuates.
- Real-world analogy: Speedometer in a car (can increase or decrease).

## 4. Learn PromQL Basics
PromQL (Prometheus Query Language) is how we interrogate our metrics data. All queries run from `http://localhost:9090/graph`.

### Open the Graph UI
- Go to http://localhost:9090/graph.
- This is where we’ll type PromQL queries and see results as **Console output** or **Graphs**.

We should see a text box at the top and options for Graph and Console.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4444).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4447).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4450).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4453).png)

### Run Basic Queries
#### 1. Instant Vector
Returns the current value for each matching time series:
```promql
up
```
- Returns `1` if the target is healthy, `0` if down. We'll see one result per scrape target.
- Our Prometheus target should show `1`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4458).png)

#### 2. Range Vector
Returns all values collected over a time window:
 ```promql
prometheus_http_requests_total[5m]
```
- Useful for inspecting raw samples and feeding into functions like `rate()`.
- We should see multiple values plotted over time.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4462).png)
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4464).png)

#### 3. Rate Function
Converts a monotonically increasing counter into a per-second rate:
```promql
rate(prometheus_http_requests_total[5m])
```
- Because counters only go up, raw counter values are rarely useful on their own. `rate()` makes them meaningful by showing how fast the counter is increasing.
- We’ll see a smooth line showing requests per second.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4466).png)
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4469).png)

#### 4. Aggregation
```promql
sum(rate(prometheus_http_requests_total[5m]))
```
- Sum across all label combinations to get a single aggregate value.
- Useful for overall traffic.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4471).png)

We’ll see a single value representing total request rate.

#### 5. Label Filtering
```promql
prometheus_http_requests_total{code="200"}
prometheus_http_requests_total{code!="200"}
```
- First query → only 200 OK requests.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4475).png)
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4478).png)

- Second query → everything except 200.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4480).png)

We’ll see filtered results by HTTP status code.

#### 6. Arithmetic
```promql
process_resident_memory_bytes / 1024 / 1024
```
- Converts bytes → MB.
- Handy for readability.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4485).png)

We’ll see memory usage in MB.

#### 7. Top-K
```promql
topk(5, prometheus_http_requests_total)
```
- Shows top 5 metrics by value.
- Great for spotting heavy hitters.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4489).png)

We’ll see the top 5 request handlers.

### Write a PromQL query that shows the per-second rate of non-200 HTTP requests to Prometheus over the last 5 minutes.
```promql
rate(prometheus_http_requests_total{code!="200"}[5m])
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4491).png)

This tells us how quickly errors (non-200s) are accumulating - a key signal for alerting.

## 5. Add Node Exporter as a Scrape Target
 
Prometheus becomes useful when it monitors something beyond itself. Add **Node Exporter**, which exposes host metrics (CPU, memory, disk, network) in Prometheus format.
 
> **Note:** We first tried `trainwithshubham/notes-app` as the target, but it is a Django + React app with no `/metrics` route. Prometheus marks it **DOWN** with `404 Not Found`. A target must expose metrics in Prometheus format, so we use Node Exporter instead.

### Update `docker-compose.yml`
Add `node-exporter` alongside Prometheus:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus_data:
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4505).png)

### Update `prometheus.yml`
Tell Prometheus to scrape Node Exporter:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4508).png)

Notice that the Node Exporter target uses the container name (`node-exporter:9100`) rather than `localhost`. Inside Docker's network, containers resolve each other by name.

### Restart the Stack
Apply changes by restarting:
```bash
docker compose up -d
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4515).png)

Why the extra restart? `docker compose up -d` only recreates containers whose Compose definition changed. Editing `prometheus.yml` does not change the Compose definition, and Prometheus reads that file only at startup, so it must be restarted to see the new job.
 
Run `docker ps`. We should see both `prometheus` and `node-exporter` running.

### Verify in Prometheus UI
- Open http://localhost:9090 **→ Status > Targets**.
- We should now see **two targets**:
    - `prometheus` (scraping itself).
    - `node-exporter` (scraping the host metrics).

- Both should show **UP**. Both targets are green and healthy.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4518).png)

### Check the Metrics
 
Confirm Node Exporter is serving metrics:
 
```bash
curl -s localhost:9100/metrics | head -n 10
```
 
Then try these queries in the Graph UI (`http://localhost:9090/graph`):
 
```
# Is each target healthy? (1 = up, 0 = down)
up
 
# Available memory in MB
node_memory_MemAvailable_bytes / 1024 / 1024
 
# CPU usage in percent, averaged over 5 minutes
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
 
> **Note:** Not all applications expose Prometheus metrics natively. Exporters such as Node Exporter and cAdvisor, and the OpenTelemetry Collector, translate metrics from systems that don't have built-in Prometheus support.
 
### Troubleshooting
 
| Symptom | Likely cause | Fix |
|---|---|---|
| Target missing after editing `prometheus.yml` | Prometheus not restarted | `docker compose restart prometheus` |
| Target DOWN, `404 Not Found` | App has no `/metrics` endpoint | Use an exporter or an instrumented app |
| Target DOWN, `connection refused` | Wrong port or container not running | Check `docker compose ps` and the port in `prometheus.yml` |
| `mapping key "command" already defined` | Duplicate `command:` key in the compose file | Keep one `command:` list |
 
---

## 6. Explore Data Retention and Storage
Understand how Prometheus stores data:
### Check Disk Usage
Prometheus stores metrics in its **local time‑series database (TSDB)** inside the container at `/prometheus`.

Run:
```bash
docker exec prometheus du -sh /prometheus
```
- `du -sh` → shows disk usage in human‑readable format.
- This tells us how much space Prometheus is currently using for its TSDB. The size depends on how long Prometheus has been running and how many series it scrapes. A fresh setup shows only a few MB (for example `1.1M /prometheus`); a longer-running one may show tens of MB.

We should see output like `50M /prometheus`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4531).png)

### Default Retention
By default, Prometheus retains data for **15 days**, after which older blocks are automatically deleted. This prevents unbounded disk growth but means historical analysis is limited to a two-week window.

### Configure Custom Retention
We can override defaults in `docker-compose.yml` under the Prometheus service:
```yaml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
```
- `--storage.tsdb.retention.time=30d` → keep data for 30 days.
- `--storage.tsdb.retention.size=1GB` → cap TSDB size at 1GB regardless of time.
- We can use either **time** or **size**, or both. If both are set, whichever limit is reached first triggers deletion of the oldest blocks.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4539).png)

### Apply and Verify
```bash
# Apply (changing command: recreates the prometheus container)
docker compose up -d

#Confirm the flags were applied
docker compose logs prometheus | grep -i retention
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4547).png)

We should see `Container prometheus  Recreated` / `Started`. Our data is kept because it lives in the `prometheus_data` volume.

### Check Retention in the UI
 
Open `http://localhost:9090` and go to **Status > Runtime & Build Information**. Find **Storage retention**; it should read `30d or 1GiB`.
 
**Status > TSDB Status** shows the number of series, label and metric cardinality, and block statistics, but not the retention settings.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/13df300e6657d446030231a3815d9aa60bef1978/2026/day-73/Screenshots/Screenshot%20(4551).png)

We can confirm Prometheus is applying our retention settings.

### Persistence Test (optional)
 
```bash
docker compose down        # removes containers, keeps the volume
docker compose up -d
docker exec prometheus du -sh /prometheus
```
 
The size should be about the same, and earlier data is still queryable.
 
> **Warning:** Never run `docker compose down -v` unless we want to delete the volume. It wipes all stored metrics.

### What happens when retention is exceeded?
- Prometheus automatically deletes the oldest data blocks.
- Only recent data (within retention window) is available for queries.
- This prevents uncontrolled disk growth.

### Why is a volume mount important for Prometheus data?
- Without a volume, TSDB data lives inside the container filesystem.
- If the container restarts or is removed, **all metrics history is lost**.
- With a mounted volume (`prometheus_data:/prometheus`), data persists across restarts.

The `prometheus_data:/prometheus` volume mount in `docker-compose.yml` is what makes Prometheus production-safe - without it, every restart is a blank slate.