# ELASTIC SCALE

The Elastic Scale pattern covers application scaling in multiple dimensions: horizontal scaling by adapting the number of Pod
replicas, vertical scaling by adapting resource requirements for Pods, and scaling the cluster itself by changing the number of cluster nodes.

## Problem

Kubernetes manages distributed applications by maintaining a declarative desired state, but determining the right resource usage and replica counts is challenging due to changing workloads. To address this, Kubernetes allows resources, replicas, and even cluster size to be adjusted dynamically, either manually or automatically. By observing real-time load and capacity metrics, Kubernetes can analyze the current state and scale itself to meet performance goats. This adaptive approach enables
systems to respond to actual usage rather than predictions, making them more efficient and resilient.

## Solution

There are two main approaches: horizontal and vertical. In Kubernetes, horizontal scaling means increasing or decreasing the number of Pod replicas, while vertical scaling allocates more resources to existing containers. Although the concepts are simple, configuring autoscaling correctly in a shared cloud environment is complex and often requires experimentation. Kubernetes offers multiple features and techniques to help determine the most effective scaling strategy for applications.

### Manual Horizontal Scaling

Manual scale relies on a human operator issuing commands to Kubernetes to adjust application capacity. It is useful when autoscaling is unavailable or during gradual tuning to find on optimal configuration for workloads with slowly changing demand. A key advantage is that it supports anticipatory scaling, allowing teams to scale ahead of expected load increases
instead of reacting after demand rises. In Kubernetes, manual scaling can be performed in two main styles.

#### Imperative scaling

A controller such as ReplicaSet is responsible for making sure a specific number of Pod instances are always up and running. Thus, scaling a Pod is as trivially simple as changing the number of desired replicas.

```shell
kubectl scale random-generator --replicas=4
```

After such a change, the ReplicaSet could either create additional Pods to scale up, or if there are more Pods than desired, delete them to scale down.

#### Declarative scaling

Using the kubectl scale command is quick and convenient for emergency reactions, but it does not persist changes outside the cluster. Since Kubernetes configurations are typically stored in source control, recreating resources from their original definitions can revert the replica count. This leads to configuration drift between the cluster state and version-controlled
manifests. To avoid this, it is best practice to update the desired replica count declaratively in the source definition and apply it to the cluster.

Kubernetes allows scaling resources like ReplicaSets, Deployments, and StatefulSets, but StatefulSets behave differently when using persistent storage by retaining PVCs during scale-down. Jobs can also be scaled, using `.spec.parallelism` instead of
`.spec.replicas`, with a similar effect of increasing processing capacity. Resource fields are referenced using JSON path
notation such as `.spec.replicas`. Both imperative and declarative manual scaling rely on human decisions and are not suitable for frequently changing workloads, which motivates automated scaling mechanisms.

### Horizontal Pod Autoscaling

Many cloud-native workloads experience __dynamic and unpredictable traffic patterns__, which makes it impractical to rely on a fixed number of Pods. Kubernetes addresses this challenge through __autoscaling__, allowing applications to automatically adapt their capacity based on actual runtime metrics rather than static assumptions. The most common and straightforward autoscaling mechanism in Kubernetes is the __HorizontalPodAutoscaler (HPA)__, which scales applications by adjusting the __number of Pod replicas.__

#### Prerequisites for Using HPA

For an HPA to function correctly, several conditions must be met:

__1. CPU or resource requests must be defined__

Each container in the target Pod must declare resource requests, such as:

```yaml
resources:
  requests:
    cpu: 200m
```

HPA uses these request values as the baseline for calculating utilization percentages.

__2. Metrics Server must be installed__

The Metrics Server is a cluster-wide component that aggregates resource usage data (CPU and memory) from nodes and Pods. Without it, HPA cannot retrieve the metrics needed for scaling decisions.

#### How HPA Works Internally

The HPA controller runs continuously as part of Kubernetes' control plane and follows a reconciliation loop similar to other controllers:

__1. Metric Collection__

The controller retrieves metrics for the target Pods via the Kubernetes __Metrics API__. 

- Resource metrics (CPU, memory) come from the Metrics Server.
- Custom or external metrics (if configured) come from the Custom Metrics API.

__2. Replica Calculation__

HPA calculates the desired number of replicas using the current metric values and the configured target. A simplified version
of the formula is:

$desiredReplicas = ceil(currentReplicas * (currentMetricsValue / desiredMetricValue)) $

**For example:**

- Current replicas: 1
- Current CPU usage: 90%
- Target CPU utilization: 50%

The calculation becomes:

```text
1 * (90 / 50) = 1.8 -> 2 replicas
```

