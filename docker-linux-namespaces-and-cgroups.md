# Docker: Linux Namespaces and Cgroups

These notes explain two Linux kernel mechanisms that help containers remain isolated while sharing the host kernel:

- **Namespaces** isolate what a process can see.
- **Control groups (cgroups)** account for and limit how much of certain system resources a group of processes can use.

This is a deeper companion to [`docker-basics.md`](docker-basics.md) and [`docker-engine.md`](docker-engine.md). PID 1 and container lifecycle are introduced in [`docker-images-and-containers.md`](docker-images-and-containers.md); this note explains the underlying PID namespace and adds the other major namespace types and resource controls.

## Why namespaces and cgroups?

A container is not a virtual machine with its own kernel. Its processes run on the host kernel, but Linux can give them isolated views of system resources.

```text
Docker CLI / Docker Engine
          |
          v
   container runtime
          |
          v
      Linux kernel
       /       \
      v         v
 Namespaces   cgroups
 What a       How much of
 process can  certain resources
 see           it can use
```

Namespaces and cgroups solve different problems; neither replaces the other.

## Linux namespaces

A **namespace** gives a set of processes an isolated view of a particular system resource. Different containers can have separate views of processes, networking, mounts, and other resources while sharing the same host kernel.

### Major namespace types

| Namespace | What it isolates |
|---|---|
| **PID** | Process ID numbers and the process tree visible to processes |
| **Network** | Network devices, addresses, routes, sockets, and network-stack state |
| **Mount** | The set of filesystem mount points visible to processes |
| **IPC** | System V IPC objects and POSIX message queues |
| **UTS** | Hostname and NIS domain name |
| **User** | User/group ID mappings and associated privilege scope |
| **Cgroup** | The cgroup hierarchy view visible to processes |
| **Time** | Certain system clocks, including offsets for monotonic and boot-time clocks |

The isolation a container receives depends on its runtime configuration. For example, `--network host` or `--pid host` can make a container share the host's network or PID namespace rather than use a separate one.

**Mental model:** namespaces answer, “What does this process see?”

Namespaces alone do not guarantee that a container cannot affect the host. Security also depends on capabilities, Linux security modules, seccomp, filesystem permissions, runtime configuration, and keeping the host kernel secure.

## PID namespaces

The initial PID namespace contains the host's process tree. A process inside a new PID namespace can see a separate process tree with its own PID numbering.

### How PIDs work

A process can have different PID values as seen from nested PID namespaces. For example:

```text
Host PID namespace                 Container PID namespace
-------------------                -----------------------
PID 1: host init                   PID 1: container main process
PID 2: host process                PID 2: child process
PID 4100: container main process
PID 4101: container child process
```

The numbers are illustrative, not fixed values. The container's main process might be PID `4100` from the host's perspective but PID `1` inside its PID namespace.

So PIDs are unique **within a given PID namespace**, but a process can have different PID identities in different nested namespaces.

### Why PID 1 matters

The first process created in a new PID namespace is assigned PID 1 in that namespace. It is the root of that namespace's process tree and has special responsibilities, including reaping orphaned child processes. Signal handling for PID 1 also has special rules.

For a normal Docker container, the configured main process is typically PID 1 inside the container. If that process exits, the container stops, even if another process had been started inside it with `docker exec`.

### Inspect PID 1 inside a container

Run a temporary container with a long-running main process:

```bash
docker run -d --name pid-demo ubuntu:24.04 sleep infinity
```

Inspect the process tree from inside:

```bash
docker exec pid-demo ps -ef
docker exec pid-demo ps -p 1 -o pid,ppid,comm
```

Compare the host's view:

```bash
docker top pid-demo
```

`docker top` shows the container's processes from the host side; displayed PIDs may differ from the PIDs visible inside the container. Exact output can vary by image and installed tools.

Clean up:

```bash
docker rm -f pid-demo
```

## Other namespace examples

### Network namespace

A separate network namespace can provide a container with its own network interfaces, routes, sockets, and network-stack state. Docker connects these namespaces to Docker networks and can publish container ports onto the host.

