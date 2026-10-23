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

- [x] Inspect live usage with `docker stats`.
- [x] Check Docker's cgroup driver and version.
- [x] Run a test container with CPU and memory limits.
- [x] Inspect configured memory, memory-plus-swap, and CPU limit fields.
- [x] Generate CPU load and observe the effect of the 0.5 CPU limit.
- [x] Stop the CPU workload and verify usage returns to idle.
- [x] Compare temporary disk-file usage with reported container memory usage.
- [x] Complete a controlled memory-allocation experiment.
- [x] Safely test memory pressure / OOM behavior.
- [x] Record exit status and whether Docker reports `OOMKilled`.

See [`docker-resource-limits-and-monitoring.md`](docker-resource-limits-and-monitoring.md) for the detailed commands, observed outputs, and conclusions.

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

**Baseline and cgroups:**

- `docker ps` showed only `ops-healthcheck` running before the resource test was created.
- `docker stats --no-stream` reported `ops-healthcheck` using `9.449 MiB / 3.746 GiB`, with `0.00%` CPU at the sampled instant.
- `docker info` reported cgroup driver `systemd` and cgroup version `2`.

**Memory limit:**

A dedicated Alpine test container named `ops-resource-test` was created with a 128 MiB memory limit and `--memory-swap=128m`:

```bash
docker run -d \
  --name ops-resource-test \
  --memory=128m \
  --memory-swap=128m \
  alpine:3.22 \
  sh -c 'while true; do sleep 5; done'
```

- `docker inspect` reported `Memory=134217728` and `MemorySwap=134217728`; these values are bytes and correspond to 128 MiB each.
- At the baseline sample, `docker stats` reported approximately `2.324 MiB / 128 MiB`.

**CPU limit:**

The container was updated to use half a CPU:

```bash
docker update --cpus="0.5" ops-resource-test
```

- `docker inspect` reported `NanoCpus=500000000`, representing 0.5 CPU. `CpuQuota` and `CpuPeriod` were `0` in this inspection output; the configured limit was represented through `NanoCpus`.
- A busy loop was started inside the container. `docker stats` reported `50.01%` CPU, consistent with the 0.5 CPU limit in this environment.
- `docker top` showed host-side PIDs, while `ps` executed inside the container showed container-side PIDs. Using the container-side PID (`62`) stopped the busy loop, after which CPU usage returned to `0.00%`.

**Disk writes versus memory usage:**

A background process wrote 2 MiB zero-filled files under `/tmp` in repeated iterations. At one observation point:

- `du -sh /tmp` reported `68.0M`.
- `docker stats` reported approximately `4.656 MiB / 128 MiB` of memory, `36.7 MB` block I/O, and four PIDs.

This demonstrates that the amount of data written to disk is not equivalent to the amount of resident memory reported by `docker stats`. The file-writing experiment was not a RAM-pressure test and was removed before the controlled memory test.

**Controlled memory-pressure / OOM experiment:**

A separate container named `ops-memory-stress` was run with a 32 MiB memory limit and `--memory-swap=32m`. The `python:3.13-alpine` image ran a Python script that allocated and retained 1 MiB `bytearray` chunks, pausing between allocations.

- `docker stats ops-memory-stress` reached `31.94MiB / 32MiB` (**99.80%** memory usage).
- The script printed successful allocation messages through approximately 28 MiB, then stopped before reaching its intended 40 MiB allocation target.
- Inspection confirmed the outcome:

```bash
docker inspect ops-memory-stress --format 'Status={{.State.Status}} OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
```

Observed output:

```text
Status=exited OOMKilled=true ExitCode=137
```

`OOMKilled=true` confirms that Docker recorded an out-of-memory kill. Exit code `137` is consistent with termination by `SIGKILL` (128 + signal 9). Python's runtime memory overhead also consumes the container's memory budget, so the process was killed before it could retain 32 MiB of payload allocations. The test container was removed after verification.

**Key resource-limit conclusions:**

- `docker stats` reports a point-in-time view of container CPU, memory, network I/O, block I/O, and process count.
- `--memory=32m` sets the memory limit for the stress container. Setting `--memory-swap=32m` to the same value means the configured memory-plus-swap limit is also 32 MiB, leaving no additional swap allowance beyond that limit.
- Disk writes and block I/O are not equivalent to resident memory usage; use a process that actually allocates and retains memory to test memory pressure.
- `docker inspect` fields such as `.State.OOMKilled`, `.State.ExitCode`, and `.State.Status` help confirm how a test container ended. Do not infer an OOM kill from memory usage alone.

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

- [`docker-healthchecks-and-restart-policies.md`](docker-healthchecks-and-restart-policies.md)
- [`docker-resource-limits-and-monitoring.md`](docker-resource-limits-and-monitoring.md)
- [`docker-images-and-containers.md`](docker-images-and-containers.md)
- [`docker-networking.md`](docker-networking.md)
- [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md)
- [`docker-compose.md`](docker-compose.md)
- [`docker-buildkit.md`](docker-buildkit.md)
- [`docker-storage-architecture.md`](docker-storage-architecture.md)
- [`docker-linux-namespaces-and-cgroups.md`](docker-linux-namespaces-and-cgroups.md)
- [`docker-registry.md`](docker-registry.md)
- [`docker-container-orchestration.md`](docker-container-orchestration.md)

