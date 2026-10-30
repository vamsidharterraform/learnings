# 🔭 Real-World Observability on AWS EKS: Which Tool Does What

> **Interview question:** *"What observability tools do you use, and what do you use each one for?"*

---

## 🎯 1. The Big Picture (start your answer here)

Observability rests on **three pillars**. No single tool covers all of them well, so you pick the right tool per layer.

| Pillar | Question it answers | Typical tools |
|---|---|---|
| 📈 **Metrics** | *"Is something wrong, and how bad?"* | CloudWatch, Prometheus, Grafana |
| 📜 **Logs** | *"What exactly happened?"* | Splunk, CloudWatch Logs, Loki / OpenSearch |
| 🧵 **Traces / APM** | *"Where is the time being spent?"* | AppDynamics, X-Ray, OpenTelemetry, Jaeger / Tempo |

📌 **One-line summary:**
**Metrics tell you *something is wrong*, logs tell you *what happened*, and traces tell you *where*.**

---

## 🗺️ 2. Tool Map: Which Layer, Which Tool

| Layer | What you watch | Best tool |
|---|---|---|
| 🌐 **Load balancer** | Latency, 5xx, healthy targets | **CloudWatch** |
| ☁️ **AWS managed services** (RDS, EC2, EBS, SQS, etc.) | Service health, saturation, errors | **CloudWatch** |
| ☸️ **EKS control plane** | API server health, audit trail, auth failures | **CloudWatch Logs** (+ Prometheus for API server metrics) |
| 🖥️ **Nodes** | CPU, memory, disk, network | **Prometheus** (node-exporter) / CloudWatch Agent |
| 📦 **Pods / containers** | Resource use, restarts, OOMKilled, pending | **Prometheus** + **Grafana** |
| ⚙️ **Application metrics** | Request rate, error rate, latency | **Prometheus** + **Grafana** |
| 📜 **Application logs** | Exceptions, timeouts, stack traces | **Splunk** |
| 🔬 **Code / transaction level** | Slow methods, slow SQL, downstream calls | **AppDynamics** |
| 🚨 **Alerting & paging** | Notify the right person | Alertmanager / CloudWatch Alarms → PagerDuty / Opsgenie / SNS |

---

## ☁️ 3. Amazon CloudWatch: *"The AWS-native layer"*

**Use it when the thing you're monitoring is an AWS service.** It needs no setup for most AWS services, because metrics arrive automatically.

### 3.1 ALB (Application Load Balancer): the first place to look for user impact

| Metric | What it tells you | 🚨 Watch for |
|---|---|---|
| **`TargetResponseTime`** | Time your **backend** took to respond (seconds). Use **p95 / p99**, not just average | Sudden rise → slow app or DB |
| **`HTTPCode_Target_5XX_Count`** | 5xx returned **by your app/pods** | Spike → application errors |
| **`HTTPCode_ELB_5XX_Count`** | 5xx generated **by the ALB itself** (502, 503, 504) | Spike → no healthy targets, timeouts, capacity |
| **`HTTPCode_Target_4XX_Count`** | Client-side errors | Spike → bad deploy, auth issue, bot traffic |
| **`RequestCount`** | Traffic volume | Drop → upstream issue. Spike → load. |
| **`HealthyHostCount` / `UnHealthyHostCount`** | Targets passing health checks | Unhealthy > 0 for long |
| **`TargetConnectionErrorCount`** | ALB couldn't connect to the target | Pods down, SG or port mismatch |
| **`RejectedConnectionCount`** | ALB hit its connection limit | Capacity problem |
| **`ActiveConnectionCount`** | Concurrent connections | Sudden surge |

💡 **Senior tip:** Distinguish **`ELB_5XX`** (the load balancer's fault) from **`Target_5XX`** (your app's fault). It tells you *where* to look in seconds.

### 3.2 EKS control plane: what CloudWatch gives you

The control plane is **managed by AWS**, so you can't SSH into it. You observe it through **logs** and **API server metrics**.

