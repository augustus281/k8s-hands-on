# AUTOMATED PLACEMENT

## Taints and Tolerations

### What are Taints and Tolerations in Kubernetes?

- **Taints** are used to prevent certain workloads (pods) from being scheduled on certain nodes.
- **Tolerations** are the way to allow some workloads to tolerate those taints and be scheduled on those nodes anyway.

### Checking if We Have a Running Cluster

You need to ensure your Kubernetes cluster is running. To do this, use the command:

```shell
kubectl get nodes
```

This command will list all the nodes in your cluster. Each node is essentially a worker machine in Kubernetes.

```shell
kubectl describe <cluster name>
```

By default, some nodes will have taints. We typically have two types of nodes:
- **Master Node (Control Plane)**: Responsible for managing the cluster.
- **Worker Node**: Where the application pods are deployed.

Describe the worker node to see its taints:

```shell
kubectl describe node <worker node name>
```
In most cases, the worker node will have **no taints** by default. This is because Kubernetes applies taints only to the 
**master node** to ensure that only management-related tasks are run there, while the rest of the workload is sent to worker nodes.

### Why is There No Taint on the Worker Node?

Taints are applied to the master node because it is dedicated to cluster management tasks. The worker nodes, however, are 
where your applications, pods, and other workloads are deployed. Since we want worker nodes to accept application pods 
by default, there are no taints on them unless we specifically add one.

### Applying a Taint to a Node
Let’s taint our worker node. This will prevent any pods from being scheduled on that node unless they have the right toleration.

#### 1. Get the list of nodes:
```shell
kubectl get nodes
```

#### 2. Apply a taint to the worker node using the following command:
```shell
kubectl taint node <worker node name> spray=mortein:NoSchedule
```
This command means that no pod will be scheduled on this node unless it has a toleration for the "spray=mortein" taint. 
Let’s verify that the taint has been applied:
```shell
kubectl describe node <worker node name>
```

### Running a Pod Without Toleration

Try running a pod on the tainted node:

```shell
kubectl run mosquito --image=nginx
```

Check if the pod was created:
```shell
kubectl get pods
```

### Creating a Pod with Toleration

Let’s create a pod that can tolerate the taint. First, create a directory for the configuration:
```shell
mkdir taint && cd taint
```

Generate the YAML file for the pod:
```shell
kubectl run bee --image=nginx --dry-run=client -o yaml > bee-pod.yaml
```

Edit the file to add the toleration. In the YAML file, under the spec section, add the following:
```yaml
tolerations:
  - key: "spray"
    operator: "Equal"
    value: "mortein"
    effect: "NoSchedule"
```

This tells Kubernetes that the pod can tolerate the taint on the worker node. Save the file, then apply it:
```shell
kubectl apply -f bee-pod.yaml
```

### Removing the Taint from the Master Node

Let’s remove the taint from the master (control plane) node to see if a pod can be scheduled on it.

#### 1. First, describe the master node:

```shell
kubectl describe node <control-plane name>
```

#### 2. Copy the taint value and remove it using:
```shell
kubectl taint node <control-plane name> <paste the copied taint value>-
```
The `-` at the end removes the taint. You’ll get a confirmation that the node is **untainted**.