See [`docker-networking.md`](docker-networking.md) for bridge networks, DNS, and port publishing.

### Mount namespace

A mount namespace controls which mount points a process sees. Docker uses mount and filesystem features to present the container's filesystem and configured mounts. Bind mounts and volumes relate to filesystem visibility and persistence, but are not the same thing as a mount namespace.

See [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md) for persistent storage.

### User namespace

A user namespace can map a user ID inside a container to a different user ID outside it. This can help reduce the host privileges associated with a process that appears to be root inside the container. User namespace remapping and rootless Docker are distinct configurations; setup depends on the host and Docker configuration.

Do not assume every Docker installation automatically uses a separate user namespace.

## Cgroups: resource accounting and limits

**Control groups (cgroups)** are a Linux kernel mechanism for organizing processes and accounting for or controlling resource usage. Docker can configure cgroup limits for containers.

| Mechanism | Main question | Examples |
|---|---|---|
| Namespaces | What can the process see? | PID tree, network interfaces, mount points |
| Cgroups | How much of a resource can the process group use, or how is usage accounted for? | CPU time, memory, I/O |

Without explicit resource limits, a container may consume a large share of the host's available CPU or memory, subject to host capacity and other controls. Limits can help protect other workloads from a noisy neighbor.

Cgroups do not necessarily reserve a fixed amount of hardware for a container. Behavior depends on the resource, configured limit, cgroup version, and host.

### Limit CPU usage

```bash
docker run --rm --cpus=0.5 ubuntu:24.04 sleep infinity
```

- `--cpus=0.5` sets a CPU quota equivalent to up to half of one CPU core's processing time over the quota period.
- It does **not** mean the container is permanently assigned half of a physical core.
- `sleep infinity` keeps the main process alive for the example.

### Limit memory

```bash
docker run --rm --memory=100m ubuntu:24.04 sleep infinity
```

- `--memory=100m` sets a memory limit of 100 MiB for the container.
- If the container exceeds its allowed memory and cannot recover, kernel out-of-memory handling may kill a process.
- A very low limit may be insufficient for some applications; choose limits based on observed workload needs.

These examples demonstrate the options; they are not recommendations for production limits.

### Inspect resource use

In one terminal, start a container with limits:

```bash
docker run -d --name resource-demo \
  --cpus=0.5 \
  --memory=100m \
  ubuntu:24.04 sleep infinity
```

View current usage:

```bash
docker stats resource-demo
```

Inspect its configuration:

```bash
docker inspect resource-demo
```

`docker inspect` returns detailed JSON, including host configuration. Exact fields and representation can vary by Docker version and cgroup configuration.

Clean up:

```bash
docker rm -f resource-demo
```

## What happens when a container uses too much CPU or memory?

- **CPU limit:** the container can be throttled when it reaches its configured CPU quota. It may run more slowly rather than being terminated simply because it hits the quota.
- **Memory limit:** if memory usage exceeds the limit and the kernel cannot reclaim enough memory, an out-of-memory event can cause a process in the container to be killed.
- **Noisy neighbor:** one workload consuming excessive resources can degrade other workloads on the same host. Limits and monitoring help reduce this risk, but do not replace capacity planning.

## Key takeaways

- Containers share the host Linux kernel; namespaces provide isolated views of kernel-managed resources.
- Linux has multiple namespace types, including PID, network, mount, IPC, UTS, user, cgroup, and time.
- PID values are relative to a PID namespace. A container's main process is typically PID 1 inside the container, even though it has a different host PID.
- Cgroups provide resource accounting and controls such as CPU and memory limits.
- Namespaces and cgroups complement each other; neither alone is a complete security boundary.
- Docker options and defaults can change isolation behavior, so check the actual container configuration.

## Further reading

- [Linux namespaces — `namespaces(7)`](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Linux cgroups — `cgroups(7)`](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [Docker resource constraints](https://docs.docker.com/engine/containers/resource_constraints/)
- [Docker Engine security](https://docs.docker.com/engine/security/)

