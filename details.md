EKS – ALB, Argo CD, ESO, Prometheus & Grafana
Interview quick revision
For each tool, remember these four things:
1. How do we install it?
2. How do we authenticate it?
3. What CRDs does it use?
4. How does it work with EKS?

1. AWS Load Balancer Controller (ALB)
1. Installation
- Create an IAM policy with required ELB permissions.
- Create an IAM role and configure IRSA using the EKS OIDC provider.
- Create a Kubernetes ServiceAccount and attach the IAM role ARN.
- Install AWS Load Balancer Controller using Helm.
- Deploy the application, Service and Ingress.
2. Authentication
- Uses IRSA (IAM Roles for Service Accounts).
- IRSA allows the controller Pod to authenticate with AWS APIs.
- It uses those permissions to create and manage ALBs, listeners and Target Groups.
3. CRDs
- TargetGroupBinding – connects Kubernetes services to AWS Target Groups.
- Ingress is a built-in Kubernetes resource, not a CRD.
4. How it works
- Controller watches the Ingress resource.
- It communicates with AWS ELB APIs using its IAM role.
- AWS creates the ALB and Target Groups.
- Traffic reaches the application Pods through the configured Target Groups.

2. Argo CD
1. Installation
- Install Argo CD in the EKS cluster using Helm or official manifests.
- It creates components such as API Server, Repo Server and Application Controller.
- Expose Argo CD UI using Ingress or LoadBalancer.
- Connect the Git repository containing Kubernetes manifests or Helm charts.
2. Authentication
- Git authentication: SSH key, Git token or HTTPS credentials.
- User authentication: Local accounts or SSO (OIDC).
- Kubernetes authentication: Uses Kubernetes ServiceAccounts and RBAC to manage cluster resources.
- Normally, IRSA is not required for basic GitOps operations.
3. CRDs
- Application – defines which Git repository and path to deploy.
- AppProject – controls project-level access and deployment permissions.
- ApplicationSet – generates multiple Argo CD Applications.
4. How it works
- Argo CD continuously monitors the Git repository.
- It compares Git's desired state with the actual EKS cluster state.
- If there is a difference, it synchronizes the resources.
- With auto-sync enabled, it can automatically deploy changes.

3. External Secrets Operator (ESO)
1. Installation
- Install External Secrets Operator using Helm.
- It installs the required CRDs and deploys the controller.
- Create an IAM policy with AWS Secrets Manager permissions.
- Create an IAM role and configure IRSA.
- Attach that role to the ESO ServiceAccount.
2. Authentication
- Uses IRSA to authenticate with AWS Secrets Manager.
- The ESO controller Pod assumes the IAM role.
- IAM permissions allow it to read secrets from AWS Secrets Manager.
3. CRDs
- ExternalSecret – specifies which secret to fetch from AWS.
- SecretStore – defines the secret provider configuration for a namespace.
- ClusterSecretStore – defines a secret provider configuration across namespaces.
4. How it works
- Create a secret in AWS Secrets Manager.
- Create a SecretStore or ClusterSecretStore.
- Create an ExternalSecret YAML referencing the AWS secret.
- ESO fetches the secret using IRSA.
- ESO creates a Kubernetes Secret.
- The application Pod consumes that Kubernetes Secret.

4. Prometheus
1. Installation
- Install Prometheus using Helm.
- Commonly, we install the kube-prometheus-stack, which includes Prometheus Operator, Prometheus and related monitoring components.
- Deploy exporters such as Node Exporter and configure scraping targets.
2. Authentication
- Uses Kubernetes ServiceAccounts and RBAC to discover Pods, Services and endpoints.
- For scraping secured endpoints, configure the required credentials, TLS or authentication.
- Normally, IRSA is not required for standard Kubernetes metrics collection.
3. CRDs
When using Prometheus Operator:
- ServiceMonitor – tells Prometheus which Services to monitor.
- PodMonitor – tells Prometheus which Pods to monitor.
- PrometheusRule – defines alerting and recording rules.
- Prometheus – defines the Prometheus instance configuration.
- Alertmanager – defines Alertmanager configuration.
4. How it works
- Prometheus discovers monitoring targets.
- It scrapes metrics from applications, Nodes and Pods.
- It stores the metrics in its time-series database.
- We use PromQL to query metrics.
- Alerts can be sent to Alertmanager.

5. Grafana
1. Installation
- Install Grafana using Helm or as part of kube-prometheus-stack.
- Expose Grafana using Ingress or LoadBalancer.
- Configure Prometheus as a data source.
2. Authentication
- Grafana supports local username/password authentication.
- It also supports SSO integrations such as OIDC.
- Kubernetes ServiceAccount/RBAC can be used for Kubernetes-related access.
- IRSA is not required for a normal Prometheus-to-Grafana setup.
3. CRDs
- Grafana itself does not have built-in Kubernetes CRDs in a standard Helm installation.
- If using Grafana Operator, it introduces CRDs such as:
  - Grafana
  - GrafanaDashboard
  - GrafanaDatasource
4. How it works
- Grafana connects to Prometheus as a data source.
- It queries metrics using PromQL.
- We create dashboards for CPU, memory, Pods, Nodes and application metrics.
- We configure alerts and visualize monitoring data.

Final interview revision – one-line answers
Tool	Authentication	Important CRDs
ALB Controller	IRSA / Pod Identity	TargetGroupBinding
Argo CD	Git credentials, SSO, Kubernetes RBAC	Application, AppProject, ApplicationSet
ESO	IRSA / Pod Identity	ExternalSecret, SecretStore, ClusterSecretStore
Prometheus Operator	Kubernetes RBAC	ServiceMonitor, PodMonitor, PrometheusRule
Grafana Operator	Grafana authentication / Kubernetes RBAC	Grafana, GrafanaDashboard, GrafanaDatasource

Important interview questions
Q1. Does every CRD require IRSA?

No. CRDs extend the Kubernetes API. IRSA is needed only when a controller needs to access AWS APIs. Kubernetes API access is generally controlled through RBAC.

Q2. Does installing a CRD perform any action?

No. The CRD registers a new resource type. The controller watches the custom resource and performs the actual action.

Q3. What is the difference between IRSA and RBAC?
- IRSA: Provides AWS IAM permissions to a Kubernetes workload.
- RBAC: Controls what a user or ServiceAccount can do inside Kubernetes.
Q4. What is the difference between Prometheus and Grafana?
- Prometheus: Collects and stores metrics.
- Grafana: Queries and visualizes metrics through dashboards.
Remember this interview concept: Installation deploys the controller, authentication gives it the required permissions, CRDs define custom resources, and the controller watches those resources and performs the actual work.