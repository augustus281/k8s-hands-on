# SINGLETON SERVICE

## Problem

Kubernetes makes it easy to scale applications by running multiple Pod replicas, which increases throughput and ensures
high availability. A Service distributes requests across these healthy replicas.

However, some workloads must run **only a single instance** at a time. Running multiple replicas can cause problems such as:

- **Duplicate scheduled tasks** when each replica triggers the same periodic job.
- **Conflicts in polling resources** (e.g., filesystem, database) where only one instance should perform the polling and processing.
- **Incorrect message ordering** when a message consumer must process events in a strict, single-threaded sequence.

In these scenarios, scaling multiple Pods can break the application's logic, so the service must remain a **singleton**.

## Solution

When an application runs multiple Pods (replicas), Kubernetes creates an **active-active** model, meaning all Pods operate concurrently.
This work well for most microservices.

However, sometimes we need an **active-passive (or master-slave)** model, where:

- only one Pod is allowed to be active, and
- the remaining Pods (if any) must stay passive or inactive.

### Out-of-Application Locking

Out-of-application locking is a strategy where **locking and concurrency control are handled outside the application itself**, using an external coordination 
system such as **Redis, Zookeeper, Consul, ETCD**, or even **database-level locks**.

In distributed systems - especially microservices and containerized environments like Kubernetes - your application often runs in **multiple instances**.
Traditional in-memory locks (e.g., mutexes) only protect resources **within a single instance**, which means they cannot prevent race conditions 
across multiple replicas.

To solve this, out-of-application locking provides centralized, shared locking mechanism that all instances can access. This ensures that only one instance
performs a critical operation at a time.

**Why it's important:**

- **Prevents race conditions across replicas** (e.g., multiple instances writing the same record).
- **Ensures consistency** in distributed workflows or scheduled tasks.
- **Coordinates long-running or exclusive operations** across an entire cluster.
- **Avoids duplicate job execution** when tasks run in horizontally scaled environments.

**How it works**

Your application requests a lock from an external system:

1. If the lock is available -> the instance acquires it and performs the action.
2. If the lock is taken -> the instance waits, retries, or skips the action.
3. When done, the instance release the lock (or it expires automatically).

### In-Application Locking

In distributed systems, you sometimes need to make sure that only one service instance is active at a time, even though 
multiple replicas may be running. A common solution is to use a distributed lock.

When an instance starts, it tries to acquire the lock:

- If it succeeds → it becomes the active instance.

- If it fails → it becomes passive and keeps retrying until the lock is released.

This pattern is widely used in high-availability systems. For example, Apache ActiveMQ uses a shared lock to ensure that only 
one broker is active, while others remain standby. This guarantees active/passive failover.

This mechanism resembles the classic Singleton pattern, but applied at the cluster level rather than inside a single process. 
Instead of a static object, a distributed lock ensures only one active application instance exists at any time.

**Multiple Pods may be running, but only one becomes the active leader. Others stay passive and take over only if the leader fails.**

### Pod Disruption Budget

A Pod Disruption (PDB) controls how many Pods of an application are allowed to be voluntarily disrupted at the same time. Unlike singleton
or leader-election mechanisms that limit the maximum number of running instances, a PDB ensures that a minimum number of Pods
always remain available during maintenance events like node draining or cluster scaling.

A PDB can specify either minAvailable (minimum number or percentage of Pods that must stay up) or maxUnavailable (maximum number of Pods 
allowed to be down), but not both. This mechanism is essential for quorum-based systems or any critical service that must maintain a minimum
level of availability during planned disruption.
