AWS Load Balancer Controller – Simple breakdown
1. Kubernetes resources (inside EKS)
These are created or managed inside Kubernetes:
1. Ingress – Defines how external traffic reaches your application.
2. Service – Exposes your application pods.
3. TargetGroupBinding – Connects a Kubernetes Service to an AWS Target Group.
4. Ingress annotations – Define AWS-specific ALB configurations.
2. AWS resources (inside AWS account)
These are actual AWS infrastructure resources:
1. Application Load Balancer (ALB) – Receives incoming HTTP/HTTPS traffic.
2. Target Group – Contains the registered application targets (pod IPs or instances).
3. Listener – Listens on ports such as 80 and 443.
4. Listener Rules – Define routing based on hostnames and URL paths.
5. Security Groups – Control network access to the ALB.
3. What does AWS Load Balancer Controller do?







linkedin.com


The controller runs as a Pod inside EKS.

Its job is:
1. Watches Kubernetes Ingress and Service resources.
2. Reads ALB annotations and configurations.
3. Uses AWS APIs with IAM permissions (IRSA or EKS Pod Identity).
4. Creates or configures ALB, Target Groups, Listeners and Listener Rules.
5. Registers application targets in Target Groups.
6. Continuously reconciles AWS infrastructure with Kubernetes configuration.
4. Complete traffic flow
User / Browser
      |
      v
AWS ALB
      |
      v
Listener (80/443)
      |
      v
Listener Rules
      |
      v
Target Group
      |
      v
Kubernetes Service
      |
      v
Application Pods


Interview answer
"The AWS Load Balancer Controller runs inside EKS. It watches Kubernetes Ingress and Service resources, reads the AWS-specific annotations, and communicates with AWS APIs using IAM permissions to provision and manage ALBs, Target Groups, Listeners and routing rules. This automates the integration between Kubernetes workloads and AWS load balancing."



CRD (Definition)
Already installed by the AWS Load Balancer Controller. It defines the TargetGroupBinding resource type.
kubectl get crd targetgroupbindings.elbv2.k8s.aws


CR (Custom Resource)
This entire YAML is the CR:
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: employee-tgb
spec:
  serviceRef:
    name: employee-service
    port: 8080
  targetGroupARN: <target-group-arn>


Here:
- kind: TargetGroupBinding → Resource type.
- metadata.name: employee-tgb → Name of the CR.
- Entire YAML → CR object.

Remember
Term	Meaning
CRD	Defines TargetGroupBinding
CR	Entire employee-tgb YAML object
CR name	employee-tgb
Controller	Watches and processes the CR

Simple analogy: CRD is the form design, CR is the filled form, and the controller is the person processing it.


1. What is a CRD?
CRD stands for Custom Resource Definition.

Imagine Kubernetes comes with some built-in resources:
- Pod
- Deployment
- Service
- ConfigMap
- Secret
- Ingress
Now imagine you install AWS Load Balancer Controller in your EKS cluster.

You want Kubernetes to understand AWS-specific resources such as:
- TargetGroupBinding
- Ingress-related AWS configurations
But Kubernetes doesn't understand every AWS-specific resource by default.

So we introduce a CRD.

Simple definition:

A CRD teaches Kubernetes a new resource type that it didn't understand before.

Think of it like installing a new vocabulary into Kubernetes.
2. Real-time example – AWS Load Balancer Controller







linkedin.com


Suppose you want to create an ALB for your application.

Normally, Kubernetes understands Deployments, Services and Ingress.

But AWS-specific configurations need additional knowledge.

We install AWS Load Balancer Controller.

During installation, its CRDs and controller components are made available.

Now Kubernetes can understand AWS-specific custom resources such as TargetGroupBinding.

For example:
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: employee-tgb
spec:
  serviceRef:
    name: employee-service
    port: 8080
  targetGroupARN: arn:aws:elasticloadbalancing:...


Here:
- apiVersion tells Kubernetes which API version to use.
- kind tells Kubernetes which resource type this is.
- TargetGroupBinding is a custom resource type defined through a CRD.
Without the corresponding CRD, Kubernetes cannot accept this custom resource.
3. What is a Controller?
Now the interesting part.

You created a CRD. Kubernetes now understands a new resource type.

But who actually performs the work?

The Controller!

A controller is like a manager who continuously watches Kubernetes resources and takes action to make the actual environment match the desired configuration.

Think about this example:

You create a Deployment:
replicas: 3


You are telling Kubernetes:

"I need three application pods running."

The Deployment Controller notices this requirement and works toward maintaining three replicas.

If one pod crashes, Kubernetes creates another to maintain the desired count.

That is the controller's job.
4. CRD vs Custom Resource vs Controller
This is where many people get confused.

Let's separate the three:
Component	Meaning	Simple example
CRD	Defines a new resource type	New form/template
Custom Resource (CR)	An actual object created using that type	Filled form
Controller	Watches the object and performs the required action	Manager who processes the form

Think of a simple analogy:
- CRD = Application form design.
- Custom Resource = Filled application form.
- Controller = Officer who processes the application.
5. How does everything work together?







medium.com


Let's say you install an operator or controller.

The flow is:
1. Install the CRD.
2. Kubernetes API server learns the new resource type.
3. Create a Custom Resource (CR) using YAML.
4. Controller continuously watches that resource.
5. Controller checks the desired configuration.
6. Controller performs the required action.
7. Controller keeps checking and reconciling the actual state.
This continuous process is called the Reconciliation Loop.
6. Understand using your EKS tools
Let's take the tools you use in your EKS environment.
Tool	CRD / Custom Resource example	What does the controller do?
AWS Load Balancer Controller	TargetGroupBinding	Connects Kubernetes services with AWS Target Groups
Argo CD	Application	Watches Git repository and synchronizes Kubernetes resources
External Secrets Operator	ExternalSecret, SecretStore, ClusterSecretStore	Fetches secrets from AWS Secrets Manager and creates Kubernetes Secrets
Prometheus Operator	ServiceMonitor, PrometheusRule	Configures Prometheus scraping and alerting
Grafana Operator	GrafanaDashboard (if using the operator)	Manages Grafana dashboards

Important: Not every Kubernetes tool requires a CRD. For example, Prometheus and Grafana can also be installed and configured without their operators.
7. Commands to understand CRDs practically
After installing the AWS Load Balancer Controller:

List all CRDs:
kubectl get crds


Find AWS Load Balancer CRDs:
kubectl get crds | grep elbv2


You may see:
targetgroupbindings.elbv2.k8s.aws


Describe a CRD:
kubectl describe crd targetgroupbindings.elbv2.k8s.aws


List actual custom resources:
kubectl get targetgroupbindings -A


Check the controller pods:
kubectl get pods -n kube-system


Check controller logs:
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller



8. Most important interview answer
If the interviewer asks:

What is a CRD and what is a Controller in Kubernetes?

Answer:

"A CRD extends the Kubernetes API by introducing new resource types that Kubernetes doesn't support by default. A Custom Resource is an actual object created using that definition. A Controller watches these custom resources and continuously reconciles the actual environment with the desired configuration. For example, the AWS Load Balancer Controller watches Kubernetes Ingress resources and provisions or configures AWS load balancers."