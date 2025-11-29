# HEALTH PROBE

## Process Health Checks

A _process health check_ is the simplest health check the Kubelet constantly performs on the container processes. If 
the container processes are not running, the probing is restarted. So even without any other health checks, the application 
becomes slightly more robust with this generic check. If your application is capable of detecting any kind of failure and 
shutting itself down, the process health check is all you need. However, for most cases that is not enough and other types 
of health checks are also necessary.

## Liveness Probes

- Check if the application is really functioning properly, including deadlocks or application freezes.
- Performed from outside the container -> ensures failure is detected even if the app cannot report itself.
- On failure -> the container is **restarted**.
- Check methods: HTTP GET, TCP Socket, Exec command.

## Readiness Probes

- **Liveness checks**: keep applications healthy by **killing unhealthy containers** and **restarting them**.
- **Problem**: restarting may not always help:
  - Container is **still starting up** -> not ready for requests.
  - Container is **overloaded**, latency increasing -> want to **temporarily stop traffic**.
- **Solution**: **Readiness probes**
  - Determine if the container is **ready to serve traffic**.
  - **Failed readiness probe** → container removed from service endpoint → **no new traffic sent to it**.
  - Methods are the same as liveness probe: HTTP GET, TCP Socket, Exec.
  - Performed **regularly**, allowing container to **warm up** or recover before receiving requests.