| Signal | Where | What you learn |
|---|---|---|
| **Control plane logs**: `api`, `audit`, `authenticator`, `controllerManager`, `scheduler` | **CloudWatch Logs** (`/aws/eks/<cluster>/cluster`). Must be **enabled** per cluster. | Who did what (audit), auth failures, scheduling decisions, API errors |
| **API server metrics** (`apiserver_request_total`, `apiserver_request_duration_seconds`) | Exposed on the API server `/metrics` endpoint, scraped by **Prometheus** or Amazon Managed Prometheus | API latency, error rate, throttling |
| **Container Insights** (CloudWatch Observability add-on) | CloudWatch metrics + logs | Cluster, node, pod CPU/memory, restarts, performance logs |

**What control plane logs help you answer:**
- 🔐 *"Who deleted that deployment?"* → **audit** log
- 🚫 *"Why can't this role authenticate?"* → **authenticator** log
- 📅 *"Why wasn't my pod scheduled?"* → **scheduler** log
- 🐢 *"Why is `kubectl` slow or failing?"* → **api** log + API latency metrics

```text
# CloudWatch Logs Insights: who deleted something in the cluster
fields @timestamp, user.username, verb, objectRef.resource, objectRef.name
| filter verb = "delete"
| sort @timestamp desc
| limit 50
```

> ⚠️ **Cost note:** audit logs are **verbose**. Enable only the log types you need, and set a **retention period** on the log group.

> 📝 **Check the current AWS docs** for any control plane metrics published natively to CloudWatch, since AWS keeps expanding what's available.

### 3.3 RDS: database health

| Metric | Meaning | 🚨 Red flag |
|---|---|---|
| `CPUUtilization` | DB compute load | Sustained high → slow queries / undersized |
| `DatabaseConnections` | Active connections | Near `max_connections` |
| `FreeableMemory` | RAM available | Dropping → memory pressure / swapping |
| `FreeStorageSpace` | Disk left | Approaching 0 → outage risk |
| `ReadLatency` / `WriteLatency` | Storage latency | Rising → I/O bottleneck |
| `ReadIOPS` / `WriteIOPS`, `DiskQueueDepth` | I/O load | Queue growing → saturated |
| `ReplicaLag` | Read replica delay | Growing → stale reads |
| **Performance Insights** | Top SQL, wait events | *"Which query is slow and why?"* |

### 3.4 EC2 and EBS

| Metric | Why it matters |
|---|---|
| `CPUUtilization`, `NetworkIn/Out` | Basic load |
| `StatusCheckFailed` (system / instance) | Host vs OS failure |
| `CPUCreditBalance` (T-series) | Throttling when credits hit 0 |
| `VolumeQueueLength`, `BurstBalance` | Disk I/O saturation |
| **Memory and disk %** | **Not collected by default.** Install the **CloudWatch Agent**. |

### 3.5 CloudWatch features worth naming

| Feature | Use |
|---|---|
| **Alarms** (+ composite alarms) | Alert on thresholds; combine conditions to reduce noise |
| **Anomaly detection** | Alert on "unusual," not just fixed thresholds |
| **Logs Insights** | Query logs with a SQL-like language |
| **Metric Math** | Compute error % = `5xx / RequestCount × 100` |
| **Dashboards** | Quick service-level views |
| **Synthetics (canaries)** | Probe endpoints from outside, to catch outages before users do |
| **Application Signals / X-Ray** | AWS-native tracing and service maps |

✅ **Best for:** AWS service health, infrastructure alarms, audit/control plane logs.
⚠️ **Weak at:** deep Kubernetes/pod-level flexibility and custom app-metric querying (Prometheus is stronger).

---

## 🔥 4. Prometheus: *"The Kubernetes and application metrics engine"*

**Use it when you need flexible, queryable, high-resolution metrics from Kubernetes and your own applications.**

### 4.1 How it works

