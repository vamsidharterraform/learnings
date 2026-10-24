# Troubleshooting Application Latency on EKS (Senior DevOps / SRE Interview)

For a senior DevOps / SRE interview, the strongest answer is not to say, *"I checked Grafana and restarted the pods."* Instead, explain how you **isolated the latency** across the application, Kubernetes, infrastructure, and downstream dependencies using **metrics, logs, and distributed traces**.

## Contents

1. [Interview-ready answer](#1-interview-ready-answer)
2. [What I would check in the real environment](#2-what-i-would-check-in-the-real-environment)
3. [Three scenarios interviewers may ask about](#3-three-scenarios-interviewers-may-ask-about)
4. [A follow-up question you should be ready for](#4-a-follow-up-question-you-should-be-ready-for)

---

## Stack and Tool Roles

Your stack is **EKS with Prometheus, Grafana, Loki, and OpenTelemetry**. Each tool has a different role:

| Tool | Role |
|------|------|
| **Prometheus + Grafana** | Identify when latency started and which resource or service is affected |
| **OpenTelemetry** | Trace a slow request across services and identify which operation is taking the most time |
| **Loki** | Investigate application errors, timeouts, retries, and other events around the same timestamp |
| **Kubernetes / AWS** | Verify pod health, CPU and memory limits, node pressure, networking, and dependencies such as RDS |

---

## 1. Interview-Ready Answer

You can use this as your spoken answer. Adapt the investigation findings to what you actually observed in your project.

> When users reported application latency, I treated it as a production incident and started by establishing the impact, rather than immediately restarting pods or scaling the cluster.
>
> First, I checked Grafana dashboards backed by Prometheus to identify when the latency started and whether it affected all requests or only a particular endpoint. I compared request rate, error rate, and latency percentiles such as p95 and p99. I also checked pod CPU and memory usage, CPU throttling, restarts, and node-level resource utilization.
>
> Next, I used OpenTelemetry distributed traces to follow slow requests through the application. I examined the duration of individual spans to determine whether the delay was coming from application processing, database queries, external API calls, or communication between services.
>
> Once I had identified the affected component, I correlated the trace timestamp with Loki application logs. I searched for exceptions, database timeouts, connection-pool exhaustion, retries, and slow-query messages.
>
> On the Kubernetes side, I checked pod status, events, resource requests and limits, HPA activity, node health, and whether pods were being scheduled or restarted. I also verified that the service endpoints and ingress or load-balancer path were healthy.
>
> Based on the evidence, I would apply a targeted mitigation, for example, fixing a slow database query, adjusting an undersized connection pool, resolving CPU throttling, or scaling replicas if the workload justified it. I would then verify recovery using p95/p99 latency, error rate, and distributed traces.
>
> Finally, I would document the root cause, customer impact, mitigation, and preventive actions, and improve alerting or dashboards if the incident exposed a monitoring gap.
>
> **My approach is to use metrics to detect and narrow the problem, traces to locate the slow operation, logs to explain the failure, and Kubernetes and AWS telemetry to validate the underlying cause.**

---

## 2. What I Would Check in the Real Environment

Follow this sequence during an incident.

### Step 1: Grafana + Prometheus (Establish the symptoms)

Check p95/p99 latency, throughput, error rate, CPU, memory, throttling, and pod restarts. Compare the incident window with the normal baseline.

Example PromQL, assuming your application exports a request-duration histogram:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)
```

> This estimates p95 request latency. The metric name and labels must match your instrumentation.

### Step 2: OpenTelemetry (Find where the time is spent)

Open a slow trace and compare its spans. For example:

| Span | Duration |
|------|----------|
| API processing | 40 ms |
| Database query | **1,800 ms** |
| External API | 30 ms |

In this hypothetical example, the **database span is the main lead**. Investigate query execution, locks, connection-pool waits, and database load before changing the application.

### Step 3: Loki (Correlate application logs)

Search the incident time window for database timeouts, retry messages, exceptions, and slow requests. Use **trace IDs** to connect logs with the relevant distributed trace when your application includes those IDs in its logs.

### Step 4: EKS (Validate the infrastructure)

```bash
kubectl get pods -A -o wide
kubectl top pods -A
kubectl top nodes
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp
```

Investigate CPU throttling, memory pressure, OOM kills, pending pods, insufficient capacity, unhealthy endpoints, and node or networking issues.

### Step 5: Mitigate and verify

Make a change **only after identifying a plausible cause**:

- Scale replicas if capacity is the bottleneck
- Fix inefficient queries if the database is slow
- Adjust resource limits when CPU throttling is demonstrated

Verify recovery using the same latency percentiles, error rate, traces, and application health checks.

---

## 3. Three Scenarios Interviewers May Ask About

| Evidence | Likely direction | Action |
|----------|------------------|--------|
| High CPU throttling and increased latency | CPU limit or CPU-intensive processing | Profile the application; review requests, limits, and replica capacity |
| Normal CPU but long database spans | Database or connection-pool bottleneck | Inspect slow queries, locks, connections, and database metrics |
| High p99 latency, normal p50, intermittent errors | A subset of slow requests, retries, or dependency issues | Compare slow and fast traces; investigate timeouts and tail latency |

> These are **investigation leads, not proof of root cause**. For example, high CPU does not automatically mean you should increase replicas.

---

## 4. A Follow-Up Question You Should Be Ready For

**Interviewer:** *"What if Grafana shows high latency, but CPU and memory are normal?"*

**A strong answer:**

> "I wouldn't assume the problem is in the EKS compute layer. I would inspect distributed traces to identify the slow span, then correlate it with logs and dependency metrics. The issue could be database locks, connection-pool exhaustion, external API latency, network delays, or retry amplification. I would isolate the affected dependency, mitigate the confirmed bottleneck, and verify recovery against the original latency baseline."

---

## Final Interview Tip

If you haven't personally investigated a real production incident, present this as **your troubleshooting approach** rather than claiming a specific root cause or resolution.

If you have a real incident from your environment, the strongest version includes:

- The **measured latency** before and after
- The **evidence** that identified the cause
- The **exact corrective action**