# Docker Operations Lab

## Objective

Practise operating and troubleshooting Docker containers using a single local lab. Record actual commands, outputs, observations, and conclusions as the lab progresses.

## Environment Baseline

- **Environment:** Ubuntu on WSL2
- **Docker Engine:** 29.7.1
- **Docker Compose:** v5.4.0
- **Storage driver:** `overlayfs`
- **Cgroup version:** 2
- **Available CPUs:** 8
- **Reported memory:** 3.746 GiB
- **Swarm:** inactive

These values describe the environment at the time of the baseline check and may change. The Docker daemon reports four stopped containers and six images before this lab. Existing containers include `mailpit`, `server1`, `server2`, and `server3`; do not remove or modify them as part of cleanup unless explicitly intended.

## Lab Checkpoints

### 1. Baseline and Lab Setup

- [ ] Review Docker Engine, Compose, storage, cgroup, and resource information.
- [ ] Confirm the lab uses newly named resources to avoid conflicts with existing containers and volumes.
- [ ] Create a dedicated working directory for lab files.
- [ ] Record the commands and relevant baseline observations.

### 2. Health Checks and Restart Policies

- [ ] Create a small test service with a health check.
- [ ] Observe healthy and unhealthy states.
- [ ] Compare container process state with application health.
- [ ] Test restart policies and record what triggers a restart.
- [ ] Verify that an unhealthy status alone does not necessarily restart a container.

### 3. Resource Limits and Monitoring

- [ ] Inspect live usage with `docker stats`.
- [ ] Run a test container with CPU and memory limits.
- [ ] Observe resource constraints and relevant container inspection fields.
- [ ] Safely test memory pressure / OOM behavior.
- [ ] Record exit status and whether Docker reports `OOMKilled`.

### 4. Troubleshooting Workflow

- [ ] Investigate a stopped container using `docker ps -a`, `docker logs`, and `docker inspect`.
- [ ] Inspect exit codes and relevant container state.
- [ ] Use `docker events` to observe lifecycle events.
- [ ] Diagnose a deliberately introduced networking or port-publishing failure.
- [ ] Inspect Docker disk usage with `docker system df`.
- [ ] Distinguish safe inspection commands from destructive prune/remove commands.

### 5. Container Security

- [ ] Run a test workload as a non-root user where supported.
- [ ] Inspect its configured user and Linux capabilities.
- [ ] Test a read-only root filesystem and identify required writable paths.
- [ ] Review safe handling of environment variables and secrets.
- [ ] Explain why mounting `/var/run/docker.sock` gives the container highly privileged access.
- [ ] Record security limitations and platform-specific observations.

### 6. Production Compose Patterns

- [ ] Define a small multi-service Compose application.
- [ ] Add health checks and health-based dependency conditions where appropriate.
- [ ] Test startup behavior and distinguish service start from service readiness.
- [ ] Use environment variables for non-secret configuration.
- [ ] Review safe approaches for secrets and persistent data.
- [ ] Confirm cleanup only removes resources created for this lab.

### 7. Multi-Stage Builds and Image Optimization

- [ ] Build a small example image using multiple build stages.
- [ ] Copy only required runtime artifacts into the final stage.
- [ ] Compare image size and inspect image history.
- [ ] Verify build context exclusions with `.dockerignore`.
- [ ] Record what was reduced and what trade-offs remain.

### 8. Image Vulnerability Scanning

- [ ] Choose an available image-scanning tool (for example, Trivy or Docker Scout).
- [ ] Scan a lab image and record the tool/version and scan date.
- [ ] Review findings, severity, and available fixes.
- [ ] Explain the limits of a vulnerability scan; a clean scan is not proof that an image is completely secure.

### 9. Lab Review and Cleanup

- [ ] Summarize the most useful troubleshooting commands.
- [ ] Record environment-specific differences caused by WSL2 or the Docker runtime.
- [ ] Remove only containers, images, networks, and volumes created specifically for this lab and no longer needed.
- [ ] Verify the final state with `docker ps -a` and `docker system df`.
- [ ] Identify any concepts that need follow-up practice.

## Results and Observations

> Add real command output and observations here as each checkpoint is completed. Do not pre-fill expected output as if it were observed.

### Checkpoint 1 — Baseline and Setup

**Commands run:**

_TODO_

**Observations:**

_TODO_

### Checkpoint 2 — Health Checks and Restart Policies

_TODO_

### Checkpoint 3 — Resource Limits and Monitoring

_TODO_

### Checkpoint 4 — Troubleshooting Workflow

_TODO_

### Checkpoint 5 — Container Security

_TODO_

### Checkpoint 6 — Production Compose Patterns

_TODO_

### Checkpoint 7 — Multi-Stage Builds and Image Optimization

_TODO_

### Checkpoint 8 — Image Vulnerability Scanning

_TODO_

### Checkpoint 9 — Review and Cleanup

_TODO_

## Key Takeaways

_To be completed after the lab using observed results._

## Related Notes

- [`docker-images-and-containers.md`](docker-images-and-containers.md)
- [`docker-networking.md`](docker-networking.md)
- [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md)
- [`docker-compose.md`](docker-compose.md)
- [`docker-buildkit.md`](docker-buildkit.md)
- [`docker-storage-architecture.md`](docker-storage-architecture.md)
- [`docker-linux-namespaces-and-cgroups.md`](docker-linux-namespaces-and-cgroups.md)
- [`docker-registry.md`](docker-registry.md)
- [`docker-container-orchestration.md`](docker-container-orchestration.md)

