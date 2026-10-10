# Docker Engine

Docker Engine is the software platform that builds and runs containers. The **Docker host** is the machine or virtual machine where the Engine runs; the Engine is not the host itself.

These notes cover the client–server architecture, the runtime stack, and secure remote access. The broader overview and command reference remain in [`docker-basics.md`](docker-basics.md).

## 1. Client–server architecture

```text
User / script
     |
     v
Docker CLI (`docker`)
     |
     | Docker Engine API
     v
Docker daemon (`dockerd`)
     |
     v
containerd
     |
     v
runc
     |
     v
Linux kernel
```

This is a simplified view of a typical Linux Docker Engine setup. The actual startup path has more detail, and not every Docker operation passes through every component shown.

### Docker CLI (`docker`)

The command-line client used to issue commands such as:

```bash
docker run nginx
docker ps
docker stop web1
docker image rm nginx
```

The CLI sends requests to a Docker Engine API endpoint. On a typical Linux installation, the default endpoint is a local Unix socket.

### Docker daemon (`dockerd`)

A background service that exposes the Docker Engine API and coordinates Docker-specific operations. It manages resources such as containers, images, networks, and volumes, and works with lower-level runtime components to run containers.

```bash
sudo systemctl status docker
sudo journalctl -u docker
```

These commands apply to systemd-based Linux hosts.

### Docker Engine API

The API is the programmatic interface clients use to request Docker operations. The CLI is one client; other tools and scripts can use the API too. It is commonly described as a REST API and can be exposed through a Unix socket, TCP socket, or SSH-mediated connection. It is not necessarily a separate service that must be installed and managed independently from `dockerd`.

| Component | Main responsibility |
|---|---|
| Docker CLI (`docker`) | Lets people and scripts issue Docker commands |
| Docker Engine API | Defines the requests clients can make |
| Docker daemon (`dockerd`) | Exposes the API and coordinates Docker resource management |
| `containerd` | Manages core container lifecycle and runtime coordination |
| `runc` | Creates and starts a container process according to the OCI runtime specification |
| Linux kernel | Provides process isolation and resource-control mechanisms |

## 2. What happens when you run a container?

For example:

```bash
docker run -d --name web1 nginx
```

At a high level:

1. The CLI sends a request to the Docker Engine API.
2. The daemon checks whether the image is available locally and pulls it from a registry if needed.
3. Docker coordinates container creation through lower-level runtime components.
4. The runtime sets up the container process and required isolation using Linux kernel features.
5. Docker reports the result to the CLI.

This is a conceptual sequence, not a guarantee that every operation follows exactly these steps. For example, an image that is already present does not need to be pulled again.

## 3. Underneath the daemon: `containerd` and `runc`

### Docker daemon (`dockerd`)

Responsible for Docker-specific functionality, including the Docker API and management of Docker resources.

### `containerd`

A container runtime component that manages container lifecycle tasks and coordinates execution through a runtime such as `runc`. `containerd` is also used outside Docker. Kubernetes can use it through the Container Runtime Interface (CRI), but Kubernetes does not require Docker Engine to run containers.

### `runc`

A low-level container runtime. It creates and starts a container process using configuration supplied by the higher-level runtime. On Linux, this involves kernel features such as namespaces and cgroups.

`runc` implements the Open Container Initiative (OCI) Runtime Specification, which standardizes how a container runtime creates and runs a container.

### Why multiple layers?

- **Docker Engine:** Docker's user-facing container platform and resource-management interface.
- **`containerd`:** container lifecycle and runtime coordination.
- **`runc`:** low-level creation and startup of the container process.
- **Linux kernel:** mechanisms that enforce process, namespace, and resource isolation.

The stack diagram helps explain responsibilities, but avoid describing it as exactly three handoffs in every container start.

## 4. Local Docker Engine access

On many Linux installations, the CLI communicates with the daemon through this Unix socket:

```text
/var/run/docker.sock
```

