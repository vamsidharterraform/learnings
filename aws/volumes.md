1. Static vs dynamic provisioning
Static provisioning
Volume created upfront
1. Create an EBS volume in AWS first.
2. Create a Kubernetes PersistentVolume (PV) that represents that existing EBS volume.
3. Create a PersistentVolumeClaim (PVC) that binds to the PV.
4. Reference the PVC in your Deployment YAML.
Existing EBS volume → PV → PVC → Pod


Dynamic provisioning
Volume created automatically
1. Create a StorageClass that defines how storage should be provisioned.
2. Create a PVC requesting storage.
3. The EBS CSI driver provisions an EBS volume dynamically.
4. Kubernetes creates/binds the PV, and your Pod uses the PVC.
StorageClass → PVC → EBS volume + PV → Pod


2. Static provisioning example
The EBS volume already exists in AWS, for example vol-0123456789abcdef0, in the same Availability Zone as the worker node that will use it.
Step 1 — Create a PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: existing-ebs-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0
    fsType: ext4


Step 2 — Create a PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: existing-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  volumeName: existing-ebs-pv
  resources:
    requests:
      storage: 10Gi


Step 3 — Reference the PVC in your Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: nginx:latest
          volumeMounts:
            - name: app-storage
              mountPath: /data
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: existing-ebs-pvc


The volume must be formatted and compatible with the requested filesystem. The example assumes an existing, suitable EBS volume.
3. Dynamic provisioning example
In EKS, the EBS CSI driver must be installed and configured with the required IAM permissions.
StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3


PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 10Gi


Use the same app-pvc reference in the Deployment's volumes section as shown in the previous example.
When a Pod uses the PVC, Kubernetes and the EBS CSI driver arrange provisioning and binding. With WaitForFirstConsumer, provisioning waits until the Pod's scheduling requirements help determine the appropriate Availability Zone.
4. Important interview points
- Static: The EBS volume already exists; you define a PV and bind a PVC to it.
- Dynamic: You create a StorageClass and PVC; the CSI driver provisions the EBS volume.
- EBS CSI driver: Connects Kubernetes persistent storage operations to AWS EBS.
- Availability Zone: An EBS volume can only attach to an EC2 instance in the same AZ.
- Access mode: EBS ReadWriteOnce generally supports read-write mounting by one node at a time; it does not mean only one Pod can use the volume.
- Deployment replicas: Multiple replicas may not be able to share one EBS volume as expected. For shared storage, consider EFS when its access characteristics fit your application.
Interview-ready answer: “In static provisioning, we create an EBS volume beforehand, define a PersistentVolume referencing it, and bind a PVC to that PV. In dynamic provisioning, we define a StorageClass and PVC, and the EBS CSI driver creates the volume automatically. The Pod references the PVC in its Deployment YAML in both cases.”