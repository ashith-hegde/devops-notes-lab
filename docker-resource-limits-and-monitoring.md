# Docker Resource Limits and Monitoring

## Objective

Practise inspecting container resource usage, applying CPU and memory limits, generating controlled CPU and memory pressure, and verifying the outcome using Docker's runtime state.

## Core Concepts

- **`docker stats`** shows a live or one-time sample of CPU, memory, network I/O, block I/O, and process count.
- **Memory limit (`--memory`)** caps the memory available to a container.
- **Memory plus swap (`--memory-swap`)** sets the combined memory-and-swap limit. When it is set to the same value as `--memory`, it provides no additional swap allowance beyond that configured limit.
- **CPU limit (`--cpus`)** limits the CPU time available to a container. A value of `0.5` represents half of one CPU's capacity.
- **OOM status** should be verified from the container's recorded state. A high memory percentage alone does not prove an out-of-memory kill.
- **Block I/O is not RAM usage.** Writing large files may increase block I/O without causing a comparable increase in resident memory.

## 1. Inspect Current Usage

Take a one-time sample of all running containers:

```bash
docker stats --no-stream
```

Inspect a specific container continuously:

```bash
docker stats ops-resource-test
```

Press `Ctrl+C` to stop the display. This does not stop the container or its workload.

## 2. Configure and Inspect Memory and CPU Limits

The main resource test container was created with a 128 MiB memory limit and the same memory-plus-swap limit:

```bash
docker run -d \
  --name ops-resource-test \
  --memory=128m \
  --memory-swap=128m \
  alpine:3.22 \
  sh -c 'while true; do sleep 5; done'
```

Set its CPU limit to half of one CPU:

```bash
docker update --cpus="0.5" ops-resource-test
```

Inspect the configured limits:

```bash
docker inspect ops-resource-test --format 'Memory={{.HostConfig.Memory}} MemorySwap={{.HostConfig.MemorySwap}} NanoCpus={{.HostConfig.NanoCpus}} CpuQuota={{.HostConfig.CpuQuota}} CpuPeriod={{.HostConfig.CpuPeriod}}'
```

Observed values:

```text
Memory=134217728 MemorySwap=134217728 NanoCpus=500000000 CpuQuota=0 CpuPeriod=0
```

The memory values are bytes (128 MiB each). `NanoCpus=500000000` represents 0.5 CPU; in this inspection, the CPU limit was represented through `NanoCpus`, while `CpuQuota` and `CpuPeriod` were `0`.

## 3. Generate and Stop CPU Load

A temporary busy loop was started inside `ops-resource-test`:

```bash
docker exec -d ops-resource-test sh -c 'while true; do :; done'
```

Monitor it:

```bash
docker stats --no-stream ops-resource-test
```

Observed CPU usage was approximately `50.01%`, consistent with the configured 0.5 CPU limit in this environment.

To find processes, compare host-side process IDs from `docker top ops-resource-test` with container-side process IDs from:

```bash
docker exec -it ops-resource-test sh
ps
```

The temporary loop was stopped using its **container-side** PID with `kill <container-side-pid>`. After the loop stopped, CPU usage returned to `0.00%`. PID values can differ between host and container namespaces; identify the process before killing it.

## 4. Disk Writes Are Not a Memory-Pressure Test

An earlier experiment wrote repeated zero-filled files under `/tmp` using `dd`. It increased the container's disk usage and block I/O but did not deliberately allocate and retain RAM. At one observation point, `du -sh /tmp` reported `68.0M`, while `docker stats` showed approximately `4.656 MiB / 128 MiB` memory and `36.7 MB` block I/O.

**Conclusion:** file size and block I/O are not substitutes for measuring resident memory. Use a controlled memory-allocation workload to test the memory limit.

## 5. Controlled Memory-Pressure Test

A separate container was run with a 32 MiB memory limit and no additional swap allowance:

```bash
docker run --name ops-memory-stress \
  --memory=32m \
  --memory-swap=32m \
  python:3.13-alpine \
  python -u -c '
import time
blocks = []
for i in range(40):
    blocks.append(bytearray(1024 * 1024))
    print(f"Allocated approximately {(i + 1)} MiB", flush=True)
    time.sleep(0.5)
time.sleep(60)
'
```

The Python script retains each allocated 1 MiB block so memory use grows over time. The Python image was pulled because it was not present locally.

In a second terminal, monitor usage:

```bash
docker stats ops-memory-stress
```

Observed peak sample:

```text
CONTAINER ID   NAME                CPU %   MEM USAGE / LIMIT   MEM %
94ec6a1a40fd   ops-memory-stress   0.78%   31.94MiB / 32MiB    99.80%
```

The script printed allocation messages through approximately 28 MiB and then stopped before reaching 40 MiB. Python's own runtime overhead also counts against the container's memory limit.

Verify the container's outcome:

```bash
docker inspect ops-memory-stress --format 'Status={{.State.Status}} OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
```

Observed output:

```text
Status=exited OOMKilled=true ExitCode=137
```

Interpretation:

- `Status=exited`: the container is no longer running.
- `OOMKilled=true`: Docker recorded that the container was killed by an out-of-memory event.
- `ExitCode=137`: consistent with termination by `SIGKILL` (128 + signal 9).

This is the verified OOM result for the lab. Do not assume every exit code `137` is an OOM kill; inspect `.State.OOMKilled` as well.

## 6. Cleanup

After collecting the OOM result, remove only the temporary stress container:

```bash
docker rm ops-memory-stress
```

If a test container is still running, stop/remove it only after you no longer need its output. Do not remove `ops-resource-test` or `ops-healthcheck` as part of this step; they are retained for the wider Docker operations lab.

## Key Takeaways

- Use `docker stats` to observe usage and compare it with configured limits.
- Use `docker inspect` to verify limits and diagnose container exit state.
- A `0.5` CPU limit can appear as roughly `50%` in `docker stats`.
- A memory limit is a ceiling for the container's total memory use, including application/runtime overhead—not just the payload explicitly allocated by the test script.
- Confirm an OOM event with `.State.OOMKilled`; do not infer it from high usage alone.
- Disk writes and block I/O do not prove RAM pressure.

## Related Notes

- [`docker-operations-lab.md`](docker-operations-lab.md)
- [`docker-linux-namespaces-and-cgroups.md`](docker-linux-namespaces-and-cgroups.md)
- [`docker-healthchecks-and-restart-policies.md`](docker-healthchecks-and-restart-policies.md)

