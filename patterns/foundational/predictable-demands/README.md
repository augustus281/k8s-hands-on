# Predictable Demands

## Problem

Kubernetes can run applications in any language that can be containerized, but different languages have varying resource
needs. While compiled languages often use less memory and run faster than interpreted or just-in-time languages, actual resource 
consumption depends more on the application's domain, business logic, and implementation. Developers, through testing, best 
understand the source requirements of service, which can be constant or spiky. Some services need persistent storage or
specific host ports. Defining these characteristics is essential for managing cloud-native applications effectively.

## Solution

Understanding a container's runtime requirements is crucial for two main reasons.

- First, with all dependencies and resource demands defined, Kubernetes can place containers intelligently for optimal
hardware utilization, ensuring smooth coexistence of multiple processes with varying priorities.
- Second, knowing resource profiles supports capacity planning: by analyzing service demands and their numbers, we can design
cost-effective host configurations to meet cluster-wide needs. Both resource profiling and capacity planning are essential
for long-term cluster management. Before examining resource profiles, it is important to understand how to declare runtime
dependencies.

