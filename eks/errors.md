“If I receive a 502 Bad Gateway error, I troubleshoot the request flow from the ALB to the backend application, layer by layer.
1. Check ALB target health: Identify unhealthy targets and review the health-check reason.
2. Check Ingress and Service: Verify the Ingress rules, Service selectors, and targetPort configuration.
3. Check Pod health: Run kubectl get pods -n <namespace> and kubectl describe pod <pod-name> -n <namespace> to identify restarts, failures, or readiness issues.
4. Check application logs: Run kubectl logs <pod-name> -n <namespace> to identify application errors, connection resets, or dependency failures.
5. Verify connectivity: Confirm the application is listening on the correct port, the protocol matches (HTTP/HTTPS), and security groups allow the required traffic.
6. Validate the fix: Confirm the ALB targets become healthy and test the application endpoint again.
Conclusion: I correlate ALB metrics and target-health details with Kubernetes events and application logs to identify the root cause, fix the failing layer, and verify that requests are succeeding.”
Remember for the interview: A 502 can result from connection resets, invalid backend responses, port or protocol mismatches, or proxy/backend communication failures. I would investigate the evidence rather than assume the application is down.

“If I receive a 503 Service Unavailable error, I investigate why the service cannot handle incoming requests and identify whether the issue is at the ALB, Kubernetes, application, or dependency layer.
1. Check ALB target health: Identify whether the targets are unhealthy or unavailable.
2. Check Ingress and Service: Verify Ingress rules, Service selectors, and whether the Service has healthy backend endpoints.
3. Check Pod availability: Run kubectl get pods -n <namespace> and check for CrashLoopBackOff, Pending, or failed readiness probes.
4. Check application logs: Run kubectl logs <pod-name> -n <namespace> to identify application errors or dependency failures.
5. Check capacity and scaling: Verify replica counts, HPA status, and whether sufficient CPU and memory are available.
6. Check database connectivity: If the application depends on a database, verify connectivity and connection-pool health.
7. Validate the fix: Confirm healthy endpoints and ALB targets, then test the application again.
Conclusion: I correlate ALB target health, Kubernetes events, Service endpoints, and application logs to identify the root cause, fix it, and verify that requests are succeeding.”
Key difference for the interview
- 502: The gateway encounters an invalid backend response or a connection failure.
- 503: The service is unable to handle the request, often because no healthy backend is available.
Important: A database failure can contribute to a 503, but it does not automatically cause one; the application or proxy must return that status.