```
Targets (pods, nodes, apps)  ←──scrape (pull) every 15–30s──  Prometheus (TSDB + PromQL)
                                                                   │
                                                  rules ──────────►│──► Alertmanager ──► Slack / PagerDuty
                                                                   │
                                                                   └──► Grafana (dashboards)
```

| Component | Role |
|---|---|
| **Prometheus server** | Scrapes (**pulls**) metrics, stores in a time-series database, evaluates rules |
| **PromQL** | Query language for slicing and aggregating metrics |
| **Alertmanager** | Deduplicates, groups, routes, and silences alerts |
| **Exporters** | Expose metrics from things that don't speak Prometheus natively |
| **Service discovery** | Auto-finds pods and services in Kubernetes |

### 4.2 The exporters you should name

| Exporter / source | Gives you |
|---|---|
| **node-exporter** | Node CPU, memory, disk, network |
| **kube-state-metrics** | Object state: pod phase, restarts, deployment replicas, resource requests/limits |
| **cAdvisor** (via kubelet) | Container CPU, memory, I/O |
| **Application `/metrics`** | Business and request metrics (via client libraries / Micrometer for Java) |
| **blackbox-exporter** | Probe HTTP/TCP/DNS endpoints |
| **CloudWatch exporter / YACE** | Pull CloudWatch metrics into Prometheus |

### 4.3 What it's best at

| Use case | Example |
|---|---|
| 🧱 **Resource saturation** | Node/pod CPU, memory, disk |
| ♻️ **Pod health** | Restarts, OOMKilled, Pending, not Ready |
| ⏱️ **Latency percentiles** | `histogram_quantile` on request duration |
| ❌ **Error rate** | 5xx ratio per service |
| 📊 **Capacity trends** | Growth over weeks |
| ☸️ **API server health** | Request latency and error codes |

```promql
# p95 request latency per service (needs histogram instrumentation)
histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))

# Error ratio (%)
100 * sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))

# Kubernetes API server p99 latency (excluding long-running WATCH)
histogram_quantile(0.99,
  sum by (le, verb) (rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])))
```

### 4.4 Amazon-managed options

| Service | What it removes |
|---|---|
| **Amazon Managed Service for Prometheus (AMP)** | Running and scaling Prometheus storage and HA yourself |
| **Amazon Managed Grafana (AMG)** | Hosting and patching Grafana, with SSO integration |

### 4.5 Limitations to be honest about

- 💾 **Local storage is short-term.** For long retention and HA, use **Thanos / Mimir / AMP**.
- 📉 It's for **metrics only**, not logs or traces.
- 💥 **High-cardinality labels** (user IDs, request IDs) can overload it.
- 🔄 A single server isn't HA by default. Run replicas or use a managed option.

✅ **Best for:** Kubernetes, pod, node, and application metrics, plus alerting rules.

---

## 📊 5. Grafana: *"The single pane of glass"*

**Prometheus stores and queries. Grafana shows.**

| Capability | Detail |
|---|---|
| 🔌 **Many data sources** | Prometheus, **CloudWatch**, Loki, Elasticsearch/OpenSearch, Splunk (plugin), SQL databases |
| 📈 **Dashboards** | Combine AWS and Kubernetes views on one screen |
| 🚨 **Alerting** | Grafana-managed alerts, routed to Slack, PagerDuty, email |
| 🔗 **Drill-down** | Click from a spike to related logs or traces |
| 🧩 **Variables** | One dashboard for many clusters, namespaces, or services |
| 🗂️ **Dashboards as code** | Version dashboards in Git (JSON / Terraform / Helm) |

**Dashboards that matter in production:**

| Dashboard | Panels |
|---|---|
| **Service overview (RED)** | Rate, Errors, Duration |
| **Node health (USE)** | Utilization, Saturation, Errors |
| **Pod / namespace** | CPU vs requests, memory vs limits, restarts |
| **Cluster capacity** | Allocatable vs requested, Pending pods |
| **ALB + RDS (CloudWatch source)** | Response time, 5xx, DB connections |

