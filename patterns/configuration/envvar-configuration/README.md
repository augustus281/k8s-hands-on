# EnvVar Configuration

## Problem

Applications should not hardcode configuration, as they need to run in multiple environments and change settings without
rebuilding the application. This is especially important in containerized systems where images are immutable and reused.
Therefore, configuration must be externalized and injected at runtime after than embedded in the application code.

## Solution

The recommended solution is to externalized application configuration using environment variables, following the Twelve-Factor
App principles. In containerized environments, default values can be defined at build time and overridden at runtime via Docker
commands or Kubernetes Pod specifications, with support for ConfigMaps (non-sensitive data) and Secrets (sensitive data).
This managed independently from application code.