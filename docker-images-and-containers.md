# Docker Images and Containers

Notes from hands-on Docker practice on Ubuntu running inside WSL2.

## Images vs Containers

A **Docker image** is a packaged template used to create containers. A **container** is an instance of an image with its own isolated processes and environment. One image can be used to create multiple containers.

```text
             Docker image
          (template/package)
                  |
          create containers
          /       |       \
         v        v        v
    Container  Container  Container
       #1         #2         #3
```

Images can be obtained from a registry or built from a Dockerfile. If a suitable public image is unavailable, you can build your own image and publish it to a registry such as Docker Hub, subject to the registry's access and publishing rules.

## Running a container in detached mode

Start a container in the background:

```bash
docker run -d --name ubuntu-test ubuntu sleep infinity
```

- `-d` = detached mode (terminal is not attached)
- `sleep infinity` keeps PID 1 running

Check running containers:

```bash
docker ps
```

Stop the container:

```bash
docker stop ubuntu-test
```

View all containers including stopped ones:

```bash
docker ps -a
```

---

## Entering a running container

Start a long-running container:

```bash
docker run -d --name ubuntu-shell ubuntu sleep infinity
```

Open an interactive shell inside the running container:

```bash
docker exec -it ubuntu-shell bash
```

- `docker exec` runs a new process inside an existing container
- `-i` keeps standard input open
- `-t` allocates a terminal

Exit the shell with `exit`. The container continues running because PID 1 is still `sleep infinity`.

---

## PID 1 and container lifecycle

Inside the container:

```bash
ps -p 1 -o pid,comm
```

Output:

```text
PID COMMAND
1   sleep
```

A container stays running as long as its PID 1 process is running.

- PID 1 exits → container exits
- Additional processes started with `docker exec` do not become PID 1

---

## Container persistence experiment

Create a file:

```bash
docker exec ubuntu-shell bash -c 'echo persistent-test > /tmp/persist.txt'
```

Remove the container:

```bash
docker stop ubuntu-shell
docker rm ubuntu-shell
```

Create a new container from the same image and check the file:

```bash
docker run -d --name ubuntu-shell-2 ubuntu sleep infinity
docker exec ubuntu-shell-2 cat /tmp/persist.txt
```

Result:

```text
No such file or directory
```

### Observation

Container files are stored in the container's writable layer and are deleted when the container is removed.

---

## Stopped vs removed containers

| Action | Container exists? | Writable data exists? |
|---|---|---|
| `docker stop` | Yes | Yes |
| `docker start` | Yes | Yes |
| `docker rm` | No | No |
| `docker rm -f` | No | No |

Stopping preserves data; removing deletes it.

---

## Sharing files with the host (bind mount)

Create a host directory:

```bash
mkdir -p ~/shared
```

Run a container with a bind mount:

```bash
docker run --rm -v ~/shared:/data ubuntu bash -c 'echo hi > /data/test.txt'
```

Read the file on the host:

```bash
cat ~/shared/test.txt
```

Output:

```text
hi
```

Files written to `/data` are stored directly on the host.

---

## How an Image Gets Built and Shared

A common image workflow is:

```text
Dockerfile -- docker build --> Image -- docker push --> Registry
                                     ^
                                     |
                         docker pull from registry
```

The Dockerfile records the instructions for assembling an image. Developers and operations teams can collaborate on the Dockerfile so that application requirements and operational needs are represented in a repeatable build. The resulting image can then be run on compatible hosts with Docker, helping provide a consistent application environment.

Typical commands:

```bash
docker build -t my-app:1.0 .
docker tag my-app:1.0 username/my-app:1.0
docker push username/my-app:1.0
docker pull username/my-app:1.0
```

The `docker tag` and `docker push` examples assume you have a Docker Hub account, have authenticated with `docker login`, and are using a repository you are allowed to publish to. Replace the example names with your own image and registry details.

---

## Building a custom image

Dockerfile:

```dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get install -y curl

CMD ["bash", "-c", "echo Hello from my first Docker image"]
```

Build:

```bash
docker build -t my-first-image .
```

Run:

```bash
docker run --rm my-first-image
```

Output:

```text
Hello from my first Docker image
```

---

## Dockerfile instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image |
| `RUN` | Execute commands during build |
| `CMD` | Default command when container starts |

Example:

```bash
docker run --rm my-first-image bash -c 'echo override'
```

Output:

```text
override
```

The runtime command overrides the image's default `CMD`.

---

## Build context

Command:

```bash
docker build -t my-first-image .
```

The final `.` means **current directory** and becomes the **build context**.

Docker sends files from that directory to the Docker daemon. Only files inside the build context can be copied into the image.

---

## Useful inspection commands

```bash
docker inspect ubuntu-shell --format '{{.State.Status}}'
docker inspect ubuntu-shell --format '{{.Config.Cmd}}'
```

Example output:

```text
exited
[sleep infinity]
```

These commands are useful for troubleshooting container state and startup commands.

---

## Command reference

This file explains commands in the context of image/container behaviour. For the consolidated command reference—including common flags such as `-d`, `--name`, `--rm`, `-it`, port mapping, image management, cleanup, volumes, networking, and Compose—see [`docker-basics.md`](docker-basics.md).

The commands used in this file include:

- `docker run` — creates a new container from an image; options such as `-d`, `--name`, `--rm`, and `-v` control how it runs.
- `docker ps` / `docker ps -a` — list running containers / all containers including stopped ones.
- `docker stop` / `docker start` / `docker rm` — stop, restart, or remove a container.
- `docker exec -it <container> bash` — start an interactive shell as an additional process in a running container.
- `docker build -t <name>:<tag> .` — build an image from the Dockerfile in the current build context.
- `docker tag`, `docker push`, and `docker pull` — tag an image, publish it to a registry, or download it from a registry.
- `docker images` / `docker rmi` — list local images or remove a local image reference.
- `docker inspect` — view detailed container or image metadata.

## Key takeaways

- Containers are isolated processes, not virtual machines.
- Containers share the host Linux kernel.
- PID 1 controls container lifetime.
- `docker exec -it` is used to troubleshoot running containers.
- Container writable data is ephemeral unless stored in a volume or bind mount.
- Docker images are built from Dockerfiles using layered filesystem changes.
- The build context determines which files are available during image build.- One image can be used to create multiple containers.
- Images can be built from Dockerfiles and shared through registries.
- Tags identify image references; avoid relying on a moving `latest` tag for repeatable deployments.
- A container's main process controls its lifecycle.

---

## Images, Registries, and Tags

Container registries store and distribute images. Docker Hub is a widely used public registry containing images for common operating systems, databases, applications, and other services.

Other registries include:

- GitHub Container Registry (GHCR)
- Google Artifact Registry
- Amazon Elastic Container Registry (Amazon ECR)
- Azure Container Registry

If an image is not available locally, Docker can pull it from a registry when running a command such as:

```bash
docker run ansible
docker run mongodb
docker run redis
```

The examples use short image names. Docker resolves these to the appropriate registry/name defaults when no registry is specified, and it uses the default `latest` tag when no tag is supplied.

### Image tags

A tag identifies a particular image reference, often associated with a software version or variant. Specify a tag after a colon:

```bash
docker run redis:7.4
docker images
docker rmi redis:7.4
```

Example `docker images` output:

```text
REPOSITORY   TAG     IMAGE ID
redis        8.6.1   036...
redis        7.4     f5d...
```

To find available tags, check the image's page in its registry, such as Docker Hub.

**Important:** `latest` is a tag name, not a guarantee that the image is the newest or most stable version. It is a moving reference that publishers can update. For repeatable deployments, prefer an intentional version tag such as `redis:7.4-alpine`, after checking that it matches your compatibility and security requirements. Even a version tag can be updated in some registries; digest pinning provides a more immutable image reference when strict reproducibility is required.


