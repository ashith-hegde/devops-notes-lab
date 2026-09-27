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
