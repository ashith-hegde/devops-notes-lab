# Docker BuildKit

Notes on the problems with traditional Docker builds and how **BuildKit** addresses them with smarter caching, secret mounts, multi-platform builds, and parallel execution.

---

## Why BuildKit Matters

Traditional Docker builds work well for simple images, but modern development and CI/CD environments introduce requirements that older build workflows handle poorly.

The main problems covered here are:

1. Packages may be downloaded repeatedly during builds.
2. Build credentials can accidentally become part of image metadata or layers.
3. Builds can become tied to a single CPU architecture.
4. Independent build stages may not take advantage of parallel execution.

These problems have a common theme: older build workflows were designed for a simpler environment and did not provide the capabilities needed for modern builds.

---

# Problems with Traditional Docker Builds

## Problem 1 — Packages Re-download Every Build

Consider a Dockerfile that runs package installation:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

or:

```dockerfile
RUN pip install flask
```

The package manager downloads what it needs from the internet during the build.

If the build needs to perform the installation again, the packages may need to be downloaded again. This can become particularly noticeable when:

- the network connection is slow
- dependencies are large
- builds run frequently
- builds run in CI/CD pipelines

Even if only one line of application source code changes, an inefficient build setup can spend unnecessary time retrieving dependencies again.

---

## Problem 2 — Secrets Can Leak into Image Metadata

Builds sometimes need credentials such as:

- API keys
- access tokens
- passwords

A common question is: **How do you make a secret available during the build without accidentally putting it into the resulting image?**

Several approaches can be problematic:

- `--build-arg`
- `ENV`
- `COPY` followed by `rm`
- multi-stage builds used incorrectly

The underlying problem is that build-time secrets need a mechanism that does not persist them in the resulting image or its build history.

---

## Problem 3 — Architecture Lock-in

Container images can be built for different CPU architectures.

Common examples include:

- **AMD64** — common on Intel/AMD systems
- **ARM64** — used by Apple Silicon Macs, AWS Graviton instances, and many ARM systems

A build performed on one architecture can therefore produce an image intended for that architecture.

```text
Intel/AMD development machine
        |
        v
     AMD64 image
        |
        v
ARM64-only production host
        |
        X
```

This becomes a problem when the image architecture does not match the architecture of the system where it needs to run.

---

## Problem 4 — Independent Stages May Run Sequentially

Consider a Dockerfile with two independent build stages:

```dockerfile
FROM node:latest AS frontend
# Build frontend

FROM golang:latest AS backend
# Build backend
```

If the two stages do not depend on each other, their work is conceptually independent.

A modern build system can identify these independent parts and execute them in parallel instead of unnecessarily waiting for one stage to finish before starting the other.

---

# BuildKit

**BuildKit** is Docker's modern build engine.

The command-line frontend commonly used to access advanced BuildKit features is **Buildx**:

```bash
docker buildx build
```

Plain Docker builds already use BuildKit in modern Docker versions. Buildx provides an explicit interface for advanced build functionality.

BuildKit became the default builder in Docker 23.0.

---

# What BuildKit Provides

| Traditional build problem | BuildKit capability |
|---|---|
| Repeated package downloads | Cache mounts |
| Build-time secret exposure | Secret mounts |
| Architecture-specific images | Multi-platform builds |
| Sequential independent stages | Parallel execution |

---

## 1. Smarter Caching

BuildKit improves build caching by tracking the inputs relevant to build steps rather than relying only on simple filesystem metadata.

Two related caching concepts are important:

- **Normal Docker build cache** — reuses results of previously completed Dockerfile steps.
- **Cache mounts** — persist package-manager cache directories between builds so package downloads can be reused even when the installation step itself needs to run again.

The second capability is particularly useful for package managers such as `pip`, `apt`, and similar tools.

---

# Fix 1 — Cache Mounts

Cache mounts address repeated package downloads.

Inside a `RUN` instruction, use:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip     pip install flask flask-mysql
```

Complete example:

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3

RUN --mount=type=cache,target=/root/.cache/pip     pip install flask flask-mysql

COPY app.py /app/

ENTRYPOINT ["python", "/app/app.py"]
```

The cache is kept separately by the builder and can be reused by later builds.

### Why This Is Different from Normal Layer Caching

Suppose:

```dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

Normal layer caching can reuse the `pip install` layer when its inputs have not changed.

But if the `RUN pip install` step has to execute again, the package manager may still need to download packages.

A cache mount gives the package manager its own reusable cache:

```text
Build 1
   |
   +--> Download packages
   |
   +--> Package-manager cache retained

Build 2
   |
   +--> Install step runs again
   |
   +--> Reuse packages from cache
