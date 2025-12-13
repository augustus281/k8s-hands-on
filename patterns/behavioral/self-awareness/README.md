# SELF AWARENESS

Some applications need to be self-aware and require information about themselves. The _Self Awareness_ pattern describes the
Kubernetes Downward API that provides a simple mechanism for introspection and metadata injection to applications.

## Problem

Although most cloud-native applications on Kubernetes are stateless, they often need runtime information about themselves
and their environment, such as Pod name, IP, resource limits, labels, or annotations. This information is useful for dynamic
configuration, logging, monitoring, and clustering. Kubernetes provides the Downward API to expose this Pod metadata and runtime
details to containers in a simple and standardized way.

## Solution

In dynamic environments, resource metadata changes frequently and must be accessed at runtime. Cloud platforms provide metadata
services for this purpose (for example, AWS EC2 and ECS). Kubernetes offers a cleaner and more native solution through the **Downward API**, which
automatically injects Pod and runtime metadata into containers via environment variables or files, without requiring custom
services or manual configurations.

![img.png](images/application_introspection_mechanisms.png)

Kubernetes allows certain Pod metadata, such as labels and annotations, to be modified which a Pod is running. When this
metadata is exposed to containers through environment variables, the values are injected only at Pod startup and do not
change unless the Pod is restarted. This makes environment variables unsuitable for use cases where metadata needs to be updated
dynamically at runtime.

To address this limitation, Kubernetes provides the **Downward API through volumes**. With this approach, Pod metadata
is mounted into the container as files. For example, labels and annotations can be written into separate files inside
a mounted volume. These files are automatically updated by Kubernetes whenever the corresponding metadata changes while the Pod
is running.

In this setup, the `labels` file contains all Pod labels in a `key=value` format, one per line, and the annotations file
contains all Pod `annotations` in the same format. Unlike environment variables, these files reflect changes immediately
without requiring a Pod restart. However, Kubernetes only updates the file contents-the application itself must be able to detect
file changes and reload the updated data. If the application does not support dynamic reload, restarting the Pod may still
be necessary to apply the changes.

Overall, Downward API volumes are useful for applications that need access to dynamic runtime metadata, such as feature flags,
logging context, metrics labeling, or configuration driven by annotations, without disrupting the running Pod.