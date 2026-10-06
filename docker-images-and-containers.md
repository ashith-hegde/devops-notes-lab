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

## Interactive mode

Interactive mode is commonly used when you need to provide input to a container or work with a shell interactively.

```text
-i  → keeps STDIN open
-t  → allocates a pseudo-terminal (TTY)
-it → combines both
```

Example:

```bash
docker run -it <image>
```

`-i` keeps standard input open, while `-t` allocates a pseudo-terminal. Together, `-it` is commonly used for an interactive terminal session.

Without `-i`, attaching a terminal does not necessarily mean keyboard input will be passed to the container's standard input.

### Piping input into a container

`-i` can also be useful when input is provided through a pipe rather than an interactive terminal.

Example:

```bash
cat schema.sql | docker run -i postgres psql -U postgres -d mydb
```

Here, the contents of `schema.sql` are passed to the container process through standard input.

The `-t` option is generally not needed for piped input because there is no interactive terminal session.

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

A Dockerfile is an instruction-and-argument format. Instructions are conventionally written in uppercase.

| Instruction | Purpose |
|---|---|
| `FROM` | Defines the base image. A Dockerfile normally starts with `FROM`. |
| `RUN` | Executes commands during the image build. |
| `COPY` | Copies files or directories from the build context into the image. |
| `ADD` | Similar to `COPY` with additional behavior; `COPY` is generally preferred for ordinary file copying. |
| `WORKDIR` | Sets the working directory for subsequent instructions and the container process. |
| `ENV` | Defines environment variables in the image. |
| `EXPOSE` | Documents the port the application is intended to listen on; it does not publish the port to the host by itself. |
| `CMD` | Provides the default command or arguments when a container starts. |
| `ENTRYPOINT` | Configures the main executable for the container. |

Example structure:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV APP_ENV=production
EXPOSE 5000

ENTRYPOINT ["python"]
CMD ["app.py"]
```

`CMD` provides defaults that can be replaced by a runtime command. `ENTRYPOINT` defines the executable the container is intended to run; runtime arguments are normally appended to it.

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

### CMD, ENTRYPOINT, and ARG

These instructions solve different problems:

| Instruction | Purpose | When it applies |
|---|---|---|
| `CMD` | Provides a default command or default arguments for the container | Runtime |
| `ENTRYPOINT` | Defines the main executable for the container | Runtime |
| `ARG` | Defines a variable that can be supplied during the image build | Build time |

#### Why does `docker run ubuntu` exit?

A container only remains running while its main process is running.

The Ubuntu image uses Bash as its default command:

```dockerfile
CMD ["bash"]
```

When you run:

```bash
docker run ubuntu
```

Docker starts Bash without an interactive terminal attached. Bash has no interactive session to maintain, so it exits, and the container exits with it.

This illustrates the basic container lifecycle:

```text
Container starts
      |
      v
Main process starts
      |
      v
Process completes/exits
      |
      v
Container exits
```

Containers are therefore normally designed to run a specific application, service, or task rather than behave like a full virtual machine.

#### Override `CMD` at runtime

A command appended to `docker run` replaces the image's default `CMD`.

For example:

```bash
docker run ubuntu sleep 5
```

This starts `sleep 5` instead of the Ubuntu image's default Bash command.

#### Define a different default with `CMD`

You can create your own image:

```dockerfile
FROM ubuntu
CMD ["sleep", "5"]
```

Now:

```bash
docker run my-sleeper
```

runs:

```text
sleep 5
```

The command can still be replaced at runtime:

```bash
docker run my-sleeper sleep 10
```

which runs:

```text
sleep 10
```

#### Use `ENTRYPOINT` for a fixed executable

`ENTRYPOINT` is useful when the image is intended to run a particular executable and the runtime argument should be treated as an argument to that executable.

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
```

Now:

```bash
docker run ubuntu-sleeper 10
```

runs:

```text
sleep 10
```

#### Use `ENTRYPOINT` and `CMD` together

A common pattern is to use `ENTRYPOINT` for the executable and `CMD` for its default arguments:

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

The default behavior is:

```text
sleep 5
```

But the default argument can be overridden:

```bash
docker run ubuntu-sleeper 10
```

which runs:

```text
sleep 10
```

This pattern is useful when the executable should remain fixed while its default arguments remain configurable.

#### Override `ENTRYPOINT` at runtime

If you need to replace the image's entrypoint completely, use `--entrypoint`:

```bash
docker run --entrypoint /bin/sh ubuntu-sleeper
```

This replaces the image's configured `ENTRYPOINT` with `/bin/sh`.

#### Shell form vs exec form

Dockerfile commands such as `CMD` and `ENTRYPOINT` can be written in shell form or exec form.

Exec form:

```dockerfile
CMD ["sleep", "5"]
ENTRYPOINT ["sleep"]
```

Shell form:

```dockerfile
CMD sleep 5
ENTRYPOINT sleep
```

The **exec form** is generally preferred for applications because Docker starts the specified executable directly, which gives clearer process and signal behavior.

#### Build-time `ARG`

`ARG` is different from `CMD` and `ENTRYPOINT` because it is used during image construction rather than when the container starts.

Example:

```dockerfile
FROM ubuntu

ARG APP_VERSION=1.0
RUN echo "Building version ${APP_VERSION}"
```

Build with the default:

```bash
docker build -t my-app .
```

Override it during the build:

```bash
docker build --build-arg APP_VERSION=2.0 -t my-app .
```

`ARG` is primarily a build-time value. If a value is needed by the application when the container runs, `ENV` is generally the more appropriate mechanism.

Do not use `ARG` for secrets. Build arguments can be exposed through image build metadata/history, which is why BuildKit secret mounts are preferred for sensitive build-time credentials.

---

## Build context

Command:

```bash
docker build -t my-first-image .
```

The final `.` means **current directory** and becomes the **build context**.

Docker sends files from that directory to the Docker daemon. Only files inside the build context can be copied into the image.

A `.dockerignore` file can be used to exclude files that do not need to be sent as build context. Keeping the build context focused can reduce unnecessary data transfer and avoid accidentally including files that should not be part of the image build.

---

## Docker image layers

Docker builds images using layers. Each relevant Dockerfile instruction can contribute a filesystem layer, and later layers build on top of earlier ones.

Example:

```text
Dockerfile

FROM python:3.12-slim     ──> Layer 1: base image
WORKDIR /app              ──> Layer 2
COPY requirements.txt .   ──> Layer 3
RUN pip install ...       ──> Layer 4
COPY . .                  ──> Layer 5
```

Conceptually:

```text
+-----------------------------+
| Layer 5: application code   |
+-----------------------------+
| Layer 4: Python packages    |
+-----------------------------+
| Layer 3: requirements file  |
+-----------------------------+
| Layer 2: /app configuration |
+-----------------------------+
| Layer 1: base image         |
+-----------------------------+
```

Inspect image layer history with:

```bash
docker history my-first-image
```

### Build cache

Docker can reuse cached build results when the relevant inputs and instructions have not changed. Dockerfile instruction order therefore affects how much of a build can be reused.

For example, keeping dependency installation separate from application source copying can allow the dependency-related step to remain cached when only application code changes:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

This can make repeated image builds faster.

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