✅ **Best for:** unified visualization and correlation across data sources.

---

## 📜 6. Splunk: *"Logs: the story of what happened"*

**Use it when you need to search, correlate, and alert on logs across many services.**

| Use case | Example |
|---|---|
| 🐛 **Find exceptions** | `NullPointerException`, `OutOfMemoryError`, stack traces |
| ⏱️ **Find timeouts** | `SocketTimeoutException`, connection pool exhausted |
| 🔗 **Trace one request** | Search by **correlation / request ID** across microservices |
| 📅 **Timeline of an incident** | What happened first, and what followed |
| 🔐 **Security / audit** | Failed logins, suspicious patterns |
| 🚨 **Log-based alerts** | Alert when error count crosses a threshold |

```text
# SPL example: errors by service in the last 30 minutes
index=prod_app sourcetype=java:app ("ERROR" OR "Exception") earliest=-30m
| stats count by service, exception_class
| sort - count

# Follow one request across services
index=prod_app correlation_id="abc-123"
| sort _time
| table _time service level message
```

**Getting EKS logs into Splunk:** a log collector as a DaemonSet (e.g. **Splunk OpenTelemetry Collector**, Fluent Bit, or Fluentd) ships container logs to Splunk.

💡 **Senior tip:** Logs are only useful if they're **structured (JSON)** and carry a **correlation ID** from the ALB to every downstream service.

✅ **Best for:** root cause evidence, exception analysis, cross-service request tracing.

---

## 🔬 7. AppDynamics: *"APM: where the time went"*

**Use it when you need to know whether the delay is in Java code, a SQL call, or a downstream dependency.**

| Capability | What you see |
|---|---|
| 🧭 **Business transactions** | Response time and error rate **per user action** (e.g. "Generate Report") |
| 🗺️ **Flow maps** | Service-to-service dependencies and where latency accumulates |
| 📸 **Transaction snapshots** | Call stack of a slow request, down to **method level** |
| 🗄️ **Database calls** | Slowest SQL, call counts, per-query time |
| 🌍 **External calls** | Slow third-party or downstream HTTP services |
| ☕ **JVM health** | Heap, GC pauses, thread counts, connection pools |
| 📏 **Baselines** | Auto-learned "normal" for alerting on deviation |

**Typical finding:** *"Report generation spends 17 of 20 seconds in one SQL query, called 400 times (N+1 problem)."* That's the kind of answer metrics and logs alone can't give you.

> ⚠️ Needs an **agent** on the application (e.g. Java agent), so it must be deployed and licensed. It's not automatic.

✅ **Best for:** code-level and transaction-level root cause.

---

## 🧰 8. Other Tools Worth Knowing

| Tool | Use |
|---|---|
| **OpenTelemetry (OTel)** | Vendor-neutral standard for emitting metrics, logs, **and traces** |
| **AWS X-Ray / Application Signals** | AWS-native distributed tracing and service maps |
| **Jaeger / Grafana Tempo** | Open-source tracing backends |
| **Grafana Loki / OpenSearch (ELK)** | Open-source log aggregation |
| **Datadog / New Relic / Dynatrace** | All-in-one commercial platforms |
| **AWS CloudTrail** | **Who did what in AWS** (API audit), not application observability |
| **VPC Flow Logs** | Network-level traffic accept/reject |
| **PagerDuty / Opsgenie** | On-call routing and escalation |

---

## 📐 9. Frameworks that Make You Sound Senior

| Framework | Stands for | Apply to |
|---|---|---|
| **Four Golden Signals** | **Latency, Traffic, Errors, Saturation** | Any user-facing service |
| **RED** | **Rate, Errors, Duration** | Request-driven services / microservices |
| **USE** | **Utilization, Saturation, Errors** | Resources: CPU, memory, disk, network |

