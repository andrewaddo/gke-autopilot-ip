# GKE Autopilot IP Range Configuration

This project contains the steps used to configure a VPC and subnet for a GKE Autopilot cluster with specific IP ranges.

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

### 4. Add Secondary Range for Pods
```bash
gcloud compute networks subnets update gke-autopilot-subnet \
    --add-secondary-ranges=gke-pods=10.102.184.0/21 \
    --region=us-central1 \
    --project=addo-argolis-demo
```

### 4.1 Add Additional Secondary Range for Small Pods
```bash
gcloud compute networks subnets update gke-autopilot-subnet \
    --add-secondary-ranges=podsmall=10.102.192.0/28 \
    --region=us-central1 \
    --project=addo-argolis-demo
```

## Cluster Creation

### 5. Create GKE Autopilot Cluster
Run the following command to create the Autopilot cluster using the pre-configured network and subnetwork ranges:

```bash
gcloud container clusters create-auto autopilot-cluster-verifyipusage \
    --region=us-central1 \
    --project=addo-argolis-demo \
    --network=gke-autopilot-vpc \
    --subnetwork=gke-autopilot-subnet \
    --cluster-secondary-range-name=gke-pods \
    --services-secondary-range-name=gke-services
```

## Verification

### 6. Verify Reserved IPs per Node
You can inspect the specific IP block (Pod CIDR) reserved for each node to see the current pod density Autopilot has selected:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{"Node: "}{.metadata.name}{"\nPod CIDR: "}{.spec.podCIDR}{"\n\n"}{end}'
```

*Note: In GKE 1.28+, Autopilot dynamically assigns these blocks based on workload. For example, a `/24` block (256 IPs) supports up to 128 Pods per node.*

#### Finding Nodes in the Google Cloud Console
In Autopilot, the "Nodes" tab is often hidden from the standard GUI navigation. To view your nodes in the Console:
1. Navigate to your cluster's **Overview** page.
2. In your browser's address bar, replace `/overview` at the end of the URL with `/nodes`.
3. Example: `.../clusters/details/us-central1/autopilot-cluster-verifyipusage/nodes?project=...`

## Advanced: Dynamic IP Allocation with Compute Classes

You can use a `ComputeClass` to explicitly control the pod density and IP reservation per node.

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

### 2. Create and Deploy Workloads
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

### 3. Verify Dynamic Slicing
Run the following command to see how Autopilot slices the secondary range differently based on the Compute Class:

```bash
kubectl get nodes -L cloud.google.com/compute-class -o jsonpath='{range .items[*]}{"Node: "}{.metadata.name}{"\nCompute Class: "}{.metadata.labels.cloud\.google\.com/compute-class}{"\nPod CIDR: "}{.spec.podCIDR}{"\n\n"}{end}'
```

#### Understanding the Output
Based on the **"Rule of Double"** (GKE reserves 2x the max pods in IPs), you will see different CIDR sizes:

*   **Low-Density (16 Pods):** Reserved a **/27** (32 IPs). This is highly efficient for small, isolated workloads.
    *   *Example Output:* `Pod CIDR: 10.102.188.32/27`
*   **High-Density (128 Pods):** Reserved a **/24** (256 IPs). This is better for packing many small microservices onto fewer nodes.
    *   *Example Output:* `Pod CIDR: 10.102.185.0/24`

This demonstrates that Autopilot does not use a fixed "one-size-fits-all" IP block, but instead "slices" your secondary IP range dynamically based on the requirements of your `ComputeClass` or workload density.

### Key Takeaway: Heterogeneous IP Reservation
In a single GKE Autopilot cluster (version 1.28+), you can have **different nodes with different max Pod limits and different IP CIDR block sizes** running simultaneously.

*   **Node-specific limits:** The `maxPodsPerNode` is locked in at the time a node is provisioned.
*   **VPC Flexibility:** The VPC correctly routes traffic to these varying "slices" (e.g., a `/24` next to a `/27`) within your secondary range.
*   **IP Optimization:** This allows you to mix high-density microservices with low-density, resource-heavy workloads in the same cluster without wasting IP addresses or hitting artificial pod-per-node limits.
