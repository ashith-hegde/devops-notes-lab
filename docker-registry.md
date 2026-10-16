# Docker Registry

Container registries store, organize, and distribute container images. When Docker needs an image that is not available locally, it can pull the image from a registry. After building an image, you can tag it for a registry and push it so other machines or deployment systems can pull it.

These notes focus on registry concepts and workflows. For image fundamentals, tags, and image-building, see [`docker-images-and-containers.md`](docker-images-and-containers.md).

## What Is a Container Registry?

A **container registry** is a service that stores and distributes container images. Registries commonly support:

- Uploading images with `docker push`.
- Downloading images with `docker pull`.
- Organizing images into repositories and tags.
- Controlling access to private images through authentication and permissions.
- Storing multiple versions of an application image.

A registry is not the same as a running container or a local Docker image cache.

```text
Developer / CI pipeline
        |
        | build and tag
        v
   Local Docker image
        |
        | docker push
        v
  Container registry
        |
        | docker pull
        v
Deployment host / server
        |
        | docker run
        v
   Running container
```

The registry distributes images; Docker Engine on each host pulls and runs them. Registry storage and local Docker image storage are separate concerns. See [`docker-storage-architecture.md`](docker-storage-architecture.md) for local image layers and Docker's storage layout.

## Anatomy of an Image Reference

Consider:

```bash
docker run nginx
```

`nginx` is a shorthand image reference. Docker resolves it using default naming conventions. A typical fully qualified reference is:

```text
docker.io/library/nginx:latest
```

The general form is:

```text
[REGISTRY_HOST[:PORT]/][NAMESPACE/]REPOSITORY[:TAG][@DIGEST]
```

| Component | Example | Meaning |
|---|---|---|
| Registry host | `docker.io` | Registry endpoint |
| Namespace / owner | `library` | Account, organization, or namespace containing the repository |
| Repository | `nginx` | Image repository name |
| Tag | `latest` | A named reference to an image version or variant |
| Digest | `sha256:...` | Content-addressed identifier for a specific manifest |

The registry host and namespace are not always explicitly written in a short image name.

### Docker Hub defaults

For the official Nginx image:

```text
nginx
nginx:1.28
docker.io/library/nginx:1.28
```

`library` is the conventional namespace for Docker Hub's official images. For a user or organization repository, a reference could look like:

```text
docker.io/myteam/my-app:1.0
```

The `docker.io` host identifies Docker Hub. Other registries use their own hostnames and may use different naming structures, for example `ghcr.io/my-org/my-app:1.0` or `registry.example.com:5000/my-app:1.0`.

### Tags and digests

If you omit a tag, Docker generally assumes `:latest`:

```bash
docker pull nginx
# Equivalent shorthand reference: nginx:latest
```

`latest` is a tag name, not a guarantee that the image is the newest version or a production-ready release. Tags can be moved to point to different image content over time.

For repeatable deployments, use a deliberate version tag or pin an image by digest when appropriate:

```bash
docker pull nginx:1.28
docker pull nginx@sha256:<digest>
```

Replace `<digest>` with a real digest. A digest identifies specific content, while a tag is a human-readable reference that can be updated. See [`docker-images-and-containers.md`](docker-images-and-containers.md) for more on image layers, building, and tags.

## Common Registry Services

| Registry | Typical image reference |
|---|---|
| Docker Hub | `docker.io/namespace/image:tag` |
| GitHub Container Registry (GHCR) | `ghcr.io/owner/image:tag` |
| Amazon Elastic Container Registry (Amazon ECR) | `<account-id>.dkr.ecr.<region>.amazonaws.com/repository:tag` |
| Azure Container Registry (ACR) | `<registry-name>.azurecr.io/repository:tag` |
| Google Artifact Registry | `<region>-docker.pkg.dev/<project>/<repository>/<image>:tag` |
| Self-hosted registry | `registry.example.com:5000/image:tag` |

A registry can host public or private repositories depending on the service and its configuration. Cloud registries commonly integrate with their provider's identity and access-management system.

## Docker Hub Pull Rate Limits

Docker Hub applies pull limits based on account tier and authentication. The current Docker documentation lists these limits over a six-hour window:

