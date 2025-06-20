## Setup Instructions

This project can be implemented on any Linux-based Server or local machine. However, this instruction assumes that the server is an AWS EC2 instance running on ubuntu 24.04

### Prerequisite

i) AWS EC2 instance running on ubuntu with git and docker installed.
ii) Basic knowledge of docker, linux cli, prometheus, and grafana

### Sever Setup (Containerized Prometheus, Grafana, and Node Exporter)

- ssh into your EC2 instance
- run ```git clone https://github.com/OdarteiL/prometheus_grafana.git```
- run ```sudo docker compose up -d```
- Verify that your containers are running by ```sudo docker compose ps``` You will see a results similar to this:

```bash
NAME            IMAGE                        COMMAND                  SERVICE         CREATED       STATUS       PORTS
grafana         grafana/grafana-oss:latest   "/run.sh"                grafana         2 hours ago   Up 2 hours   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
node-exporter   prom/node-exporter:latest    "/bin/node_exporter …"   node-exporter   2 hours ago   Up 2 hours   0.0.0.0:9100->9100/tcp, [::]:9100->9100/tcp
prometheus      prom/prometheus:latest       "/bin/prometheus --c…"   prometheus      2 hours ago   Up 2 hours   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
```

- Open the Prometheus Client in your browser ```http://your-instance-ip:9090/targets``` It should look similar to this image ![Prometheus Target Page]
- Open the Grafana Dashboard in your browser ```http://your-instance-ip:3000``` It should look similar to this image ![Grafana Dashboard]

  - Enter ```admin``` as both the username and password.mv ./alert_notification.png ./Screenshots/

  - Update your password when prompted.

### Building the Dashboards