| Signal | CloudWatch example | Prometheus example |
|---|---|---|
| **Latency** | ALB `TargetResponseTime` (p95/p99) | `histogram_quantile` on request duration |
| **Traffic** | ALB `RequestCount` | `rate(http_requests_total[5m])` |
| **Errors** | `HTTPCode_Target_5XX_Count` | 5xx ratio |
| **Saturation** | RDS `DatabaseConnections`, `DiskQueueDepth` | Pod CPU vs limit, node memory %, Pending pods |

---

## 🚨 10. Real-Time Production Incident Example

**Scenario:** Users report that report generation takes **20 seconds instead of 3 seconds**.

| Step | Tool | What you investigate |
|---|---|---|
| 1 | **CloudWatch** | Check ALB target response time, HTTP 5xx errors, and RDS metrics. |
| 2 | **Grafana** | Examine EKS pod CPU, memory, restarts, and request latency. |
| 3 | **Prometheus** | Use PromQL to identify high latency, resource pressure, or elevated error rates. |
| 4 | **Splunk** | Search Java logs using the request or correlation ID to find exceptions and timeouts. |
| 5 | **AppDynamics** | Identify whether the delay originates in Java code, a database call, or a downstream dependency. |

> ⚠️ **Important:** These are the investigations you would perform, not proof that a particular tool has already been configured to collect all those signals.

### 🧠 How to read the findings (decision flow)

```
        Users report slowness
                 │
                 ▼
   CloudWatch: ALB TargetResponseTime ↑ ?
        │                    │
       NO                   YES
        │                    │
        ▼                    ▼
 Problem is client/      Is the ALB 5xx / unhealthy hosts ↑ ?
 network/DNS/frontend         │
                       ┌──────┴───────┐
                      YES             NO (slow but successful)
                       │               │
                       ▼               ▼
              Pods unhealthy /     RDS slow? (CPU, connections,
              crashing / scaling   latency, Performance Insights)
              → Grafana/Prometheus        │
                                   ┌──────┴───────┐
                                  YES             NO
                                   │               │
                                   ▼               ▼
                           Slow query /      Pod CPU throttled / memory
                           lock → AppDynamics  pressure? → Prometheus
                           (SQL)                       │
                                                       ▼
                                           Logs + traces: Splunk +
                                           AppDynamics for the exact cause
```

### 🔍 Example: what each tool might reveal

| Tool | Possible finding |
|---|---|
| **CloudWatch** | `TargetResponseTime` p95 jumped from 3s to 20s. 5xx is flat, and RDS `DatabaseConnections` is near the limit. |
| **Grafana** | Backend pods at ~100% of CPU limit (throttled). No restarts. |
| **Prometheus** | `container_cpu_cfs_throttled_periods_total` is climbing, and request p95 latency is rising. |
| **Splunk** | `Timeout waiting for connection from pool` errors for that correlation ID. |
| **AppDynamics** | "Generate Report" spends most of its time waiting on a DB call. Connection pool exhausted. |
| **Conclusion** | A new release increased report size, so queries are held longer and the connection pool is exhausted. |

---

## 🎤 11. Interview Answer You Can Practice

> *"In our enterprise reporting application deployed on AWS EKS, we use different observability tools for different layers.*
>
> *We use **CloudWatch** for AWS infrastructure and service monitoring, including ALB metrics, EC2, RDS, and relevant EKS logs and metrics.*
>
> *We use **Prometheus and Grafana** for Kubernetes and application metrics, resource utilization, pod health, dashboards, and alerting.*
>
> *We use **Splunk** for centralized log analysis and troubleshooting application exceptions across our Java microservices.*
>
> *We use **AppDynamics** for application performance monitoring, transaction tracing, response-time analysis, and identifying database or code-level bottlenecks.*
>
> *For production incidents, we correlate infrastructure metrics, Kubernetes metrics, application logs, and transaction traces to isolate the root cause and reduce MTTR."*

### ⚡ 30-second version

