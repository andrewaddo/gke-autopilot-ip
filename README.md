# GKE Autopilot IP Range Configuration

This project contains the steps used to configure a VPC and subnet for a GKE Autopilot cluster with specific IP ranges and demonstrates dynamic IP allocation.

## Networking Setup

### 1. Create VPC Network
```bash
gcloud compute networks create gke-autopilot-vpc \
    --subnet-mode=custom \
    --project=addo-argolis-demo
```

### 2. Create Subnet with Primary Range (Nodes)
The primary range is used for the GKE nodes.
```bash
gcloud compute networks subnets create gke-autopilot-subnet \
    --network=gke-autopilot-vpc \
    --range=10.102.164.0/25 \
    --region=us-central1 \
    --project=addo-argolis-demo
```

### 3. Add Secondary Range for Services
```bash
gcloud compute networks subnets update gke-autopilot-subnet \
    --add-secondary-ranges=gke-services=10.102.164.128/25 \
    --region=us-central1 \
    --project=addo-argolis-demo
```

### 4. Add Secondary Range for Pods (Minimum /24 for Autopilot)
Autopilot requires at least a `/24` (256 IPs) for the primary Pod secondary range.
```bash
gcloud compute networks subnets update gke-autopilot-subnet \
    --add-secondary-ranges=podsmall=10.102.192.0/24 \
    --region=us-central1 \
    --project=addo-argolis-demo
```

## Cluster Creation

### 5. Create GKE Autopilot Cluster
Run the following command to create the Autopilot cluster using the `podsmall` range:

```bash
gcloud container clusters create-auto autopilot-cluster-small \
    --region=us-central1 \
    --project=addo-argolis-demo \
    --network=gke-autopilot-vpc \
    --subnetwork=gke-autopilot-subnet \
    --cluster-secondary-range-name=podsmall \
    --services-secondary-range-name=gke-services
```

## Verification & Insights

### 6. Verify Reserved IPs per Node
In Autopilot (v1.28+), IP allocation is dynamic. Each node is assigned a "slice" of the secondary range based on its workload density.

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{"Node: "}{.metadata.name}{"\nPod CIDR: "}{.spec.podCIDR}{"\n\n"}{end}'
```

#### Finding Nodes in the Google Cloud Console
In Autopilot, the "Nodes" tab is often hidden from the standard GUI navigation. To view your nodes in the Console:
1. Navigate to your cluster's **Overview** page.
2. In your browser's address bar, replace `/overview` at the end of the URL with `/nodes`.

## Advanced: Heterogeneous IP Reservation
By using a `ComputeClass`, you can mix nodes with different Pod densities in the same cluster.

### 1. Create and Apply Compute Classes
Define classes with different `maxPodsPerNode` values in a file named `compute-classes.yaml`:

```yaml
apiVersion: cloud.google.com/v1
kind: ComputeClass
metadata:
  name: low-density
spec:
  autopilot:
    enabled: true
  priorities:
  - maxPodsPerNode: 16
    machineFamily: e2
---
apiVersion: cloud.google.com/v1
kind: ComputeClass
metadata:
  name: high-density
spec:
  autopilot:
    enabled: true
  priorities:
  - maxPodsPerNode: 128
    machineFamily: e2
```

Apply them to the cluster:
```bash
kubectl apply -f compute-classes.yaml
```

### 2. Deploy Workloads
Target the classes using the `nodeSelector` in a file named `workloads.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-low-density
spec:
  replicas: 5
  selector:
    matchLabels:
      app: low-density
  template:
    metadata:
      labels:
        app: low-density
    spec:
      nodeSelector:
        cloud.google.com/compute-class: low-density
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-high-density
spec:
  replicas: 5
  selector:
    matchLabels:
      app: high-density
  template:
    metadata:
      labels:
        app: high-density
    spec:
      nodeSelector:
        cloud.google.com/compute-class: high-density
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

Apply the workloads:
```bash
kubectl apply -f workloads.yaml
```

### 3. Verify Dynamic Slicing Results
*   **Low-Density (16 Pods):** Reserves a **/27** (32 IPs).
*   **Default/High-Density:** Reserves a **/24** (256 IPs).

This "slicing" behavior allows for high IP efficiency while maintaining the management simplicity of Autopilot.

## IP Exhaustion Experiment

A test was conducted to observe the behavior of Autopilot when restricted to the minimum allowed Pod secondary range of `/24` (256 IPs).

### 1. Setup
*   **Secondary Range:** `podsmall` (`10.102.192.0/24`).
*   **Initial Node:** A single default node was provisioned, which Autopilot assigned the **entire `/24` block**.

### 2. The Test
Workloads were deployed targeting custom `ComputeClasses`. Since the existing node did not match the class labels, Autopilot attempted to scale up new nodes.

### 3. Observed Result
The pods remained in a `Pending` state with the following cluster event:
```text
Warning  FailedScaleUp  pod/...  Node scale up in zones ... failed: IP space exhausted.
```

### 4. Conclusion
Even though the actual IP utilization was low (only a few pods running), the **IP Reservation** logic of Autopilot allowed a single node to "hog" the entire `/24` range. This confirms that:
*   A `/24` is the absolute minimum for Autopilot but can lead to immediate exhaustion if nodes are assigned large slices.
*   IP Exhaustion in GKE is a function of **Node Reservation (CIDR slicing)** rather than individual Pod IP usage.
