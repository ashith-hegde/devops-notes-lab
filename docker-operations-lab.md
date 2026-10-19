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

These values describe the environment at the time of the baseline check and may change. The environment already contained Docker resources before the lab began. Inspect resources before removing anything, and clean up only resources created specifically for this lab.

## Lab Checkpoints

### 1. Baseline and Lab Setup

- [x] Review Docker Engine, Compose, storage, cgroup, and resource information.
- [x] Confirm the lab uses newly named resources to avoid conflicts with existing containers and volumes.
- [x] Create a dedicated working directory for lab files.
- [x] Record the commands and relevant baseline observations.

### 2. Health Checks and Restart Policies

- [x] Create a small test service with a health check.
- [x] Observe healthy and unhealthy states.
- [x] Compare container process state with application health.
- [x] Test restart policies and record what triggers a restart.
- [x] Verify that an unhealthy status alone does not necessarily restart a container.

See [`docker-healthchecks-and-restart-policies.md`](docker-healthchecks-and-restart-policies.md) for the concepts, commands, observations, and conclusions from this checkpoint.

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

> Record real command output and observations here as each checkpoint is completed. Do not pre-fill expected output as if it were observed.

### Checkpoint 1 — Baseline and Setup

**Observations:**

- The lab was run in Ubuntu on WSL2.
- Docker Engine 29.7.1 and Docker Compose v5.4.0 were reported at baseline.
- The daemon reported the `overlayfs` storage driver, cgroup v2, 8 CPUs, approximately 3.746 GiB of memory, and inactive Swarm mode.
- A dedicated working directory was created for the lab.

### Checkpoint 2 — Health Checks and Restart Policies

**Health-check experiment:**

A detached Nginx container named `ops-healthcheck` was created with a `wget` health check against `http://127.0.0.1/`, a 5-second interval, a 3-second timeout, and two retries.

```bash
docker run -d --name ops-healthcheck \
  --health-cmd='wget -q -O /dev/null http://127.0.0.1/ || exit 1' \
  --health-interval=5s \
  --health-timeout=3s \
  --health-retries=2 \
  nginx:alpine
```

- The container initially reported `running healthy`.
- Moving `/usr/share/nginx/html/index.html` to a backup filename caused the health check to fail. The container continued running but became `unhealthy`; the check received HTTP `403 Forbidden` because the default index file was missing and directory listing was not available.
- Restoring the index file returned the container to `running healthy`. The latest health-check records had `ExitCode: 0`.
- The experiment demonstrated that container process state and application health are separate signals.

**Health check versus restart policy:**

The restart policy was set to `unless-stopped`:

```bash
docker update --restart unless-stopped ops-healthcheck
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' ops-healthcheck
```

The policy inspection returned `unless-stopped`. After the index file was moved again, the observed state was:

```text
running unhealthy 0
```

The container did not restart merely because its health check failed. The index file was restored afterward.

**Process-exit experiments:**

- A direct `docker kill --signal=SIGKILL` against a container configured with `unless-stopped` left it exited with code `137` and restart count `0`. A direct, explicit stop/kill operation is not a reliable way to demonstrate an unexpected process failure because Docker treats it as a manual stop for restart-policy purposes.
- A separate Alpine container was created with `--restart on-failure` and a command that slept for 15 seconds before exiting with status `1`. Its observed restart count increased from `1` to `2` across successive inspections, and it was running at both inspection times. This confirmed that `on-failure` restarted it after non-zero exits.
- For another `unless-stopped` test, killing the container's main process from inside its own PID namespace did not terminate it. The container's PID 1 has special signal-handling behavior. After the process was terminated from the host PID namespace with elevated privileges, Docker restarted the container: the host PID changed from `13828` to `19479`, and `RestartCount` became `1`.
- The temporary test containers `restart-test`, `restart-test2`, and `restart-test3` were removed after the experiments. `ops-healthcheck` was retained for later lab work.

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

- A container can be `running` while its health status is `unhealthy`.
- A Docker health check reports health; by itself, an unhealthy status does not cause Docker to restart the container.
- Restart policies respond to the main process exiting, subject to the configured policy and Docker's manual-stop behavior.
- `RestartCount`, `.State.ExitCode`, `.State.Health.Status`, and Docker lifecycle events help distinguish process exits from health-check failures.
- Prefer Docker's container-management commands for routine operations instead of directly signalling host-side container processes.

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

