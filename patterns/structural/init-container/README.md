# INIT CONTAINER

Initialization is a common requirement in application design to prepare the environment before the main process starts.
In Kubernetes, this responsibility is handled by **init containers**, which run before the main application containers
within a Pod.

Init containers are used to perform prerequisite tasks such as setting up file permissions, initializing data or database
schemas, installing required resources, or waiting for external dependencies to become available. This approach separates
initialization logic from the main application, improves security, and keeps the application image lightweight and focused
on its core responsibility.

Init containers are defined within a Pod and run sequentially before application containers, which run in parallel. 
They function like constructors in object-oriented programming, preparing the environment for the main application. 
Init containers are usually small, short-lived, and idempotent; if an init container fails, the entire Pod is restarted.

Init containers share the same resources, volumes, and security settings as application containers but do not have 
readiness checks. They also influence Pod-level resource calculations: the Pod’s request and limit are based on the 
highest init container resource or the sum of application container resources, whichever is greater. This can affect 
scheduling efficiency if init containers require significantly more resources.

Finally, init containers support separation of concerns: application engineers focus on the application logic, 
while deployment engineers handle initialization and configuration. For example, an HTTP server container serves files, 
while an init container clones a Git repository; both share a volume to exchange data.

## Practice

Let's create this bare Pod into the current namespace with the following:

```bash
kubectl create -f 
```

## More Information:
1. [Init Containers](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/#creating-a-pod-that-has-an-init-container)