> *"Metrics tell me something is wrong, logs tell me what happened, and traces tell me where. So I start at the ALB in CloudWatch for user impact, move to Prometheus and Grafana for pod and node health, use Splunk with the correlation ID for the evidence, and AppDynamics for the code or SQL bottleneck."*

---

## 💬 12. Be Ready for These Follow-Up Questions

**Q: Why both CloudWatch and Prometheus?**
CloudWatch integrates with AWS services; Prometheus provides flexible, queryable metrics for Kubernetes and instrumented applications.

**Q: Why both Splunk and AppDynamics?**
Splunk is primarily for log search and correlation; AppDynamics focuses on application performance and transaction diagnostics.

**Q: What is the difference between Prometheus and Grafana?**
Prometheus collects and queries time-series metrics; Grafana visualizes metrics and other supported data sources.

**Q: How do you reduce MTTR?**
Use actionable alerts, correlation IDs, dashboards, logs, traces, and runbooks to quickly narrow down the failing component.

### ➕ More follow-ups to prepare

**Q: Prometheus pulls and CloudWatch receives. What's the difference?**
Prometheus **scrapes** (pull) targets on an interval, which suits dynamic Kubernetes. CloudWatch mostly **receives** metrics pushed by AWS services or the agent.

**Q: ALB shows 502/504. Where do you look?**
`HTTPCode_ELB_5XX_Count` plus `UnHealthyHostCount` and `TargetConnectionErrorCount`. Then check pod readiness, health check path, security groups, and idle timeout vs app response time.

**Q: How do you avoid alert fatigue?**
Alert on **symptoms** (user-facing latency/errors) rather than every cause, use sensible durations (`for: 5m`), group and deduplicate in Alertmanager, add runbook links, and review noisy alerts regularly.

**Q: What's a correlation ID, and why does it matter?**
A unique ID generated at the edge (ALB / gateway) and passed through every service and log line, so one user request can be traced across Splunk logs and AppDynamics transactions.

**Q: Metrics vs logs vs traces?**
Metrics are cheap, aggregated numbers for **detection**. Logs are detailed events for **evidence**. Traces follow one request across services for **localization**.

**Q: What does an SLO have to do with this?**
SLOs (e.g. 99.9% of requests under 3s) define what "healthy" means. Alerts are then driven by **error budget burn rate**, not arbitrary thresholds.

**Q: How would you monitor the EKS control plane?**
Enable **control plane logging** to CloudWatch (at least `audit`, `authenticator`, `api`), and scrape **API server metrics** with Prometheus to watch request latency and error codes.

---

## ✅ 13. Quick Decision Cheat Sheet

| If you need to know… | Go to |
|---|---|
| Is the ALB slow or returning 5xx? | **CloudWatch** |
| Is RDS the bottleneck? | **CloudWatch** (+ Performance Insights) |
| Who changed something in the cluster? | **CloudWatch Logs** (EKS audit log) |
| Which pod is hot or restarting? | **Prometheus / Grafana** |
| Are pods Pending or OOMKilled? | **Prometheus** |
| What's the error rate and p95 latency? | **Prometheus / Grafana** |
| What exception was thrown? | **Splunk** |
| What did one specific request do? | **Splunk** (correlation ID) |
| Which method or SQL query is slow? | **AppDynamics** |
| Who gets paged? | **Alertmanager / CloudWatch Alarms → PagerDuty** |

---

## 🧭 14. Be Honest in Interviews

> For interviews, describe the tools and workflows you have genuinely used. If you have only practiced a particular integration in your lab, distinguish that from your production experience rather than claiming it was deployed in production.

A strong, honest line:
*"In production I've mainly worked with X and Y. I've also practiced Z in a lab setup, so I understand how it works but wouldn't claim production ownership of it."*

---

### 🏁 Closing line

> **"I use CloudWatch for AWS and control plane signals, Prometheus and Grafana for Kubernetes and application metrics, Splunk for log evidence, and AppDynamics for code-level tracing. I correlate all four with a shared request ID to cut MTTR, and I alert on user-facing symptoms, not just resource thresholds."**