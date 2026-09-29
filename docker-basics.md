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

## Docker Command Reference

Use `docker --help` or `docker <command> --help` (for example, `docker run --help`) to see supported options. Exact options can vary by Docker version.

### Check Docker and discover resources

| Command | What it does / useful options |
|---|---|
| `docker --version` | Prints the Docker CLI version. It does not by itself prove the Docker Engine is running. |
| `docker info` | Shows details about the Docker Engine, host, storage driver, and configured resources. |
| `docker compose version` | Shows the installed Docker Compose version. |
| `docker ps` | Lists running containers. `-a` includes stopped containers; `-q` prints only container IDs. |
| `docker images` | Lists local images. `-a` includes intermediate images; `-q` prints image IDs. `docker image ls` is the newer equivalent form. |
| `docker network ls` | Lists Docker networks. |
| `docker volume ls` | Lists Docker-managed volumes. |

### Run and manage containers

| Command | What it does / useful options |
|---|---|
| `docker run hello-world` | Creates and runs a container from the image; Docker pulls the image if needed. `hello-world` exits after printing its message. |
| `docker run nginx` | Creates and runs a container in the foreground by default. If the main process exits, the container stops. |
| `docker run -d --name web1 nginx` | `-d` runs detached (background); `--name` assigns a readable container name. |
| `docker run --rm image command` | Removes the container automatically after it exits. Useful for temporary tasks; it does not remove the image. |
| `docker run -it ubuntu bash` | `-i` keeps standard input open and `-t` allocates a terminal, allowing an interactive shell. |
| `docker run -p 8080:80 nginx` | Publishes host port 8080 to container port 80. Format: `HOST_PORT:CONTAINER_PORT`. |
| `docker run -v ~/web-content:/usr/share/nginx/html nginx` | Mounts a host path into the container. `HOST_PATH:CONTAINER_PATH` is a bind mount; the target path depends on the application. |
| `docker stop <container>` | Gracefully asks a running container to stop. Use a name or ID. |
| `docker start <container>` | Starts an existing stopped container. It does not create a new container. |
| `docker rm <container>` | Removes a stopped container. Stop it first if it is still running. `-f` force-removes a running or stopped container. |
| `docker exec <container> <command>` | Runs an additional command in an already-running container. Add `-it` for an interactive terminal, e.g. `docker exec -it web1 sh`. |
| `docker attach <container>` | Connects the terminal to the main process's standard input/output streams; it does not start a new process. Be careful with interactive signals. |
| `docker logs <container>` | Displays the container's captured output. `-f` follows new output; `--tail 100` shows the last 100 lines. |
| `docker inspect <container-or-image>` | Shows detailed JSON metadata. `--format '{{.State.Status}}'` can extract a specific field. |
| `docker port <container>` | Shows published port mappings for a container. |

### Download, build, tag, and publish images

| Command | What it does / useful options |
|---|---|
| `docker pull nginx:tag` | Downloads an image from a registry. Specify `:tag` to choose a version; without one, Docker defaults to `latest`. |
| `docker build -t my-app:1.0 .` | Builds an image from a Dockerfile in the build context. `-t` sets the image name and tag; `.` means the current directory is the build context. |
| `docker tag my-app:1.0 username/my-app:1.0` | Adds another name/tag to the same local image; it does not rebuild the image. |
| `docker login` | Authenticates with a registry before pushing private or owned images. |
| `docker push username/my-app:1.0` | Uploads the tagged image to a registry where you have permission to publish. |
| `docker rmi nginx:tag` | Removes a local image reference. Containers that use the image may need to be removed first; an image used by multiple tags may remain under another tag. |

### Volumes, networks, and cleanup

| Command | What it does / useful options |
|---|---|
| `docker volume create app-data` | Creates a named volume for persistent data. |
| `docker volume inspect app-data` | Shows volume details, including its mountpoint on the Docker host. |
| `docker volume rm app-data` | Removes a volume that is not in use by a container. Check the volume before deleting it. |
| `docker volume prune` | Removes unused local volumes after confirmation. Check carefully before confirming because volume data can be lost. |
| `docker container prune` | Removes all stopped containers after confirmation. |
| `docker system prune` | Removes unused Docker resources, including stopped containers, unused networks, dangling images, and build cache. Review the prompt before confirming; `-a` removes more unused images. |
| `docker compose up -d` | Creates/starts services defined in the Compose file; `-d` runs them in the background. |
| `docker compose ps` | Shows the status of services/containers managed by the current Compose project. |
| `docker compose down` | Stops and removes the Compose project's containers and networks. Named volumes are not removed by default; `-v` also removes declared named volumes and attached anonymous volumes. Use with care. |

### Quick command patterns

```bash
# Run a named container in the background
docker run -d --name web1 -p 8080:80 nginx

# Check running and stopped containers
docker ps
docker ps -a

# Open a shell in a running container
docker exec -it web1 sh

# Build and run a custom image
docker build -t my-app:1.0 .
docker run --rm my-app:1.0
```

**Important distinctions:** `docker run` creates a new container; `docker start` restarts an existing one. `docker stop` preserves the container and its writable layer; `docker rm` deletes the container and that writable layer. Data intended to outlive a container should be stored in a volume or bind mount. `docker system prune` and volume-removal commands can delete data/resources, so review their scope before confirming.

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
