# Docker Compose

Docker Compose defines and manages a multi-container application using a YAML configuration file. Instead of creating and configuring each container with separate `docker run` commands, you describe the services and their settings in a Compose file and manage the stack together.

The conventional filename is `compose.yaml`. `compose.yml` is also commonly used.

## Why use Docker Compose?

An application may depend on several services, such as:

- A web application
- A database
- A cache or message broker
- Supporting services used for testing or automation

Compose keeps the configuration for these services in one place. This makes the environment easier to start, stop, inspect, and reproduce.

## Compose V2 vs. legacy Compose V1

Use the current Compose V2 command:

```bash
docker compose
```

For example:

```bash
docker compose up -d
docker compose ps
docker compose down
```

The older standalone command was:

```bash
docker-compose
```

Compose V1 has reached end of life. Compose V2 is distributed as a Docker CLI plugin, so it is invoked as `docker compose` with a space.

Check whether Compose is available:

```bash
docker compose version
```

## A simple Compose file

Create a file named `compose.yaml`:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  cache:
    image: redis:alpine
```

This defines two services:

- `web` runs Nginx and publishes container port `80` on host port `8080`.
- `cache` runs Redis.

Start the services:

```bash
docker compose up -d
```

Check their status:

```bash
docker compose ps
```

Open the web service at:

```text
http://localhost:8080
```

View service logs:

```bash
docker compose logs
docker compose logs web
docker compose logs -f web
```

Stop and remove the containers and the Compose-created network:

```bash
docker compose down
```

Run Compose commands from the directory containing the Compose file. If the file is elsewhere or has a different name, specify it with `-f`:

```bash
docker compose -f path/to/compose.yaml up -d
docker compose -f path/to/compose.yaml ps
docker compose -f path/to/compose.yaml down
```

## How Compose networking works

By default, Compose creates a project network and connects services to it unless the configuration says otherwise.

Services attached to the same Compose network can usually reach one another using their **service names** as DNS names. For example, an application service can connect to Redis using hostname `cache` and Redis's container port `6379`:

```text
web service  ---->  cache:6379
```

The web service does not need to know Redis's container IP address. Docker's internal DNS resolves the service name on that network.

Important distinctions:

- **Service name:** the name used in the Compose file, such as `cache`.
- **Container port:** the port the service listens on inside its container, such as Redis port `6379`.
- **Published host port:** a port exposed on the Docker host. It is needed when a service must be accessed from outside the Compose network, but not merely for one Compose service to reach another.

For example, if only the `web` service needs to reach `cache`, you generally do not need to publish Redis's port to the host.

A service can communicate with another service by its service name only when both are attached to a shared network. If you define custom networks, check which services are attached to each network.

## Build a custom image with Compose

A service can use an existing image from a registry:

```yaml
services:
  web:
    image: nginx:alpine
```

Or Compose can build an image from a directory containing a Dockerfile and the application's build context:

```yaml
services:
  web:
    build:
      context: ./web
    ports:
      - "8080:80"
```

In this example, Compose builds from the `./web` directory relative to the Compose file's project/base directory. That directory should contain a Dockerfile unless another Dockerfile path is specified.

Build and start the service:

```bash
docker compose up -d --build
```

The `--build` option asks Compose to build images before starting the services.

### `image` vs. `build`

| Setting | Purpose |
|---|---|
| `image` | Names an image to use. Compose can pull it if it is not available locally. |
| `build` | Describes how to build an image from a Dockerfile and build context. |
| Both together | Names the image produced by the build; useful when you want to control the resulting image name/tag. |

Example using both:

```yaml
services:
  web:
    build:
      context: ./web
    image: example-web:1.0
```

A build context is the directory whose files are available to the Docker build. Use `.dockerignore` in that directory to exclude unnecessary files and avoid including files that should not be sent as build context.

## Example: the Ansible Server Usage Monitor lab

The `ansible-server-usage-monitor` project uses Compose to manage a small lab environment rather than starting each container individually.

Its Compose configuration defines three simulated Linux servers and a Mailpit service used to test email reporting. The simulated servers expose SSH on different host ports so that the Ansible controller running in WSL2 can connect through `localhost`:

| Service | Host port | Container port | Purpose |
|---|---:|---:|---|
| `server1` | `2221` | `22` | Simulated managed server |
| `server2` | `2222` | `22` | Simulated managed server |
| `server3` | `2223` | `22` | Simulated managed server |
| Mailpit SMTP | `1025` | `1025` | Local email test endpoint |
| Mailpit Web UI | `8025` | `8025` | Inspect test emails in a browser |

From the project root, the environment is managed with:

```bash
docker compose -f docker/compose.yml up -d
docker compose -f docker/compose.yml ps
docker compose -f docker/compose.yml down
```

The project also builds a custom Ubuntu-based image for the simulated servers. That image contains the SSH server, the Ansible account, and the `/data` directory that the playbook monitors.

This demonstrates two Compose responsibilities:

1. **Build or select images** for the services.
2. **Create and manage the containers, networks, and port mappings** that make up the lab environment.

The project has a custom network for the simulated servers. Mailpit is configured separately, so do not assume every service in this particular project can resolve every other service by name; services must share a network to communicate through Compose's service-name DNS.

## Useful commands

| Command | Purpose |
|---|---|
| `docker compose version` | Shows the installed Compose version. |
| `docker compose config` | Parses and renders the effective Compose configuration. Useful for catching YAML/configuration errors. |
| `docker compose up` | Creates and starts the services. Runs attached to the terminal by default. |
| `docker compose up -d` | Starts the services in the background. |
| `docker compose up -d --build` | Builds configured images and starts the services. |
| `docker compose ps` | Lists the services/containers for the current Compose project. |
| `docker compose logs` | Displays service logs. |
| `docker compose logs -f <service>` | Follows logs for a particular service. |
| `docker compose stop` | Stops services without removing their containers. |
| `docker compose start` | Starts existing stopped service containers. |
| `docker compose down` | Stops and removes project containers and networks. |
| `docker compose pull` | Pulls images configured for services. |

### Data and cleanup caution

`docker compose down` does not remove named volumes by default, so named-volume data normally remains. Adding `-v` removes declared named volumes and attached anonymous volumes and can delete stored data:

```bash
docker compose down -v
```

Use this only when you intentionally want to remove those volumes and their data.

## Key takeaways

- Compose describes a multi-container application in YAML.
- Use the current `docker compose` command; `docker-compose` is the legacy V1 command.
- `docker compose up -d` starts a stack in the background.
- Compose normally creates a project network, and services on a shared network can resolve one another by service name.
- `image` selects an image; `build` builds one from a Dockerfile and build context.
- Publish ports when a service needs to be reached from the Docker host or outside the Compose network.
- `docker compose down` removes project containers and networks, but named volumes are retained unless you request volume removal.

