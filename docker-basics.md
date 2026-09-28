# Docker Basics

Basic Docker commands and concepts practiced on Ubuntu running inside WSL2.

## Why Docker?

Docker packages applications together with their dependencies into self-contained containers. This helps provide a consistent environment across development, testing, and production and reduces dependency and version conflicts.

Docker standardizes application deployment, making it faster, more reliable, and easier to scale across different environments.

## What is Docker?

Docker is an open-source containerization platform that packages applications and their dependencies into isolated units called containers.

Unlike traditional virtualization, containers share the host operating system kernel rather than running a full guest operating system for each application. This makes containers lightweight, fast, and portable.

Docker helps address the "works on my machine" problem by providing a consistent execution environment across development, testing, and production.

## Core Components

- **Docker Engine** — the underlying container platform used to build and run containers.
- **Docker Image** — a packaged, immutable template used to create containers.
- **Docker Container** — a running instance of an image.
- **Dockerfile** — a file containing instructions for building an image.
- **Docker Hub / Registry** — repositories used to store and distribute container images.
- **Docker CLI** — the command-line interface used to interact with Docker.

## Containers

Containers are isolated environments with their own processes, network interfaces, and mounts. Unlike virtual machines, containers share the host OS kernel.

The Open Container Initiative (OCI) defines open standards for container image formats and container runtimes.

A typical modern Docker container runtime stack includes:

```text
Docker CLI
    |
Docker Engine
    |
containerd
    |
runc
```

Docker provides a higher-level interface that makes working with containers easier without requiring users to manage low-level container runtime components directly.

## How Docker Works

An operating system can be viewed as having two broad layers:

- **OS kernel** — interacts with the underlying hardware and provides core operating-system functionality.
- **User-space software** — tools and programs such as shells, libraries, compilers, file managers, and other utilities that make up the user environment.

Docker containers share the underlying kernel of the Docker host. The container provides the user-space software it needs while relying on the host kernel for core operating-system functionality.

For example, on a Linux host, containers can use different Linux user-space environments while sharing the host's Linux kernel.

> **Note:** Containers rely on the host kernel. A Linux Docker host cannot natively run a Windows container that requires the Windows kernel. A separate Windows environment, such as a virtual machine, can be used when a different kernel is required.

### Containers vs Virtual Machines

Containers and virtual machines both provide isolation, but they achieve it differently.

```text
Virtual Machines                         Containers

+----------------------+                 +----------------------+
|      Application     |                 |      Application     |
+----------------------+                 +----------------------+
|     Guest OS         |                 |   Container runtime  |
+----------------------+                 +----------------------+
|   Virtual Hardware   |                 |      Host Kernel     |
+----------------------+                 +----------------------+
|     Host OS          |                 |      Host OS          |
+----------------------+                 +----------------------+
|      Hardware        |                 |      Hardware        |
+----------------------+                 +----------------------+

Each VM includes a guest OS.              Containers share the
                                           host OS kernel.
```

| Aspect | Virtual Machine | Container |
|---|---|---|
| Operating system | Includes a guest OS | Shares the host kernel |
| Isolation | Virtualized hardware and guest OS | Process and namespace isolation |
| Overhead | Generally higher | Generally lower |
| Startup | Generally slower | Generally faster |
| Portability | Depends on VM image/hypervisor | Portable when the required host kernel is available |

The key distinction is that a virtual machine virtualizes the hardware environment and runs a complete guest operating system, while a container isolates processes while sharing the host kernel.

## Commands

- `docker --version`
- `docker run hello-world`
- `docker ps`
- `docker ps -a`
- `docker images`
- `docker compose version`

## Key Concepts Learned

- Docker Engine and Docker CLI installation
- Pulling images from Docker Hub
- Difference between images and containers
- Running vs exited containers
- PID 1 and container lifecycle
- Containers share the host Linux kernel
- Containers are isolated processes, not virtual machines
- Containers share the host kernel while providing their own user-space software
- Docker containers depend on the host kernel type
- Containers and virtual machines provide isolation using different approaches
