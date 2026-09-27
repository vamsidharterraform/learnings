AWS DevOps – EKS Ingress, ALB & Route 53 Interview Playbook
1. What is Kubernetes Ingress?
Answer:
Kubernetes Ingress is an API object used to define how external HTTP/HTTPS traffic should be routed to services running inside a Kubernetes cluster.
Ingress itself is not the load balancer. It contains routing rules.
For example:
Internet
   |
Route 53
   |
   v
AWS ALB
   |
   v
Ingress Rules
   |
   +-------------------+
   |                   |
   v                   v
frontend-service    backend-service
   |                   |
   v                   v
Frontend Pods       Backend Pods
Example:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: employee-ingress
spec:
  rules:
    - host: employee.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: employee-backend
                port:
                  number: 8080

          - path: /
            pathType: Prefix
            backend:
              service:
                name: employee-frontend
                port:
                  number: 80
Here:
employee.example.com/api/*
        -> backend-service

employee.example.com/*
        -> frontend-service
2. Is Ingress a Load Balancer?
Answer:
No.
Ingress is a Kubernetes API resource that defines routing rules.
An Ingress Controller implements those rules.
In AWS EKS, a common implementation is the:
AWS Load Balancer Controller
It watches Kubernetes Ingress resources and creates/configures an AWS Application Load Balancer.
Kubernetes Ingress
       |
       v
AWS Load Balancer Controller
       |
       v
AWS ALB
So I would explain it as:
"Ingress defines what traffic should go where, while the Ingress Controller implements those rules using the underlying load-balancing infrastructure."

3. What is AWS Load Balancer Controller?
The AWS Load Balancer Controller is a Kubernetes controller that manages AWS load-balancing resources.
For an Ingress resource, it can create:
ALB
Listeners
Listener Rules
Target Groups
Security Group configuration
Target registrations
Example:
Ingress
   |
   v
AWS Load Balancer Controller
   |
   +---- ALB
   |
   +---- Listener :80
   |
   +---- Listener :443
   |
   +---- Target Groups
4. Explain the complete request flow
This is one of the most important interview questions.
Suppose the user accesses:
https://app.example.com/api/employees
The flow is:
User
 |
 v
Route 53
 |
 | DNS resolution
 v
ALB
 |
 | HTTPS :443
 v
ALB Listener
 |
 | Listener Rule
 v
Target Group
 |
 v
Kubernetes Service
 |
 v
Backend Pod
 |
 v
Application
With AWS Load Balancer Controller using IP target mode, the ALB can register pod IPs directly.
So the flow can effectively be:
Internet
   |
Route 53
   |
   v
ALB
   |
   v
Target Group
   |
   v
Pod IP
   |
   v
Backend Container
5. What is path-based routing?
Path-based routing means routing traffic based on the URL path.
Example:
app.example.com/
app.example.com/api
app.example.com/reports
app.example.com/admin
We can route them to different services.
                ALB
                 |
       +---------+---------+
       |         |         |
       v         v         v
      /        /api     /reports
       |         |         |
       v         v         v
 frontend    backend    reporting
 service     service    service
Example:
paths:
  - path: /
    pathType: Prefix
    backend:
      service:
        name: frontend
        port:
          number: 80

  - path: /api
    pathType: Prefix
    backend:
      service:
        name: backend
        port:
          number: 8080

  - path: /reports
    pathType: Prefix
    backend:
      service:
        name: reports
        port:
          number: 8080
6. What is host-based routing?
Host-based routing routes requests based on the hostname.
For example:
www.example.com
api.example.com
reports.example.com
admin.example.com
We can send each host to a different service.
                 ALB
                  |
       +----------+----------+
       |          |          |
       v          v          v
www.example   api.example  reports.example
       |          |          |
       v          v          v
   frontend    backend     reports
Example:
rules:

  - host: www.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: frontend
              port:
                number: 80

  - host: api.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: backend
              port:
                number: 8080
7. Path-based vs Host-based routing
Feature	Path Based	Host Based
Routing based on	URL path	Hostname
Example	/api	api.example.com
ALB	Same ALB possible	Same ALB possible
Useful for	Microservices	Separate applications/domains
TLS	Same certificate can work	Certificate must cover hosts
Example	app.com/api	api.app.com


8. Can multiple applications use a single ALB?
Yes.
This is a very important real-world scenario.
Suppose we have:
app.company.com
api.company.com
reports.company.com
admin.company.com
We don't necessarily need four ALBs.
We can use:
                    ALB
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
     Host 1        Host 2        Host 3
       |             |             |
       v             v             v
   Frontend       Backend       Reports
This can reduce infrastructure overhead and simplify centralized TLS and routing.
9. How do you configure multiple Ingress resources to use one ALB?
With AWS Load Balancer Controller, we can use an IngressGroup.
Example:
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: employee-platform
Multiple Ingress objects using the same group can share an ALB.
Example:
Ingress 1
group.name = employee-platform

Ingress 2
group.name = employee-platform

Ingress 3
group.name = employee-platform
The controller can consolidate them onto the same ALB.
10. What happens if two teams create Ingress rules for the same host/path?
This is an important production question.
If multiple Ingress resources belong to the same IngressGroup and define conflicting rules, the AWS Load Balancer Controller has to resolve the rules according to its ordering and configuration behavior.
From a production governance perspective, I would avoid allowing unrestricted teams to modify a shared ALB.
I would use:
Central ingress ownership
        +
RBAC
        +
Namespace boundaries
        +
Admission policies
        +
Clearly defined host/path ownership
The important point in an interview is:
"Shared ALBs require governance because one team's Ingress configuration can affect the routing configuration of other teams."

11. What is pathType?
Kubernetes supports:
Exact
Prefix
ImplementationSpecific
Exact
path: /api
pathType: Exact
Matches:
/api
but not necessarily:
/api/users
Prefix
path: /api
pathType: Prefix
Matches:
/api
/api/users
/api/orders
/api/users/123
ImplementationSpecific
Behavior depends on the Ingress Controller.
For AWS environments, I generally prefer explicitly defined Exact or Prefix rules when they meet the requirement, because the routing intent is clearer.
12. What is the difference between ALB and NLB?
ALB	NLB
Layer 7	Layer 4
HTTP/HTTPS	TCP/UDP/TLS
Host routing	IP/port based
Path routing	No normal HTTP path routing
HTTP headers	Limited/HTTP aware
Web applications	Network-heavy workloads
WAF integration	Common


For Kubernetes:
Ingress
   |
   v
ALB
is commonly used for HTTP/HTTPS application routing.
While:
Service type LoadBalancer
   |
   v
NLB
is common for TCP/UDP or other Layer-4 workloads.
13. What is the difference between Ingress and Service LoadBalancer?
Service
Example:
kind: Service
spec:
  type: LoadBalancer
This typically provisions an AWS load-balancing resource for that service.
If you have:
frontend
backend
reports
you could end up with multiple load balancers.
Ingress
Ingress allows:
                    ALB
                     |
          +----------+----------+
          |          |          |
          v          v          v
       frontend    backend    reports
So one ALB can handle multiple HTTP applications.
14. What is ALB Target Type?
AWS Load Balancer Controller supports target modes such as:
instance
ip
Instance mode
Traffic goes to:
ALB
 |
 v
Node
 |
 v
kube-proxy / Service
 |
 v
Pod
IP mode
Traffic can go directly to:
ALB
 |
 v
Pod IP
For EKS with the AWS VPC CNI, IP target mode is commonly used for application pods.
15. Why would you use IP target mode?
Suppose:
Node 1
  |
  +--- Pod A
  +--- Pod B
If using instance targets:
ALB
 |
 v
Node 1
 |
 v
Service
 |
 v
Pod
With IP targets:
ALB
 |
 v
Pod IP
This removes an extra service/node hop and allows ALB target registration directly against pod IPs.
16. What is the role of Route 53?
Route 53 is AWS's DNS service.
Suppose the application is:
https://employee.company.com
DNS resolution can work like:
employee.company.com
          |
          v
       Route 53
          |
          v
         ALB
          |
          v
       EKS Pods
Route 53 doesn't perform Kubernetes routing.
It resolves the domain to the load balancer.
17. Does Route 53 send traffic directly to Kubernetes pods?
No.
Normally:
Route 53
   |
   v
ALB
   |
   v
Target Group
   |
   v
Pod
Route 53 is responsible for DNS resolution.
ALB is responsible for HTTP/HTTPS load balancing and routing.
Kubernetes/Ingress configuration determines the application routing.
18. How does Route 53 point to an ALB?
Using an Alias record.
Example:
api.company.com
        |
        v
Route 53 Alias
        |
        v
internal/external ALB
Typical record:
Name: api.company.com
Type: A
Alias: Yes
Target: ALB
19. Why use Alias instead of CNAME?
For AWS resources, Route 53 Alias records are commonly preferred because they can point directly to supported AWS resources such as:
ALB
CloudFront
S3 website endpoints
API Gateway
An Alias record also works at the zone apex where a normal CNAME cannot.
Example:
company.com
20. How do you configure HTTPS?
Typical architecture:
Client
  |
  | HTTPS 443
  v
ALB
  |
  | HTTP/HTTPS
  v
Kubernetes Service
  |
  v
Pod
Certificate:
AWS Certificate Manager
          |
          v
         ALB
Ingress annotation example:
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
And:
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
21. How do you redirect HTTP to HTTPS?
A common ALB configuration is:
HTTP :80
   |
   v
Redirect
   |
   v
HTTPS :443
Example annotation:
alb.ingress.kubernetes.io/ssl-redirect: '443'
Then:
http://app.company.com
becomes:
https://app.company.com
22. Where does TLS terminate?
One common architecture is:
Client
  |
 HTTPS
  |
  v
ALB
  |
 TLS termination
  |
 HTTP
  |
  v
Pod
This is called TLS termination at the ALB.
Another architecture can re-encrypt traffic:
Client
  |
 HTTPS
  v
ALB
  |
 HTTPS
  v
Pod
The decision depends on security requirements and the application's trust model.
23. Production scenario: One ALB, multiple services
Suppose you have:
www.company.com
api.company.com
reports.company.com
I'd design:
                    Route 53
                       |
                       v
                      ALB
                       |
             +---------+---------+
             |         |         |
             v         v         v
            Host      Host      Host
             |         |         |
             v         v         v
         Frontend    Backend   Reporting
Ingress rules:
www.company.com       -> frontend
api.company.com       -> backend
reports.company.com   -> reporting
24. Production scenario: One domain, multiple microservices
Suppose:
company.com
has:
/api
/orders
/payments
/reports
Architecture:
                  ALB
                   |
      +------------+------------+
      |            |            |
      v            v            v
     /api       /orders      /payments
      |            |            |
      v            v            v
    API Pod     Order Pod    Payment Pod
This is path-based routing.
25. Can host-based and path-based routing be combined?
Yes.
This is a very common real-world design.
Example:
api.company.com/users
api.company.com/orders

admin.company.com/users
admin.company.com/reports
Routing can be based on both:
Host + Path
Example:
api.company.com/orders
        |
        +--> Order Service

api.company.com/users
        |
        +--> User Service

admin.company.com/reports
        |
        +--> Reporting Service
26. What happens if no Ingress rule matches?
The ALB can return an HTTP error such as:
404
depending on the listener/routing configuration.
Troubleshooting:
kubectl get ingress -A

kubectl describe ingress <name> -n <namespace>

kubectl get svc -A

kubectl get pods -A
Then check the ALB listener rules.
27. Scenario: ALB returns 404
Suppose:
https://api.company.com/orders
returns:
404
I would check in this order:
Step 1
Check DNS:
nslookup api.company.com
or:
dig api.company.com
Step 2
Check Ingress:
kubectl get ingress -A
Step 3
Describe:
kubectl describe ingress <ingress-name> -n <namespace>
Step 4
Verify host:
api.company.com
matches the request.
Step 5
Verify path:
/orders
matches the Ingress rule.
Step 6
Check ALB listener rules.
Step 7
Check service:
kubectl get svc -n <namespace>
Step 8
Check endpoints:
kubectl get endpoints -n <namespace>
or:
kubectl get endpointslices -n <namespace>
28. Scenario: ALB returns 502
This is a very important interview scenario.
A 502 generally indicates that the ALB couldn't successfully communicate with the configured target.
I would investigate:
ALB
 |
 v
Target Group
 |
 v
Target health
 |
 v
Pod
 |
 v
Application
Check:
kubectl get pods -n <namespace>

kubectl get svc -n <namespace>

kubectl get endpoints -n <namespace>

kubectl describe ingress <name> -n <namespace>
Then inspect target health from AWS.
Possible causes:
Wrong target port
Wrong Service port
Pod not listening
Security group
Network policy
Application failure
Incorrect health check path
Readiness failure
Target registration issue
29. What is the difference between 404 and 502?
404
Usually means:
Request reached the routing layer
but no appropriate route/resource was found.
Potential issue:
Host
Path
Ingress rule
Application route
502
Usually means:
ALB could not successfully communicate with the backend target.
Potential issue:
Target health
Port
Security group
Pod
Application
Network
30. Scenario: ALB target is unhealthy
Suppose AWS Target Group shows:
Unhealthy
I would verify:
Health check path
Example:
/health
Can the application actually respond to it?
Test inside the cluster:
kubectl exec -it <pod> -- curl localhost:8080/health
Port
Check:
kubectl get svc <service> -o yaml
Verify:
Service port
TargetPort
ContainerPort
Security groups
Verify ALB → target traffic is allowed.
NetworkPolicy
If NetworkPolicies exist, ensure the ALB traffic is permitted.
31. What is a readiness probe?
Readiness determines whether a Pod is ready to receive application traffic.
Example:
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
If readiness fails:
Pod remains Running
but is not considered Ready.
Therefore it should not receive normal Service traffic.
32. What is the difference between readiness and liveness?
Probe	Purpose
Readiness	Can the application receive traffic?
Liveness	Is the application alive?
Startup	Has the application finished starting?


Example:
Liveness failure
      |
      v
Container restart

Readiness failure
      |
      v
Remove from service traffic
33. Production scenario: Application deployment causes 502s
Suppose deployment changed:
backend:8080
to:
backend:9090
but Service still uses:
targetPort: 8080
Traffic:
ALB
 |
 v
Pod
 |
 v
8080
 |
 X
Application listening on 9090
Result:
502 / connection failure
Troubleshooting:
kubectl get svc backend -o yaml

kubectl describe pod <pod>

kubectl exec -it <pod> -- ss -lntp
34. What is ALB health check configuration?
The ALB periodically checks targets.
Example:
Protocol: HTTP
Port: traffic-port
Path: /health
If the target responds successfully:
Healthy
Otherwise:
Unhealthy
Ingress annotations can customize health checks.
For example:
alb.ingress.kubernetes.io/healthcheck-path: /health
35. Production scenario: /api works but /api/users doesn't
First check the path type.
If:
path: /api
pathType: Exact
then:
/api
matches.
But:
/api/users
does not match as an exact path.
Use:
path: /api
pathType: Prefix
if the intention is to route all /api/* traffic.
36. Scenario: Two applications share the same ALB
Example:
Application A
api.company.com

Application B
reports.company.com
Both use:
alb.ingress.kubernetes.io/group.name: shared-alb
Result:
                    ALB
                     |
           +---------+---------+
           |                   |
           v                   v
   api.company.com     reports.company.com
           |                   |
           v                   v
       Backend             Reports
Benefits:
Less infrastructure
Centralized TLS
Centralized ingress
Lower operational overhead
But governance becomes important.
37. Blue/Green deployment with ALB
Suppose:
Blue = v1
Green = v2
Architecture:
                   ALB
                    |
             +------+------+
             |             |
             v             v
           Blue          Green
            v1             v2
Initially:
100% -> Blue
After validation:
100% -> Green
This can be implemented using target groups/listener rules or deployment tooling.
38. Canary deployment
Canary means gradually sending traffic to the new version.
Example:
95% -> v1
5%  -> v2
Then:
80% -> v1
20% -> v2
Then:
50% -> v1
50% -> v2
Finally:
0% -> v1
100% -> v2
The exact mechanism depends on the ingress/controller/deployment strategy.
39. What is weighted routing in Route 53?
Route 53 weighted routing allows DNS responses to be distributed according to configured weights.
Example:
app.company.com
       |
       +---- ALB Blue
       |
       +---- ALB Green
For example:
Blue  = 90
Green = 10
This is DNS-level traffic distribution.
It is different from ALB listener-based routing.
40. Route 53 Failover Routing
A classic disaster recovery architecture:
                 Route 53
                    |
             Health Check
                    |
          +---------+---------+
          |                   |
          v                   v
       Primary             Secondary
         ALB                  ALB
          |                    |
          v                    v
       EKS AZs             DR EKS
If the primary endpoint becomes unhealthy, Route 53 can return the secondary endpoint.
41. What is the difference between ALB failover and Route 53 failover?
ALB
Handles:
Application-level load balancing
Target health
Path routing
Host routing
Route 53
Handles:
DNS-level routing
Failover
Weighted routing
Latency routing
Geolocation routing
Think:
Route 53 = Which endpoint?

ALB = Which application target?
42. Multi-Region architecture
Example:
                    Route 53
                       |
             +---------+---------+
             |                   |
             v                   v
        us-east-1             us-west-2
             |                   |
            ALB                 ALB
             |                   |
            EKS                 EKS
Route 53 can route users between regions using appropriate routing policies.
For disaster recovery, failover routing can be used.
43. Scenario: DNS works but application doesn't
Suppose:
nslookup app.company.com
returns the ALB DNS name/IPs.
But browser returns:
502
DNS is probably not the primary issue.
Continue:
DNS
 |
 v
ALB
 |
 v
Target Group
 |
 v
Pod
Check target health and application connectivity.
44. Scenario: DNS does not resolve
Check:
dig app.company.com
Verify:
Hosted Zone
Record
Alias target
Nameservers
Domain delegation
If the hosted zone is public, verify the domain's registrar delegation points to the Route 53 hosted-zone nameservers.
45. Scenario: HTTPS certificate error
Check:
Certificate ARN
Certificate status
Domain names
ALB listener
Certificate association
Example:
api.company.com
must be covered by the certificate.
A wildcard certificate such as:
*.company.com
can cover:
api.company.com
www.company.com
but does not generally cover:
api.dev.company.com
because that is another subdomain level.
46. Scenario: One ALB has 50+ services
An interviewer might ask:
Would you keep putting everything behind one ALB?

There isn't one universal answer.
I would consider:
Number of applications
ALB quotas
Traffic volume
Security boundaries
Team ownership
Blast radius
Operational complexity
Routing rule limits
Possible architecture:
                  Route 53
                     |
        +------------+------------+
        |                         |
        v                         v
    Public ALB                Internal ALB
        |                         |
    Web APIs                  Internal APIs
For larger platforms, multiple ALBs can provide better isolation.
47. Public vs Internal ALB
Internet-facing ALB
Internet
   |
   v
Internet-facing ALB
   |
   v
EKS
Internal ALB
Corporate Network / VPN
          |
          v
     Internal ALB
          |
          v
         EKS
Internal ALBs are commonly used for private applications.
48. How would you expose a private application?
Example:
Corporate User
      |
      v
VPN / Direct Connect
      |
      v
Internal ALB
      |
      v
EKS Service
      |
      v
Pods
Route 53 private hosted zones can be used for private DNS names.
Example:
internal.company.com
49. What happens if the ALB is deleted manually?
If the ALB is managed by AWS Load Balancer Controller, the Kubernetes Ingress remains the desired state.
The controller continuously reconciles Kubernetes resources with AWS resources.
Conceptually:
Desired state
     |
     v
Ingress
     |
     v
Controller
     |
     v
AWS ALB
If the actual state differs, the controller attempts to reconcile it.
This is the Kubernetes controller pattern.
50. What is reconciliation?
This is a strong senior-level interview concept.
Kubernetes controllers continuously compare:
Desired State
      vs
Actual State
Example:
Desired:
Ingress should have ALB

Actual:
ALB doesn't exist
Controller detects the difference and attempts to create/reconcile the resource.
51. How would you troubleshoot Ingress from Kubernetes?
My normal sequence would be:
kubectl get ingress -A
Then:
kubectl describe ingress <name> -n <namespace>
Then:
kubectl get svc -n <namespace>
Then:
kubectl get endpointslices -n <namespace>
Then:
kubectl get pods -n <namespace> -o wide
Then controller logs:
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller
Then inspect AWS:
ALB
Listeners
Rules
Target Groups
Target Health
Security Groups
52. Interview scenario: Ingress exists but ALB wasn't created
I would investigate:
1. Controller running?
kubectl get pods -n kube-system
2. Controller logs
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller
3. IAM permissions
Verify the controller's IAM role.
4. Service account
kubectl get sa -n kube-system
5. OIDC / IRSA
Verify the service account is correctly associated with the IAM role.
6. Subnets
Check whether suitable subnets are correctly tagged/discoverable.
7. Ingress configuration
kubectl describe ingress <name>
53. Why does AWS Load Balancer Controller need IAM permissions?
The controller runs inside Kubernetes but needs to create and manage AWS resources.
For example:
Create ALB
Create Target Group
Modify Listener
Register Targets
Configure Security Groups
Therefore it needs AWS permissions.
The preferred architecture is generally:
Kubernetes ServiceAccount
          |
          v
       OIDC
          |
          v
      IAM Role
          |
          v
     AWS APIs
54. Why use IRSA?
IRSA means:
IAM Roles for Service Accounts
It allows a Kubernetes service account to assume a specific AWS IAM role.
Instead of:
Every pod
   |
   v
Same node IAM permissions
we can use:
AWS Controller Pod
       |
       v
ServiceAccount
       |
       v
IAM Role
This provides more granular permissions.
55. Interview scenario: ALB works but pod cannot access RDS
Don't immediately blame Ingress.
The request path might be:
User
 |
 v
ALB
 |
 v
Backend Pod
 |
 v
RDS
Ingress only handles:
User -> Backend
The Pod → RDS connection depends on:
Security Groups
Network ACLs
Routing
DNS
RDS endpoint
NetworkPolicy
Database port
Credentials
For MySQL:
TCP 3306
must be reachable.
56. Interview scenario: frontend works, backend API fails
Example:
Frontend:
https://app.company.com

API:
https://app.company.com/api
Frontend loads but API returns:
502
I would check:
ALB listener rule
       |
       v
/api rule
       |
       v
Backend target group
       |
       v
Backend service
       |
       v
Backend pods
Then verify application-level configuration:
CORS
API URL
Environment variables
Authentication
Backend health
57. How does CORS relate to Ingress?
CORS is an application/browser security mechanism.
Ingress/ALB routing determines where traffic goes.
Example:
frontend.company.com
        |
        | API request
        v
api.company.com
The browser may enforce CORS depending on the response headers.
So:
Ingress routing problem
and:
CORS problem
are different layers.
58. What is sticky session?
Sticky sessions mean requests from a client can continue going to the same backend target for a period.
Example:
Client A
   |
   +--> Pod 1
   |
   +--> Pod 1
   |
   +--> Pod 1
rather than:
Pod 1
Pod 2
Pod 3
For cloud-native applications, I generally prefer designing applications to be stateless when possible.
Session state can instead be stored in systems such as:
Redis
Database
DynamoDB
rather than relying heavily on a specific pod.
59. What happens during a rolling deployment?
Suppose:
v1 Pods
Pod 1
Pod 2
Pod 3
We deploy:
v2
Kubernetes gradually creates/removes pods based on the Deployment strategy.
Readiness probes help prevent traffic from being sent to pods that aren't ready.
Conceptually:
ALB
 |
 +--> v1 Pod
 +--> v1 Pod
 +--> v2 Pod (not ready)
Once v2 becomes ready:
ALB
 |
 +--> v1
 +--> v2
 +--> v2
Eventually:
ALB
 |
 +--> v2
 +--> v2
 +--> v2
60. What is the most important troubleshooting model?
In an interview, don't randomly run commands.
Use layers:
Layer 1
DNS

   ↓

Layer 2
ALB

   ↓

Layer 3
Listener / Rule

   ↓

Layer 4
Target Group

   ↓

Layer 5
Service

   ↓

Layer 6
Endpoint / Pod

   ↓

Layer 7
Application

   ↓

Layer 8
Database / External Dependency
This gives you a systematic troubleshooting approach.
61. Real-time troubleshooting example
Interviewer:
"Users are getting 502 from the ALB. How will you troubleshoot?"

Strong answer:
"First I'll determine whether the problem is at DNS, ALB routing, target health, Kubernetes service discovery, networking, or the application layer. Since DNS resolution and ALB access are already confirmed, I'll start with the target group health."

Then:
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>
kubectl get svc -n <namespace>
kubectl get endpointslices -n <namespace>
kubectl get pods -o wide -n <namespace>
Then:
AWS Console
   |
   +-- ALB
   +-- Listener
   +-- Listener Rules
   +-- Target Group
   +-- Target Health
Then:
kubectl logs <pod> -n <namespace>
Finally verify:
Security Groups
NetworkPolicy
Application port
Health endpoint
That is a much stronger answer than simply saying:
"I will check the logs."

62. Scenario: ALB health check is failing but pod is healthy
This is a common production issue.
Suppose:
Pod
Running
Ready
but:
ALB Target
Unhealthy
Possible mismatch:
ALB health check:
GET /health

Application:
GET /actuator/health
Therefore:
ALB -> /health -> 404
while:
Pod -> /actuator/health -> 200
Solution:
Configure the correct health-check path.
63. Scenario: ALB shows healthy but users still get 500
Important distinction:
ALB Target Health = Healthy
doesn't mean:
Application = Fully healthy
The ALB may successfully connect to the application, but the application itself could return:
500
because of:
Database failure
Code bug
Dependency failure
Authentication issue
Configuration issue
Therefore:
ALB health
and:
Application health
are not identical.
64. Scenario: Route 53 points to wrong ALB
Check:
dig api.company.com
Then verify the Route 53 record.
Architecture should be:
api.company.com
       |
       v
Correct ALB
       |
       v
Correct EKS cluster
If the record points to an old ALB:
DNS
 |
 v
Old ALB
 |
 X
Old/removed application
65. Scenario: Application works internally but not externally
Test from inside cluster:
kubectl exec -it <pod> -- curl http://backend-service:8080/health
If that works:
Pod -> Service -> Pod
is functioning.
Then test:
External
  |
  v
ALB
  |
  v
Target
This narrows the problem to:
ALB
Ingress
Security Group
Target registration
Health checks
DNS
66. Scenario: Application works through Service but not through ALB
Suppose:
kubectl port-forward svc/backend 8080:8080
works.
But:
https://api.company.com
returns 502.
Then focus on:
Ingress
ALB
Target Group
Health Check
Security Groups
Target Type
rather than debugging the application first.
67. What are common ALB annotations?
Some commonly encountered annotations include:
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
alb.ingress.kubernetes.io/ssl-redirect: '443'
alb.ingress.kubernetes.io/healthcheck-path: /health
alb.ingress.kubernetes.io/group.name: shared-alb
68. What is the difference between IngressClass and Ingress?
Ingress defines the actual routing rules.
IngressClass identifies which controller should handle that Ingress.
Conceptually:
Ingress
   |
   | references
   v
IngressClass
   |
   v
Controller
This becomes important when a cluster has multiple ingress controllers.
69. Can you have multiple ingress controllers?
Yes.
For example:
AWS Load Balancer Controller
        |
        v
AWS ALB

NGINX Ingress Controller
        |
        v
NGINX
IngressClass can determine which controller handles a particular Ingress.
70. Senior interview question: Why not put every service behind its own ALB?
A good answer:
"I would avoid creating a separate ALB for every HTTP microservice unless there is a strong isolation or security requirement. Using shared ALBs can reduce infrastructure overhead and centralize routing and TLS. However, I would consider separate ALBs when teams need independent ownership, different security boundaries, very high traffic, different exposure requirements, or when shared routing would create an unacceptable blast radius."

71. Senior interview question: How would you design ingress for production EKS?
I would consider:
                    Route 53
                       |
                       v
                  CloudFront
                       |
                       v
                     WAF
                       |
                       v
                     ALB
                       |
             +---------+---------+
             |                   |
             v                   v
        Frontend/API        Internal APIs
             |                   |
             v                   v
            EKS                 EKS
             |
       +-----+-----+
       |           |
       v           v
   Services      Pods
Depending on the requirements, CloudFront/WAF may or may not be placed in front of the ALB.
72. How would you secure an ALB?
Consider:
HTTPS only
ACM certificates
HTTP -> HTTPS redirect
AWS WAF
Security Groups
Private ALB where appropriate
Least-privilege IAM
NetworkPolicies
Authentication/authorization
Access logging
Monitoring
73. How do you monitor ALB?
CloudWatch metrics can include:
RequestCount
TargetResponseTime
HTTPCode_ELB_4XX_Count
HTTPCode_ELB_5XX_Count
HTTPCode_Target_4XX_Count
HTTPCode_Target_5XX_Count
HealthyHostCount
UnHealthyHostCount
You can build alerts such as:
5XX rate > threshold
UnhealthyHostCount > 0
TargetResponseTime > threshold
74. ALB 5XX vs Target 5XX
This is a very good interview question.
There is a difference between:
HTTPCode_ELB_5XX_Count
and:
HTTPCode_Target_5XX_Count
Conceptually:
ELB 5XX
The load balancer itself generated the error.
Target 5XX
The backend application/target returned a 5XX.
This helps determine whether to investigate:
ALB configuration
or:
Application
75. Production architecture – complete example
For your EKS project, you can explain it like this:
                         Users
                           |
                           v
                    Route 53 DNS
                           |
                           v
                     CloudFront
                           |
                           v
                         WAF
                           |
                           v
                    Internet ALB
                           |
             +-------------+-------------+
             |                           |
       app.company.com             api.company.com
             |                           |
             v                           v
        Frontend TG                Backend TG
             |                           |
             v                           v
        React Pods                Java Pods
                                         |
                              +----------+----------+
                              |                     |
                              v                     v
                             RDS                   HDFS
For an interview, explain:
"Route 53 handles DNS resolution. The request reaches the ALB. AWS Load Balancer Controller creates and manages the ALB from Kubernetes Ingress definitions. Host or path rules determine the backend service. The ALB forwards traffic to healthy pod targets. The backend service then communicates with dependencies such as RDS or HDFS."

76. Rapid-fire interview questions
Q: What does Ingress do?
A: Defines HTTP/HTTPS routing rules.
Q: Does Ingress create an ALB?
A: The Ingress Controller can provision/configure the ALB.
Q: What creates the ALB in EKS?
A: AWS Load Balancer Controller.
Q: What is host-based routing?
A: Routing based on hostname.
Q: What is path-based routing?
A: Routing based on URL path.
Q: Can multiple services use one ALB?
A: Yes.
Q: Can multiple Ingress objects share an ALB?
A: Yes, using an IngressGroup with appropriate configuration.
Q: What is Route 53's responsibility?
A: DNS resolution and DNS-level routing policies.
Q: Does Route 53 route directly to pods?
A: Normally no; it resolves the application name to an endpoint such as an ALB.
Q: What is ALB IP target mode?
A: ALB registers pod IPs as targets.
Q: What causes 502?
A: Often a problem communicating with the backend target—such as port mismatch, unhealthy target, security/networking issues, or application connectivity problems.
Q: What causes 404?
A: Often no matching route/resource, such as a host/path mismatch.
Q: How do you enable HTTPS?
A: ACM certificate + ALB HTTPS listener.
Q: How do you redirect HTTP to HTTPS?
A: Configure an ALB HTTPS listener and HTTP-to-HTTPS redirect.
Q: What is IRSA?
A: IAM Roles for Service Accounts.
Q: Why is readiness important?
A: It prevents traffic from being sent to a pod that isn't ready.
Q: What is the difference between ALB and NLB?
A: ALB is primarily Layer 7 HTTP/HTTPS; NLB is primarily Layer 4 TCP/UDP/TLS.
77. The "Tell me your production experience" answer
If the interviewer asks:
"Explain your experience with EKS ingress."

You can answer:
"In our EKS platform, we use Kubernetes Ingress with the AWS Load Balancer Controller to expose HTTP/HTTPS applications through AWS Application Load Balancers. We use host-based and path-based routing depending on the application architecture. Route 53 provides DNS resolution, and ACM certificates are attached to the ALB for TLS termination.
For example, we can have the frontend exposed through one host and backend APIs through another host or path. The ALB listener rules route traffic to the appropriate target groups, and with IP target mode the ALB can register pod IPs directly.
For troubleshooting, I follow a layered approach: first DNS, then ALB listeners and rules, target-group health, Kubernetes Ingress, Service, EndpointSlices, pod readiness, application logs, security groups, and NetworkPolicies.
For production deployments, I also consider health checks, readiness probes, HTTPS redirects, WAF, monitoring, centralized logging, and appropriate separation between public and internal ALBs."

78. The most important architecture to remember
For interviews, remember this:
                    USER
                      |
                      v
                  ROUTE 53
                      |
                      v
                 +---------+
                 |   ALB   |
                 +---------+
                      |
              Listener :443
                      |
                Listener Rule
                /           \
               /             \
       Host / Path          Host / Path
            |                   |
            v                   v
       Target Group        Target Group
            |                   |
            v                   v
       K8s Service          K8s Service
            |                   |
            v                   v
          Pods                Pods
            |                   |
            +---------+---------+
                      |
                 Application
                      |
             +--------+--------+
             |                 |
             v                 v
            RDS              HDFS
And the responsibility of each component:
Component	Responsibility
Route 53	DNS
ACM	TLS certificates
WAF	Web traffic filtering
ALB	L7 load balancing
Listener	Port/protocol
Listener Rule	Host/path routing
Target Group	Backend targets + health checks
Ingress	Kubernetes routing declaration
AWS LB Controller	Reconciles Ingress to AWS resources
Service	Kubernetes service abstraction
EndpointSlice	Tracks service endpoints
Pod	Runs application
Security Group	Network access control
NetworkPolicy	Pod-level traffic control
CloudWatch	AWS monitoring
Prometheus	Kubernetes/application metrics
Grafana	Visualization


79. Final interview mindset
For a Senior AWS DevOps / Platform Engineer interview, don't answer only with definitions.
For almost every question, explain it using this pattern:
1. What is it?
        ↓
2. Why do we use it?
        ↓
3. Production architecture
        ↓
4. Example
        ↓
5. Failure scenario
        ↓
6. Troubleshooting
        ↓
7. Security / HA consideration
For example, if they ask:
"What is path-based routing?"

Don't stop at:
"It routes based on the URL path."

Instead:
Path-based routing
       |
       v
app.company.com/api
       |
       v
ALB listener
       |
       v
/api rule
       |
       v
backend target group
       |
       v
backend pods
Then explain:
"In production I would also verify rule priority, target health, Service endpoints, readiness probes, security groups, and application health if the route starts returning 4xx or 5xx."

That style demonstrates real operational experience, rather than memorized Kubernetes definitions.