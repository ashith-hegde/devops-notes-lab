# Docker Storage Architecture

These notes cover Docker's storage layout, image layers, copy-on-write, storage drivers, and the containerd image store.

For hands-on named-volume and bind-mount examples, see [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md). For introductory image-layer and container-lifecycle concepts, see [`docker-images-and-containers.md`](docker-images-and-containers.md).

## Docker's Data Directory

On Linux, Docker traditionally stores data under `/var/lib/docker`. The exact layout depends on the Engine version and storage backend.

A simplified layout for a classic storage backend is:

```text
/var/lib/docker/
├── containers/   # Container metadata and runtime files
├── image/        # Image metadata
├── volumes/      # Docker-managed volume data
└── overlay2/     # Filesystem layers when using the classic overlay2 driver
```

This is an illustration, not a guaranteed layout on every installation.

Docker Engine 29.0 and later uses the containerd image store by default on fresh installations. In this configuration, image content and container snapshots are stored under `/var/lib/containerd`, while other Docker data, including volumes, remains under `/var/lib/docker` by default.

Important points:

- `/var/lib/docker` is the traditional default data root, not a guaranteed location for every type of Docker data.
- The daemon's `data-root` setting changes Docker's data directory, but does not relocate image and snapshot data stored by the containerd image store.
- Never manually edit or delete files inside Docker's internal storage directories.

Useful commands:

```bash
docker info
docker info --format '{{.DockerRootDir}}'
docker info --format '{{.Driver}}'
docker system df
```

The fields shown by `docker info` depend on the Engine version and backend.

## Image Layers

Docker images are built from filesystem layers. Layers allow image content to be reused across images and containers.

Example Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Conceptually, the image contains the base image, dependency-installation changes, and application files. This is a simplified breakdown: not every Dockerfile instruction creates a filesystem layer. Instructions such as `RUN`, `COPY`, and `ADD` commonly create filesystem layers, while `WORKDIR` and `CMD` do not each create their own filesystem layer.

### Why layers are useful

- **Reuse:** Images can share common image content.
- **Build caching:** Docker can reuse eligible cached results when instructions and relevant inputs have not changed.
- **Distribution:** Images can be transferred as a collection of layers rather than one monolithic filesystem.

Layer sharing and build caching are related but different. Sharing avoids duplicate image content where possible; build caching avoids repeating build work when a previous result can be reused.

Separating dependency installation from application source copying can allow the dependency step to remain cached when only application code changes.

Inspect image history:

```bash
docker history <image-name>
```

This shows recorded layer history and command information; it is not a complete filesystem diff or a precise measure of each layer's unique disk usage.

## The Container Writable Layer

Image layers are read-only. When Docker creates a container, it adds a container-specific writable layer on top of the image filesystem.

```text
Container filesystem view
┌──────────────────────────────┐
│ Container writable layer     │  Runtime changes
├──────────────────────────────┤
│ Read-only image layer        │
├──────────────────────────────┤
│ Read-only image layer        │
├──────────────────────────────┤
│ Base image layers            │
└──────────────────────────────┘
```

Each container has its own writable layer, while containers created from the same image can share underlying read-only image content.

Files written only to the container's writable layer are temporary with respect to the container lifecycle:

- `docker stop` preserves the container and its writable layer.
- `docker start` starts that same container with its existing writable-layer data.
- Removing the container removes its writable layer.
- A replacement container created from the same image does not restore files that existed only in the removed container's writable layer.

For data that must survive container replacement, use a named volume or bind mount. See [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md).

## Copy-on-Write

With layered filesystems such as OverlayFS, Docker can use a copy-on-write approach to make read-only image content appear as part of a container's writable filesystem.

At a high level:

- **Read an unchanged file:** The container can read it from an underlying read-only layer.
- **Create a new file:** The new file is written to the writable layer.
- **Modify an existing file:** The filesystem may copy the file into the writable layer before applying the modification. This is often called *copy-up*.
- **Delete a file from an image layer:** The underlying read-only layer remains unchanged; the writable layer records the deletion using filesystem-specific mechanisms.

Copy-on-write does not mean Docker duplicates the entire image whenever a container changes a file. The exact implementation depends on the storage backend and filesystem.

Because writes to the writable layer require filesystem-layer management, volumes are generally preferred for persistent application data and can avoid some writable-layer overhead.