Useful commands:

```bash
docker --version
docker version
docker info
docker context ls
```

- `docker --version` prints the CLI version; it does **not** prove the daemon is running.
- `docker version` shows client and server version information when the daemon is reachable.
- `docker info` reports details about the connected Engine and host.
- `docker context ls` lists configured Docker contexts.

Access to the Docker socket is highly privileged. Membership in the `docker` group commonly grants root-equivalent control over a rootful Docker host. Grant it only to trusted users.

## 5. Connecting to a remote Docker Engine

The CLI can connect to a remote Engine if the remote endpoint is configured and reachable and the client is authorized.

### Recommended approach: SSH

Docker can use SSH to reach the remote host and forward requests to its Docker socket.

Create a context:

```bash
docker context create remote-engine \
  --docker host=ssh://user@docker-host.example.com
```

Switch to it and verify the connection:

```bash
docker context use remote-engine
docker info
docker ps
```

Return to the default context when finished:

```bash
docker context use default
```

Replace the example username and hostname. SSH must be configured, and the remote user must have permission to access the Docker socket. Use SSH keys and normal host-key verification practices.

For a one-off command:

```bash
docker -H ssh://user@docker-host.example.com ps
```

### Alternative: TLS-secured TCP

A remote daemon can expose its API over TCP with TLS and client-certificate verification. The conventional port is **2376**.

```bash
docker --tlsverify \
  -H=tcp://docker-host.example.com:2376 \
  ps
```

This command assumes the required CA certificate and client certificate/key are configured through Docker's TLS options or environment variables, and that the server is configured to require and validate trusted client certificates. The command alone does not configure TLS on the daemon.

### Avoid unauthenticated TCP

| Endpoint / port | Meaning | Guidance |
|---|---|---|
| `unix:///var/run/docker.sock` | Local Unix socket on a typical Linux host | Restrict local permissions |
| `ssh://user@host` | Remote access through SSH | Usually the simplest recommended remote option |
| TCP `2376` | Conventionally used for TLS-secured Docker API access | Use proper certificate verification and network restrictions |
| TCP `2375` | Conventionally used for unencrypted, unauthenticated Docker API access | Do not expose to an untrusted network |

**Important:** A port number does not guarantee security. Security depends on daemon configuration, authentication, certificate verification, firewall rules, and network exposure. An attacker who can control a rootful Docker daemon may be able to gain control of the host. Do not expose the Docker API without strong access controls.

The default behavior for insecure TCP listeners can vary by Engine version and configuration. Do not rely on a particular version refusing such a listener as your security control.

## 6. Quick comparison

| Term | Remember it as |
|---|---|
| Docker host | Machine running Docker Engine |
| Docker Engine | Platform for building and running containers |
| Docker CLI | Command-line client |
| Docker Engine API | Interface used by clients and tools |
| `dockerd` | Daemon exposing the API and managing Docker resources |
| `containerd` | Container lifecycle/runtime management component |
| `runc` | Low-level OCI runtime |
| OCI Runtime Specification | Standard contract for creating and running containers |
| Docker context | Named configuration for a Docker endpoint |

## Key takeaways

- Docker Engine is software that runs on a Docker host.
- The CLI sends requests to the daemon through the Docker Engine API.
- The API is exposed by the daemon; it is not necessarily a separately managed server.
- `dockerd`, `containerd`, and `runc` have different responsibilities.
- Containers rely on kernel features for isolation.
- Remote access should use SSH or correctly configured TLS, not an exposed unauthenticated API.
- Treat access to a rootful Docker socket or daemon as highly privileged.

## Further reading

- [Docker Engine overview](https://docs.docker.com/engine/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Protect access to the Docker daemon socket](https://docs.docker.com/engine/security/protect-access/)
- [Configure remote access for the Docker daemon](https://docs.docker.com/engine/daemon/remote-access/)
- [containerd documentation](https://containerd.io/)
- [Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)

