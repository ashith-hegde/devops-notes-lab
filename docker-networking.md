# Docker Networking

These notes combine the hands-on Docker networking lab performed in WSL2 with the core concepts needed to understand Docker network modes, port publishing, user-defined networks, and container name resolution.

## Goal

Run Nginx containers, publish their ports to the host, inspect their network configuration, and understand how Docker bridge networking and network namespaces work.

## Default Docker Networks

A typical Docker Engine installation creates three built-in networks:

```bash
docker network ls
```

- `bridge` — the default bridge network for containers that do not specify another network.
- `host` — uses the host's network namespace on supported platforms.
- `none` — disables normal container networking, leaving loopback available.

These modes have different behavior; they are not three equivalent bridge networks.

## Bridge Networking

The default `bridge` network is a private network implemented on the Docker host. On a typical Linux installation, Docker connects it to a host-side bridge interface such as `docker0`.

A container started without `--network` is normally attached to the default bridge:

```bash
docker run -d --name web1 nginx
```

Containers attached to the same bridge can communicate using their IP addresses, subject to firewall rules and the services listening inside the containers. The default bridge does not provide the same automatic container-name DNS resolution as a user-defined bridge.

Inspect the network configuration and IP address:

```bash
docker inspect web1
docker inspect web1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

A container may receive an address from a Docker-managed subnet such as `172.17.0.0/16`, but the actual subnet and address depend on the host configuration.

> **Note:** `docker run ubuntu` often exits immediately because the image's default command finishes. Use an interactive shell or a long-running application when you need a container to remain running for inspection.

## Port Publishing

A containerized application can be running successfully inside a container without being reachable through a port on the Docker host. The `-p` option publishes a container port on a host port.

Basic syntax:

```text
-p <host-port>:<container-port>
```

For example:

```bash
docker run -d --name web1 -p 8080:80 nginx
```

This maps host port `8080` to container port `80`:

```text
Docker host                      Container
port 8080  --------------------> port 80
```

Open the service at:

```text
http://localhost:8080
```

In the WSL2 lab, the Nginx welcome page appeared at `http://localhost:8080`. Accessing `http://localhost:80` did not work because this container published host port `8080`, not host port `80`.

The host port and container port do not have to match. For example:

```bash
docker run -d --name web-alt -p 80:5000 <image>
```

maps host port `80` to container port `5000`, assuming the application inside the image listens on port `5000`.

### View published ports

```bash
docker port web1
```

Example output from the lab:

```text
80/tcp -> 0.0.0.0:8080
80/tcp -> [::]:8080
```

This indicates that the published port is bound to all IPv4 and IPv6 host addresses, respectively, as reported by Docker. Exact output can vary by platform and configuration.

To inspect the published-port configuration:

```bash
docker inspect web1 --format '{{json .NetworkSettings.Ports}}'
```

Publishing to all host interfaces may make the service reachable beyond the local machine, depending on routing and firewall rules. To bind the published port to loopback explicitly:

```bash
docker run -d --name web-local -p 127.0.0.1:8082:80 nginx
```

## Hands-On Lab: Verify Nginx and Host Ports

Start the first container:

```bash
docker run -d --name web1 -p 8080:80 nginx
```

Verify its status:

```bash
docker ps
```

In the lab, its status was `Up`.

Test the application from inside the container:

```bash
docker exec web1 sh -c 'curl -I http://localhost:80 | head -n 1'
```

Observed output:

```text
HTTP/1.1 200 OK
```

Inside the container, `localhost` refers to that container's own network namespace. This command works only if `curl` is installed in the image; if it is unavailable, test from the host or use another tool present in the container.

### Check host listeners

In the WSL2 lab, this command showed no listener for host port `80`:

```bash
ss -tulpn | grep ':80 '
```

This command showed a listener for host port `8080`:

```bash
ss -tulpn | grep ':8080 '
```

The exact listener output can vary with Docker Engine, Docker Desktop, WSL2, and the host networking implementation. Use the commands as diagnostic checks, not as a universal guarantee of how Docker publishes ports on every platform.

## Multiple Containers Using the Same Container Port

Start a second Nginx container with a different host port:

```bash
docker run -d --name web2 -p 8081:80 nginx
```

Both containers can run at the same time:

- `web1` — host port `8080` to container port `80`
- `web2` — host port `8081` to container port `80`

Both containers can use container port `80` because each has its own network namespace. When publishing both services on the same host address, their host ports must not conflict.

Test them at:

```text
http://localhost:8080
http://localhost:8081
```

## Container IP Addresses and Network Mode

Inspect the IP addresses:

```bash
docker inspect web1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker inspect web2 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Example addresses observed in the lab:

```text
web1 = 172.17.0.2
web2 = 172.17.0.3
```

These are examples, not fixed addresses. Docker may assign different IPs when containers or networks are recreated.

Inspect the address and network mode together:

```bash
docker inspect web1 --format 'IP={{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} NetworkMode={{.HostConfig.NetworkMode}}'
```

Example output:

```text
IP=172.17.0.2 NetworkMode=bridge
```

This indicates that the container used bridge networking in that lab.

## Host Networking

With host networking, a container shares the host's network namespace rather than receiving its own isolated network namespace.

```bash
docker run --rm --network host nginx
```

On Linux, Nginx attempts to listen directly on the host's port `80`; no `-p` mapping is needed.

- The container does not get a separate container network IP in host mode.
- Port conflicts are host port conflicts: two processes cannot both bind the same address and port combination.
- Network isolation between the container and host is reduced.
- Support and behavior vary by platform. Check the current Docker Desktop documentation for host-networking support and requirements.

## None Network

The `none` network disables normal external container networking:

```bash
docker run --rm -it --network none alpine sh
```

The container retains its loopback interface, but Docker does not connect it to a normal bridge network. It therefore cannot use ordinary Docker networking to communicate with other containers or external hosts.

This mode can be useful for workloads that do not need network access. It is not a complete security boundary by itself; other container permissions and configuration still matter.

## User-Defined Bridge Networks

For multi-container applications on a single Docker host, a user-defined bridge network is usually preferable to the default bridge.

Create a network:

```bash
docker network create --driver bridge app-net
```

Attach containers to it:

```bash
docker run -d --name app1 --network app-net nginx
docker run -d --name app2 --network app-net nginx
```

Inspect the network and its attached containers:

```bash
docker network inspect app-net
```

User-defined bridge networks provide several advantages:

- Containers attached to the same network can communicate with one another.
- Docker's embedded DNS can resolve container names and network aliases on the network.
- Network membership can be managed explicitly.
- Separate networks can help segment groups of containers.

A user-defined bridge does **not** automatically make a network internal-only or block all external access. If you need a bridge network intended to have no external connectivity, consider the `--internal` option and verify that the resulting behavior meets your requirements.

### Choose a subnet (optional)

Docker can allocate a subnet automatically. If you need to specify one, choose a range that does not overlap with your LAN, VPN, cloud network, or other Docker networks:

```bash
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  app-net
```

Do not copy this example subnet blindly; check for address conflicts first.

## Embedded DNS and Container Names

On user-defined bridge networks, Docker provides an embedded DNS server inside containers, commonly reachable at `127.0.0.11`. Containers can use another container's name or a configured network alias instead of relying on an IP address.

This is preferable to hard-coding container IP addresses because IP addresses can change when containers are recreated.

The default bridge network does not provide the same automatic name-based resolution behavior by default.

Test name resolution from a temporary container:

```bash
docker run --rm --network app-net busybox:1.36 nslookup app1
```

The target container must be running and attached to `app-net` for this example to resolve.

## Docker Compose Networking

Docker Compose normally creates a project network and attaches the project's services to it unless the configuration says otherwise. Services on a shared Compose network can resolve one another using their **service names**.

For example, an application can connect to a database at `db:5432` if both services share a network and the database listens on port `5432`.

Container-to-container communication uses the destination's container port. You generally do not need to publish that port to the host unless clients outside the Compose network need to reach it.

See `docker-compose.md` for Compose configuration, service networking, and the Ansible Server Usage Monitor lab.

## Docker Logs

View the latest Nginx logs:

```bash
docker logs web1 | tail -n 5
```

The lab showed Nginx startup messages, an HTTP `200` for `/`, and an HTTP `404` for `/favicon.ico`. The favicon response is usually harmless when the requested file is not present; it does not by itself indicate that Nginx is unhealthy.

## Under the Hood

Docker networking relies on several Linux networking features:

- **Network namespaces** — isolate network interfaces, routing tables, and related network state for a container.
- **Virtual Ethernet (veth) pairs** — connect a container's network interface to a host-side interface, often connected to a Linux bridge.
- **Linux bridge** — forwards traffic between interfaces connected to the bridge.
- **IP routing, firewall rules, and NAT** — support traffic forwarding and published-port behavior, depending on network mode and Docker configuration.
- **Embedded DNS** — resolves container names and network aliases on user-defined networks.

The exact implementation can vary by operating system, Docker version, and configuration. The bridge and veth description is the typical model for bridge networking on a Linux Docker host.

## Useful Network Commands

| Command | Purpose |
|---|---|
| `docker network ls` | Lists Docker networks. |
| `docker network inspect <network>` | Displays network configuration and attached containers. |
| `docker network create <name>` | Creates a user-defined network. |
| `docker network connect <network> <container>` | Connects an existing container to a network. |
| `docker network disconnect <network> <container>` | Disconnects a container from a network. |
| `docker network rm <network>` | Removes a network that is no longer in use. |
| `docker inspect <container>` | Displays container configuration, including network settings. |
| `docker port <container>` | Lists published port mappings. |
| `docker exec <container> <command>` | Runs a command inside a running container for troubleshooting. |
| `docker logs <container>` | Displays the container's logs. |

## Key Takeaways

- Containers normally use the default bridge when no network is specified.
- Container ports exist within a container's network namespace; published host ports expose services outside that namespace.
- Multiple containers can listen on the same container port while using different published host ports.
- User-defined bridge networks provide automatic container-name DNS and explicit network membership.
- Use container or Compose service names rather than relying on dynamic IP addresses.
- Host networking shares the host's network namespace on supported platforms; `none` disables normal networking.
- `docker inspect`, `docker port`, `docker network inspect`, `docker exec`, and `docker logs` are useful troubleshooting tools.
- Network namespaces, veth pairs, bridges, routing, firewall rules, and NAT help implement Docker networking on Linux.

## Cleanup

Remove the two lab containers when you are finished:

```bash
docker rm -f web1 web2
```

Remove the user-defined network only after disconnecting/removing any containers attached to it:

```bash
docker network rm app-net
```

## Related Notes

- `docker-basics.md` — Docker overview and command reference.
- `docker-compose.md` — Compose configuration and service-name networking.
- `docker-linux-namespaces-and-cgroups.md` — Linux namespaces and resource controls.
- `docker-storage-architecture.md` — Docker storage internals.
- `docker-volumes-and-bind-mounts.md` — Persistent storage and mount types.