## Storage Drivers and Snapshotters

The storage backend manages how image content and container filesystems are stored. Terminology depends on which backend the Engine uses.

### Classic storage drivers

Older and upgraded Docker Engine installations may use classic storage drivers. `overlay2` is a well-known example on Linux.

Other drivers have included `btrfs`, `zfs`, `vfs`, and `fuse-overlayfs` for applicable environments. Availability depends on the operating system, kernel, Docker version, and installation mode.

With the classic `overlay2` driver, OverlayFS combines read-only lower layers with a container's writable upper layer to present a unified filesystem view.

### The containerd image store

Docker Engine 29.0 and later uses the containerd image store by default on fresh installations. Upgraded installations may continue using the classic storage backend unless configured otherwise.

The containerd image store uses **snapshotters** rather than classic Docker storage drivers to manage image and container filesystem snapshots. The default snapshotter is commonly `overlayfs`, but other snapshotters can provide different capabilities.

| Classic storage backend | Containerd image store |
|---|---|
| Uses classic storage drivers, such as `overlay2` | Uses containerd snapshotters |
| Commonly stores image and container filesystem data under `/var/lib/docker` | Stores image content and container snapshots under `/var/lib/containerd` by default |
| Has limitations with locally stored multi-platform image indexes and attestations | Supports local multi-platform image storage and image attestations |
| Uses the classic driver's filesystem-layer implementation | Snapshotter-dependent filesystem behavior |

Both approaches rely on operating-system filesystem and kernel facilities. The containerd image store is a different storage architecture, not simply another name for the `overlay2` driver.

### Why the containerd image store matters

Depending on Engine version, image format, and configuration, it enables capabilities such as:

- Storing multi-platform images locally, including AMD64 and ARM64 variants.
- Working with image attestations, such as provenance and software bill of materials (SBOM) metadata.
- Supporting Wasm-related container workflows.
- Using alternative snapshotters that can support lazy pulling or peer-to-peer image distribution.

These capabilities are not all automatically enabled by the image store alone. Some require a compatible snapshotter, runtime, image, builder, or additional configuration.

The containerd image store may use more disk space than the classic backend because it can retain both compressed image content and unpacked filesystem data.

## Persistent Data: Volumes vs. the Writable Layer

The container writable layer is not a substitute for persistent application storage.

| Storage mechanism | Managed by | Survives container removal? | Typical use |
|---|---|---:|---|
| Container writable layer | Docker's storage backend | No | Temporary runtime changes |
| Named volume | Docker | Yes, unless the volume is removed | Database and application data |
| Bind mount | Host filesystem / operator | Yes, as long as host data remains | Source code, configuration, host-managed files |
| `tmpfs` mount | Host memory | No; temporary | Transient data that should not persist to disk |

For mount syntax and hands-on labs, see [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md).

## Useful Inspection and Maintenance Commands

```bash
# Show Docker's storage backend and data-root
docker info

# Show Docker disk usage
docker system df

# Show detailed disk usage, including more build-cache detail
docker system df -v

# Inspect image history
docker history <image-name>

# Inspect a container's configured mounts
docker inspect <container-name> --format '{{json .Mounts}}'
```

Use Docker's own commands to inspect and manage its data. Avoid manually deleting files under `/var/lib/docker` or `/var/lib/containerd`; doing so can corrupt local image or container state.

## Key Takeaways

- Docker stores image content, container filesystems, metadata, and volumes using backend-specific layouts.
- `/var/lib/docker` is the traditional default data root, but the containerd image store places image content and container snapshots under `/var/lib/containerd` by default.
- Image layers are read-only and can be shared; each container has its own writable layer.
- Copy-on-write lets a container modify its filesystem without modifying the original image layers.
- Data stored only in the container writable layer is removed with the container.
- Named volumes and bind mounts are the usual choices for data that must outlive a container.
- Classic storage drivers such as `overlay2` and containerd snapshotters are related to layered filesystems but are not interchangeable terms.
- Never manually modify Docker's internal storage directories.

## Official References

- [Docker storage overview](https://docs.docker.com/engine/storage/)
- [Storage drivers](https://docs.docker.com/engine/storage/drivers/)
- [Containerd image store](https://docs.docker.com/engine/storage/containerd/)
- [Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker daemon configuration](https://docs.docker.com/engine/daemon/)