In practice, Kubernetes applies additional logic to smooth out fluctuations, handle multiple Pods, and avoid repid scaling (thrashing).

__3. Scaling Decision__

If multiple metrics are defined (e.g., CPU and memory), HPA evaluates each metric independently and selects the **highest proposed replica count** to ensure all constraints are satisfied.

__4. Applying. the Desired State__

Once the desired replica count is computed, HPA updates the target resource (such as a Deployment), and Kubernetes creates
or removes Pods to reconcile the actual state with the desired state.

#### Supported Target Resources

HPA can scale any Kubernetes resource that exposes the __scale subresource__, including:

- Deployments
- ReplicaSets
- StatefulSets

However, __best practice is to attach the HPA to a Deployment__, not directly to a ReplicaSet. Deployments create new ReplicaSets during rolling updates, and HPAs attached to ReplicaSets are not automatically transferred, which can lead to lost autoscaling behavior.

![1766424440061](image/README/1766424440061.png)

#### Types of Metrics Used for Autoscaling

#### 1. Standard (Resource) Metrics

Standard metrics are the simplest and most commonly used metrics in HPA. These metrics are defined with:

```bash
.spec.metrics.resource[].type = Resource
```

They represent resource usage metrics such as CPU and memory and are available for any container in any Kubernetes cluster using the same metric names.

Key characteristics:

- Metrics can be expressed at:

    + A __percentage__ of resource utilization (e.g., 50% CPU usage). 
    + An __absolute value__ (e.g., 200m CPU).

- All calculations are based on __resource requests__, not resource limits
- These metrics are typically provided by:

    + Metrics Server
    + Heapster (legacy)

Because they are generic, easy to configure, and widely supported, resource metrics are usually the first choice
when setting up autoscaling.

#### 2. Custom Metrics

Custom metrics allow autoscaling based on __application-specific signals__ rather than just CPU or memory. These metrics use:

```python
.spec.metrics.resource[*].type = Object or Pod
```

There are two subtypes:

- **Pod metrics**: metrics associated with individual Pods (e.g., requests per Pod).
- **Object metrics**: metrics associated with other Kubernetes objects (e.g., queue length on a Service).

Custom metrics require a more advanced monitoring setup and are exposed via the aggregated API Server at:

```lua
custom.metrics.k8s.io
```

They are typically provided through adapters such as:

- Prometheus Adapter
- Datadog
- Azure Monitor
- Google Stackdriver

Custom metrics enable more intelligent autoscaling but increase operational complexity.

#### 3. External Metrics

External metrics represent data __outside the Kubernetes cluster__. These metrics are useful when autoscaling depends on
external systems rather than internal resource usage.

__Example:__ Scaling consumer Pods based on the depth of a cloud-based message queue (e.g., SQS, Pub/Sub).

Like custom metrics, external metrics require a dedicated adapter that integrate external systems into Kubernetes autoscaling framework.

#### Key Considerations When Configuring HPA

Autoscaling is not "set and forget". Achieving reliable behavior requires experimentation, tuning, and a solid understanding
of application behavior.

#### 1. Metric Selection

Choosing the right metrics is one of the most critical decisions in autoscaling.

A good autoscaling metric must:

- Have a **direct and preferably linear operation** with the number of Pods.
- Decrease per Pod when the number of Pods increases

Good examples:

- CPU usage
- Requests per second (QPS)
- Queue length per Pod

Problematic example:

- Memory usage.

Memory is often a poor autoscaling metric because increasing the number of Pods does not necessarily reduce memory usage per Pod. If memory is not released or redistributed, the HPA may continuously scale up until it reaches the maximum replica limit, leading to inefficient or incorrect behavior.

#### 2. Preventing Thrashing

Thrashing occurs when the number of replicas rapidly fluctuates due to unstable or noisy metrics.

To prevent this, HPA uses several stabilization techniques:

- Ignores high CPU samples during Pod startup
- Applies smoothing logic when scaling up
- Uses a configurable stabilization window when scaling down
- Chooses the __highest recommended replica count__ within that window to avoid premature scale-down

These mechanisms help ensure autoscaling decisions are stable and predictable.

#### 3. Delayed Reaction

Autoscaling decisions involve multiple components and steps, which introduce unavoidable delays:

1. cAdvisor collects metrics on each node

2. Kubelet exposes metrics

3. Metrics Server scrapes the data

4. HPA controller evaluates metrics periodically

5. Scaling formulas add intentional delay to avoid thrashing

Increasing delays improves stability but reduces responsiveness. Reducing delays improves reaction time but increases platform load and the risk of oscillations. Finding the right balance is an iterative tuning process.

### Vertical Pod Autoscaling