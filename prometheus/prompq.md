Prometheus – 15 Important PromQL Queries for Senior DevOps Interviews
These are practical PromQL queries I recommend preparing for EKS, Kubernetes, Linux, node monitoring, application monitoring and production troubleshooting.
1. Node monitoring
1. Node CPU utilization (%)
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))


Purpose: Identify nodes with high CPU utilization.

2. Node memory utilization (%)
100 * (
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)


Purpose: Identify nodes experiencing high memory consumption.

3. Available memory on nodes
node_memory_MemAvailable_bytes / 1024 / 1024


Purpose: Check available memory in MB.

4. Node disk utilization (%)
100 * (
  1 -
  node_filesystem_avail_bytes{
    mountpoint="/",
    fstype!="tmpfs"
  }
  /
  node_filesystem_size_bytes{
    mountpoint="/",
    fstype!="tmpfs"
  }
)


Purpose: Identify nodes running out of disk space.

5. Node disk I/O utilization
rate(node_disk_io_time_seconds_total[5m]) * 100


Purpose: Identify disks experiencing high I/O activity.
2. Kubernetes / Pod monitoring
6. Top 10 memory-consuming pods
topk(10,
  sum by (namespace, pod) (
    container_memory_working_set_bytes{
      container!="",
      image!=""
    }
  )
)


Purpose: Identify which pods consume the most memory.

7. Top 10 CPU-consuming pods
topk(10,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{
      container!="",
      image!=""
    }[5m])
  )
)


Purpose: Identify CPU-intensive workloads.

8. Pod restart count
sum by (namespace, pod) (
  increase(kube_pod_container_status_restarts_total[1h])
)


Purpose: Identify pods restarting frequently during the last hour.

9. Pods in Pending state
kube_pod_status_phase{phase="Pending"} == 1


Purpose: Identify pods that are waiting to be scheduled or started.

10. Pods not Ready
kube_pod_status_ready{condition="false"} == 1


Purpose: Identify pods that are not ready to receive traffic.
3. OOM and resource monitoring
11. OOMKilled containers
kube_pod_container_status_last_terminated_reason{
  reason="OOMKilled"
} == 1


Purpose: Identify containers whose last termination reason was OOMKilled.

12. Container memory usage versus memory limit (%)
100 *
container_memory_working_set_bytes{
  container!="",
  image!=""
}
/
kube_pod_container_resource_limits{
  resource="memory",
  unit="byte"
}


Purpose: Identify containers approaching their configured memory limits.

13. Container CPU usage versus CPU limit (%)
100 *
rate(container_cpu_usage_seconds_total{
  container!="",
  image!=""
}[5m])
/
kube_pod_container_resource_limits{
  resource="cpu",
  unit="core"
}


Purpose: Identify containers consuming a high percentage of their CPU limits.
4. Application monitoring
14. HTTP 5xx error rate
sum(rate(http_requests_total{status=~"5.."}[5m]))


Purpose: Identify the rate of server-side HTTP errors.

15. Application request rate (RPS)
sum(rate(http_requests_total[5m]))


Purpose: Identify how many requests the application receives per second.

5. Quick interview revision
Area	Query number	What you investigate
Node CPU	1	High CPU
Node memory	2, 3	Memory pressure
Node disk	4, 5	Disk space and I/O
Pod memory	6	Memory-intensive pods
Pod CPU	7	CPU-intensive pods
Pod restarts	8	CrashLoopBackOff investigation
Pending pods	9	Scheduling issues
Pod readiness	10	Readiness failures
OOMKilled	11	Memory-related container termination
Memory limits	12	Memory limit utilization
CPU limits	13	CPU limit utilization
HTTP errors	14	Application failures
Request rate	15	Application traffic

6. Important things to remember
- rate() → Calculates per-second average increase of a counter over a time window.
- increase() → Calculates total counter increase over a time window.
- sum() → Adds values.
- avg() → Calculates average.
- topk() → Returns the top K results.
- by (namespace, pod) → Groups results by namespace and pod.
- [5m] → Looks at a five-minute range.
Important: These queries assume the relevant exporters and metric sources are configured. For example, node metrics need node-exporter, Kubernetes object metrics need kube-state-metrics, and container resource metrics generally come from kubelet/cAdvisor. Application HTTP metrics need application instrumentation. Some metric labels can differ depending on your Prometheus setup.

My recommendation: Practise queries 1, 2, 6, 7, 8, 11, 12 and 14 first. These are especially useful in real-time EKS troubleshooting and senior DevOps interviews.