[Follow this blog](https://betterstack.com/community/guides/monitoring/visualize-prometheus-metrics-grafana/#configuring-the-prometheus-data-source) to for a step-by-step guide on building the dashboard.





# Section 1: Architecture and Setup Understanding

## Question 1: Container Orchestration Analysis

Without those mounts, Node Exporter won’t be able to access any real system metrics. It would just see what's inside the container, which totally misses the point of monitoring the host. From my hands-on setup:

* `/proc` gives stats about the system like CPU, memory, and processes
* `/sys` exposes hardware and kernel module details
* `/` (root) helps track disk usage
  Mounting these is what lets the exporter "see" the actual system, not just the container's world.

## Question 2: Network Security Implications

Creating a custom network like `prometheus-grafana` improves container isolation and lets services talk using names instead of IPs. But here’s what I noticed during the lab:

* Exposing ports 9090, 9100, and 3000 without protection is risky
* Prometheus and Node Exporter have no auth, and Grafana might still use default creds
* No HTTPS or TLS by default
  In production, I’d only expose Grafana (via HTTPS), lock everything else down to internal access, and use a reverse proxy with basic auth. Also, regenerate passwords and add TLS certs.

## Question 3: Data Persistence Strategy

Prometheus uses a volume for `/prometheus`, and Grafana for `/var/lib/grafana`:

* Prometheus stores lots of time-series data
* Grafana keeps dashboards, user prefs, etc.
  Both need persistence, but for slightly different reasons. Prometheus can be rebuilt, but losing Grafana dashboards would be a pain. Without volumes:
* You’d lose all metric history on restart
* Dashboards would disappear
* You’d have to reconfigure users and settings every time
  Definitely not production-safe.

# Section 2: Metrics and Query Understanding

## Question 4: PromQL Logic Breakdown

The query `node_time_seconds - node_boot_time_seconds` gives uptime.

* `node_time_seconds` is the current time
* `node_boot_time_seconds` is when the system started
* Subtracting them shows how long the system has been up
  Possible issues:
* System clock issues (like NTP drift)
* Leap seconds
* If the target is down, the query may still give a stale value
  An alternative I tried was using `up * on() (time() - node_boot_time_seconds)` to ensure the target is up first.

## Question 5: Memory Metrics Deep Dive

Using `MemTotal - MemAvailable` is much more accurate than using `MemFree`.

* `MemFree` only counts totally unused memory
* `MemAvailable` includes reclaimable memory (like cache)
  Linux likes to use free memory for caching, so `MemFree` can look low even on a healthy system. I actually saw this in action—my dashboard kept alerting "low memory" until I switched to `MemAvailable`, which fixed the false alerts.

## Question 6: Filesystem Query Analysis

Query:

```promql
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

This gives disk **used** space as a ratio. Divide to get the "free" ratio, subtract from 1 to get "used." Makes it easier to trigger alerts like "90% full."
Problems:

* Only works for root `/`
* If you include tmpfs or overlay filesystems, you'll get misleading results
  Improved version:

```promql
1 - (
  node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|overlay"} /
  node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs|overlay"}
)
```

This filters out unwanted filesystems.

# Section 3: Visualization and Dashboard Design

## Question 7: Visualization Type Justification

Each panel type was chosen for a reason:

* **Stat**: Simple values like "Current CPU Usage"
* **Time Series**: Perfect for trend analysis (e.g., CPU load over time)
* **Gauge**: Shows how close you are to a limit (e.g., disk usage)
  From my perspective, Stat is best for quick-glance data, Time Series for spotting spikes, and Gauge for thresholds. Wrong choices here can make a dashboard confusing.

## Question 8: Threshold Configuration Strategy

The lab used 80% as a static warning threshold for disk usage, but that doesn’t fit all systems. Better approach:

* **DB servers**: Warn at 70%
* **Web servers**: Maybe 85% is okay
* Use multi-level thresholds:

  * Warning: 70%
  * Critical: 85%
  * Emergency: 95%
    Also, monitor how fast the disk fills up, not just the current level.

## Question 9: Dashboard Variable Implementation

`$job` is a Grafana variable that dynamically updates queries. Here's what I found:

* It populates dropdowns based on label values
* When multiple are selected, Grafana turns it into a regex (like `job=~"(job1|job2)"`)
  Issue: If someone names a job `.*`, it could match everything.
  Fix: Use `job=~"^$job$"` to avoid regex injection.
  I tested variables by selecting edge cases, empty values, and long lists to make sure nothing breaks.

# Section 4: Production and Scalability Considerations

## Question 10: Resource Planning and Scaling

Monitoring 100 servers will need more resources:

* CPU: \~4 cores
* RAM: \~16GB
* Storage: \~500GB (with 15s scrape interval and 30-day retention)
  Problems you’ll hit first:
* Memory (for active series)
* Disk I/O (Prometheus writes constantly)
  Optimizations:
* Use SSDs
* Add recording rules for heavy queries

## Question 11: High Availability Design

Current setup is single-node, which means one failure = everything down. My HA plan:

* Multiple Prometheus nodes (scraping same targets)
* Shared storage or remote write to Thanos
* Grafana in HA mode with a shared database
* Load balancer in front
  Trade-off: more complexity, but way more reliable. Worth it if you're monitoring prod systems.

## Question 12: Security Hardening Analysis

Issues I spotted:

1. No auth on Prometheus
2. Grafana using default credentials
3. No encryption
4. Containers exposed on public ports
5. Containers running as root
   My fixes:

* Add TLS everywhere
* Set up real users and passwords
* Use Docker secrets or environment variables for creds
* Run containers as non-root
* Segment the network so only trusted services can connect

# Section 5: Troubleshooting and Operations

## Question 13: Debugging Methodology

When a target shows "DOWN":

1. **Check network**: Can Prometheus reach the target?

   ```bash
   curl http://<target-ip>:9100/metrics
   ```
2. **Check config**:

   ```bash
   docker exec -it prometheus promtool check config /etc/prometheus/prometheus.yml
   ```
3. **Logs**:

   ```bash
   docker logs prometheus
   docker logs node-exporter
   ```
4. **Firewalls**: Make sure ports are open
   Most common issues: target not running, port blocked, or config typo.

## Question 14: Performance Optimization

Heavy queries:

```promql
rate(node_cpu_seconds_total[5m])
```

Better:

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

Improvements I used:

* Recording rules for heavy queries
* Limit dashboards to shorter time ranges
* Reduce refresh interval
  Also monitored Prometheus itself using:
* `prometheus_tsdb_head_series`
* `prometheus_http_request_duration_seconds`

## Question 15: Capacity Planning Scenario

Prometheus storage grows fast. Causes:

* More targets = more series
* Shorter scrape intervals
* Longer retention
  Storage formula:

```
daily_usage = series_count * samples/day * bytes/sample
```

For long-term strategy:

* Keep 7d hot data on SSD
* 30d on standard disk
* Archive older data to S3 or similar
* Use downsampling and remote write for cost savings

# Lessons Learned

* Monitoring isn't just "plug and play"
* It's easy to misinterpret metrics if you don’t understand Linux internals
* False alerts are worse than no alerts

