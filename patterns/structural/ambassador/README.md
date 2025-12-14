# AMBASSADOR

The Ambassador pattern is a specialized _Sidecar_ responsible for hiding complexity and providing a unified interface for
accessing services outside the Pod.

## Problem

Containerized applications often need to communicate with external services that are difficult to access reliably due to
dynamic addresses, load balancing, protocol differences, or data format complexity. Embedding this access logic directly
into the application increases responsibility and reduces reusability. The Ambassador pattern solves this by isolating all
external service access logic into a separate container, allowing the main application to focus solely on business functionality.

## Solution

The Ambassador pattern is used to isolate and abstract the complexity of accessing external services from an application
container. Instead of embedding service discovery, sharding logic, retries, timeout, or protocol-specific handling inside
the main application, a dedicated Ambassador container handles these concerns and exposes a simple local interface (usually via localhost).
This keeps the application single-purpose, easier to reuse, and independent of environment-specific configurations. By decoupling
external service access, the Ambassador pattern improves maintainability, flexibility, and resilience in containerized and Kubernetes-based systems.

![img.png](ambassador.png)