```

This can reduce network usage and improve build times.

---

# Dockerfile Syntax Directive

When using BuildKit-specific Dockerfile syntax such as `RUN --mount`, the Dockerfile can begin with:

```dockerfile
# syntax=docker/dockerfile:1
```

This selects the Dockerfile frontend syntax that supports these features.

For BuildKit features that rely on newer Dockerfile syntax, placing the syntax directive at the top of the Dockerfile makes the intended frontend explicit.

---

# Fix 2 — Secret Mounts

Secret mounts allow a secret to be made available to a single build step without copying it into the image.

Example:

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine

RUN apk add --no-cache curl

RUN --mount=type=secret,id=mykey     curl -H "Authorization: Bearer $(cat /run/secrets/mykey)"     https://api-example.com/private
```

The secret is available inside the build step at:

```text
/run/secrets/mykey
```

### Pass the Secret from Outside

```bash
docker buildx build   --secret id=mykey,src=./key.txt   -t myapp .
```

BuildKit mounts the secret only for the relevant `RUN` instruction.

Conceptually:

```text
Host secret
    |
    v
BuildKit
    |
    v
/run/secrets/mykey
    |
    v
One RUN instruction
    |
    v
Secret unmounted
```

The secret is not intended to become part of the resulting image layer.

### Important Principle

Do not treat:

```dockerfile
ARG API_KEY=...
```

or:

```dockerfile
ENV API_KEY=...
```

as secure ways to provide build secrets.

The purpose of a secret mount is to provide the credential only where it is needed during the build.

---

# Fix 3 — Multi-Platform Builds

BuildKit can build an image for multiple target architectures.

Example:

```bash
docker buildx build   --platform linux/amd64,linux/arm64   -t <image>   --push .
```

For this build:

```text
                  BuildKit
                     |
          +----------+----------+
          |                     |
          v                     v
     linux/amd64           linux/arm64
          |                     |
          +----------+----------+
                     |
                     v
             Container Registry
                     |
                     v
        Multi-architecture image
```

The registry stores the platform-specific images under a single image reference using a **multi-platform manifest**.

When a compatible client pulls the image, the appropriate platform image can be selected.

### Why This Matters

A single image reference can support environments such as:

- AMD64 developer/production systems
- ARM64 developer/production systems
- Apple Silicon development machines
- ARM-based cloud instances

This reduces the need to maintain completely separate image names for each architecture.

---

# Fix 4 — Parallel Build Stages

BuildKit analyzes the dependency relationships between build stages.

If two stages are independent, their work can be performed in parallel.

Example:

```dockerfile
FROM node:latest AS frontend

# Frontend build


FROM golang:latest AS backend

# Backend build
```

Conceptually:

```text
                 BuildKit
                    |
          +---------+---------+
          |                   |
          v                   v
      Frontend              Backend
       build                 build
          |                   |
          +---------+---------+
                    |
                    v
              Final result
```

You do not need to add special Dockerfile instructions just to enable this behavior. BuildKit determines which parts of the build graph are independent and can execute them concurrently.

---

# BuildKit vs Traditional Build

| Area | Traditional approach | BuildKit |
|---|---|---|
| Dockerfile build cache | Basic cache reuse | Smarter build graph and caching |
| Package-manager cache | Not directly mounted into build steps | `type=cache` mounts |
| Build secrets | Easy to accidentally persist | `type=secret` mounts |
| Multiple architectures | Separate build workflows often required | `--platform` multi-platform builds |
| Independent stages | Can be unnecessarily sequential | Can execute in parallel |
| Advanced build interface | `docker build` | `docker buildx build` |

---

# Practical Mental Model

Think of BuildKit as more than just a faster version of `docker build`.

```text
Traditional Docker Build
        |
        +--> Layer cache
        +--> Sequential build mindset
        +--> Limited secret handling
        +--> Architecture-specific builds

BuildKit
        |
        +--> Build graph
        +--> Smarter cache handling
        +--> Cache mounts
        +--> Secret mounts
        +--> Multi-platform builds
        +--> Parallel execution
```

The important distinction is that **normal layer caching and BuildKit cache mounts solve different problems**.

Layer caching can skip an unchanged build step entirely.

Cache mounts help when a build step must run again but its package manager can reuse previously downloaded content.

---

# Key Takeaways

- BuildKit is Docker's modern build engine.
- Buildx provides the command-line interface for advanced BuildKit functionality.
- Modern Docker uses BuildKit by default.
- Cache mounts can reduce repeated package downloads.
- Secret mounts provide build-time credentials without intentionally persisting them in the resulting image.
- Multi-platform builds allow one image reference to support multiple CPU architectures.
- BuildKit can execute independent build stages in parallel.
- `# syntax=docker/dockerfile:1` enables the modern Dockerfile frontend syntax used by features such as `RUN --mount`.
- Normal layer caching and cache mounts are related but solve different problems.
- BuildKit is especially valuable in CI/CD environments where build speed, repeatability, security, and multi-platform support matter.