| User type | Pull limit |
|---|---:|
| Unauthenticated | 100 pulls per IPv4 address or IPv6 `/64` subnet |
| Docker Personal (authenticated) | 200 pulls |
| Docker Pro, Team, and Business (authenticated) | Unlimited, subject to fair-use policies |

Limits and policies can change. Check the official [Docker Hub pull usage and limits documentation](https://docs.docker.com/docker-hub/usage/pulls/) before relying on specific numbers.

Repeated builds, CI jobs, or many hosts pulling the same images can encounter rate limits. Authenticating where appropriate, using an approved registry or pull-through cache, and avoiding unnecessary repeated pulls can help.

## Authentication and Private Registries

A private registry or private repository restricts who can pull or push images. Authentication proves the client's identity; repository permissions determine what that identity is allowed to do.

Log in to Docker Hub:

```bash
docker login
```

Log in to another registry by specifying its hostname, and port if needed:

```bash
docker login ghcr.io
docker login registry.example.com:5000
```

Then pull an image using its full registry reference:

```bash
docker pull registry.example.com:5000/team/my-app:1.0
```

For a private repository, you must have the required permissions. Cloud registries may require a provider-specific login step or short-lived token before Docker commands work.

**Credential safety:**

- Do not put passwords, access tokens, or cloud credentials directly into scripts or source control.
- Prefer the supported credential helper or token-based login method for your environment.
- Avoid passing a password directly as a command-line argument, where it may be exposed in shell history or process information.
- Use least-privilege access: grant only the required pull or push permissions.
- Log out when appropriate on shared systems:

```bash
docker logout
docker logout ghcr.io
```

Docker login credentials are typically stored in Docker's client configuration or through a configured credential helper. Treat that configuration as sensitive.

## Build, Tag, Push, and Pull an Image

The registry path must be part of the tag used for the push. Docker does not infer your intended remote repository just because an image exists locally.

For example, assume you built an image named `my-app:1.0`:

```bash
docker build -t my-app:1.0 .
```

Tag it for a Docker Hub account or organization (replace `myteam` with a namespace where you have push permission):

```bash
docker tag my-app:1.0 myteam/my-app:1.0
```

Authenticate and push:

```bash
docker login
docker push myteam/my-app:1.0
```

Another machine can pull it using the same reference:

```bash
docker pull myteam/my-app:1.0
docker run --rm myteam/my-app:1.0
```

For a private repository, the pulling user or machine must authenticate and have pull permission. The example assumes the image is suitable for running without additional configuration.

Useful commands:

```bash
docker image ls
docker image inspect my-app:1.0
docker image tag my-app:1.0 myteam/my-app:1.0
docker image push myteam/my-app:1.0
docker image pull myteam/my-app:1.0
```

The `docker tag` command creates another local reference to the same image content; it does not rebuild the image. The push uploads image content and associated manifest information needed by the registry.

## Run a Local Registry for Learning

The open-source Distribution Registry can be run as a container. This is useful for learning registry workflows on a development machine or in a controlled lab.

The following example uses `registry:3` and a named volume so registry data remains outside the container's writable layer:

```bash
docker volume create registry-data

docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:3
```

Check the container and its logs:

```bash
docker ps
docker logs registry
```

The registry listens on port `5000` inside the container, and `-p 5000:5000` publishes that port on the host. Port publishing is covered in [`docker-networking.md`](docker-networking.md).

### Push an image to the local registry

Use a small public image for the lab:

```bash
docker pull busybox:1.36
docker image tag busybox:1.36 localhost:5000/busybox:1.36
docker image push localhost:5000/busybox:1.36
```

Confirm that the local registry can provide the image:

```bash
docker image pull localhost:5000/busybox:1.36
docker image ls
```

The image reference `localhost:5000/busybox:1.36` tells Docker to use the registry reachable at port `5000` on the local machine.

### Access the registry from another machine

On another machine, `localhost` means that other machine itself, not the machine hosting the registry. Use the registry host's reachable DNS name or IP address instead, for example:

```bash
docker pull registry-host.example.com:5000/busybox:1.36
```

For this to work, the registry must be reachable from that machine, the network and firewall must allow access, and the Docker client must trust the registry's transport configuration. Do not assume that a registry exposed over plain HTTP will work with a default Docker Engine configuration.

**Security warning:** the simple local example is a lab setup, not a production deployment. It does not configure TLS or authentication. Do not expose an unsecured registry to an untrusted network.

### Clean up the lab

Remove the container:

```bash
docker rm -f registry
```

The named volume remains and retains the registry data. Remove it only if you no longer need the lab's stored images:

```bash
docker volume rm registry-data
```

Named volumes and bind mounts are covered in [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md).

## Production Considerations

A production registry is a service to operate and secure, not just a container listening on port `5000`.

- **TLS:** use HTTPS with a certificate trusted by registry clients. Do not send credentials to a registry over unencrypted HTTP.
- **Authentication and authorization:** restrict access and grant only the permissions needed to pull or push.
- **Persistent storage:** configure reliable storage and understand backup and restore procedures. A container's writable layer is not a suitable registry data-retention plan.
- **Availability and capacity:** consider monitoring, storage capacity, backups, and recovery requirements.
- **Image governance:** control who can publish images, scan images for vulnerabilities, and remove obsolete or unapproved images according to policy.
- **Tagging and deployment:** prefer deliberate version tags; consider digest pinning where repeatability and supply-chain controls require it.
- **Network exposure:** publish only the required ports and restrict access using firewall rules, routing, and network controls.
- **Cloud identity:** for ECR, ACR, or Artifact Registry, use the provider's recommended identity and permission mechanisms.

The Distribution Registry supports configurable storage backends and TLS/authentication. Review the official [registry deployment documentation](https://distribution.github.io/distribution/docs/) and your cloud provider's documentation before deploying a production service. Requirements vary by registry implementation and environment.

## Troubleshooting

| Symptom | Things to check |
|---|---|
| `pull access denied` or `unauthorized` | Image name, repository visibility, login state, and pull permissions |
| `denied: requested access to the resource is denied` during push | Correct namespace/repository, authentication, and push permission |
| `manifest unknown` | Repository name and tag/digest; confirm that the image was pushed |
| `connection refused` | Registry container/service status, listening port, published port, routing, and firewall |
| TLS or certificate error | Registry hostname matches the certificate and the client trusts the issuing CA |
| `http: server gave HTTP response to HTTPS client` | Client is attempting HTTPS against an HTTP-only registry; configure TLS rather than casually disabling transport security |
| Docker Hub `429` response | Check pull usage, authentication, and current Docker Hub limits |

Useful checks for a local registry:

```bash
docker ps --filter name=registry
docker logs --tail 50 registry
docker inspect registry --format '{{json .NetworkSettings.Ports}}'
curl -i http://localhost:5000/v2/
```

For an unauthenticated Distribution Registry, a successful `/v2/` API check commonly returns `200 OK`; exact behavior depends on the registry's authentication and TLS configuration. Use `curl` against HTTP only for this isolated local lab.

## Key Takeaways

- A registry stores and distributes images; Docker Engine pulls images from it and runs containers.
- Image references can include a registry host, namespace, repository, tag, and digest.
- Short names such as `nginx` use Docker's default registry and naming conventions.
- `docker tag` creates a registry-qualified local reference; `docker push` uploads it.
- Pulling a private image requires authentication and permission.
- A local registry is useful for practice, but a remotely accessible registry needs TLS, access control, and reliable storage.
- `localhost` always refers to the machine from which the command is run.
- A tag can move; a digest identifies specific image content.

## Related Notes

- [`docker-basics.md`](docker-basics.md) — common Docker commands and concepts.
- [`docker-images-and-containers.md`](docker-images-and-containers.md) — building images, image references, tags, and container lifecycle.
- [`docker-storage-architecture.md`](docker-storage-architecture.md) — image layers, local image storage, and Docker's data root.
- [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md) — persistent storage for the local registry lab.
- [`docker-networking.md`](docker-networking.md) — port publishing and network reachability.
- [`docker-compose.md`](docker-compose.md) — running multi-container applications and services.
- [`docker-buildkit.md`](docker-buildkit.md) — modern image build tooling and build cache.

## References

- [Docker Hub pull usage and limits](https://docs.docker.com/docker-hub/usage/pulls/)
- [CNCF Distribution Registry — deployment documentation](https://distribution.github.io/distribution/docs/)
- [CNCF Distribution Registry — overview](https://distribution.github.io/distribution/about/)

