# Docker Images and Containers

Notes from hands-on Docker practice on Ubuntu running inside WSL2.

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

## Key takeaways

- Containers are isolated processes, not virtual machines.
- Containers share the host Linux kernel.
- PID 1 controls container lifetime.
- `docker exec -it` is used to troubleshoot running containers.
- Container writable data is ephemeral unless stored in a volume or bind mount.
- Docker images are built from Dockerfiles using layered filesystem changes.
- The build context determines which files are available during image build.

---

## Images and Registries

Docker images can be shared through container registries. Docker Hub is a widely used public registry containing images for common operating systems, databases, applications, and other services.

Other container registries include:

- GitHub Container Registry (GHCR)
- Google Artifact Registry
- Amazon Elastic Container Registry (Amazon ECR)
- Azure Container Registry

Images can be pulled from registries and used to create containers. For example:

```bash
docker run ansible
docker run mongodb
docker run redis
```

The image name identifies the image that Docker needs to obtain before creating the container when it is not already available locally.

