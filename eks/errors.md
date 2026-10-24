# AWS EKS Interview Preparation: 502 vs 503 Errors

## 1. HTTP 502 – Bad Gateway

### Interview Answer

“If I receive a **502 Bad Gateway** error, I troubleshoot the request flow from the ALB to the backend application, layer by layer.”

### Troubleshooting Steps

1. **Check ALB Target Health:** Identify unhealthy targets and review the health-check reason.
2. **Check Ingress and Service:** Verify Ingress rules, Service selectors, and `targetPort` configuration.
3. **Check Pod Health:** Run the following commands to identify restarts, failures, or readiness issues.
   ```bash
   kubectl get pods -n <namespace>
   kubectl describe pod <pod-name> -n <namespace>
   ```
4. **Check Application Logs:** Identify application errors, connection resets, or dependency failures.
   ```bash
   kubectl logs <pod-name> -n <namespace>
   ```
5. **Verify Connectivity:** Confirm the application is listening on the correct port, the HTTP/HTTPS protocol matches, and security groups allow the required traffic.
6. **Validate the Fix:** Confirm ALB targets become healthy and test the application endpoint again.

### Conclusion

“I correlate ALB metrics and target-health details with Kubernetes events and application logs to identify the root cause, fix the failing layer, and verify that requests are succeeding.”

**Remember:** A 502 can result from connection resets, invalid backend responses, port or protocol mismatches, or proxy/backend communication failures. I investigate the evidence rather than assuming the application is down.

---

## 2. HTTP 503 – Service Unavailable

### Interview Answer

“If I receive a **503 Service Unavailable** error, I investigate why the service cannot handle incoming requests and identify whether the issue is at the ALB, Kubernetes, application, or dependency layer.”

### Troubleshooting Steps

1. **Check ALB Target Health:** Identify whether targets are unhealthy or unavailable.
2. **Check Ingress and Service:** Verify Ingress rules, Service selectors, and whether the Service has healthy backend endpoints.
3. **Check Pod Availability:** Look for `CrashLoopBackOff`, `Pending`, or failed readiness probes.
   ```bash
   kubectl get pods -n <namespace>
   kubectl get endpoints -n <namespace>
   ```
4. **Check Application Logs:** Identify application errors or dependency failures.
   ```bash
   kubectl logs <pod-name> -n <namespace>
   ```
5. **Check Capacity and Scaling:** Verify replica counts, HPA status, and whether sufficient CPU and memory are available.
   ```bash
   kubectl get hpa -n <namespace>
   kubectl top pods -n <namespace>
   ```
6. **Check Database Connectivity:** If the application depends on a database, verify connectivity and connection-pool health.
7. **Validate the Fix:** Confirm healthy endpoints and ALB targets, then test the application again.

### Conclusion

“I correlate ALB target health, Kubernetes events, Service endpoints, and application logs to identify the root cause, fix it, and verify that requests are succeeding.”

**Remember:** A database failure can contribute to a 503, but it does not automatically cause one. The application or proxy must return that status.

---

## 3. Key Difference: 502 vs 503

| Error | Meaning | Common Cause |
|---|---|---|
| **502 – Bad Gateway** | Gateway receives an invalid backend response or encounters a connection failure. | Connection reset, incorrect port, protocol mismatch. |
| **503 – Service Unavailable** | Service cannot handle the request. | No healthy backend targets, unavailable Pods, or insufficient capacity. |

### Interview Tip

Always troubleshoot **layer by layer**:

**ALB → Ingress → Service → Pods → Application Logs → Dependencies (Database, etc.)**

Identify the actual failing layer using health checks, Kubernetes events, and logs instead of assuming the cause from the HTTP status code alone.