# Container Orchestration

Container orchestration automates the deployment, placement, scaling, networking, and recovery of containerized applications across one or more hosts.

Running a single container with `docker run` is straightforward. Operating an application reliably as demand grows, containers fail, or hosts become unavailable is a different problem. Orchestration helps manage that complexity.

These notes introduce the purpose of orchestration and the main concepts behind solutions such as Docker Swarm and Kubernetes. They do not replace the hands-on Docker topics covered in the other notes.

## Why Is Orchestration Needed?

For a small application, you might start a container manually:

```bash
docker run -d --name web nginx
```

As the application grows, several operational questions appear:

- **Scaling:** How do you run additional application instances when demand increases?
- **Health and recovery:** How do you detect an unhealthy container and replace it?
- **Host failure:** What happens if the host running the container becomes unavailable?
- **Placement:** Which host has enough resources to run a new container?
- **Networking and discovery:** How do services find and communicate with instances running on different hosts?
- **Updates:** How can you deploy a new application version while limiting disruption?
- **Configuration and secrets:** How do you manage environment-specific configuration and sensitive values consistently?
- **Operations at scale:** How can you manage many workloads without manually running commands on every host?

Manually running more `docker run` commands can work for small labs, but it does not provide a complete system for scheduling, recovery, or coordinated updates across hosts.

Container orchestration addresses these problems by allowing an operator to declare the desired state of an application and letting the orchestration platform work to maintain that state.

## What Does an Orchestrator Do?

Depending on the platform and configuration, an orchestrator can provide:

- **Scheduling:** Places workloads on suitable nodes based on resource requirements, constraints, and available capacity.
- **Desired-state management:** Tracks how many instances should be running and attempts to reconcile actual state with desired state.
- **Self-healing:** Replaces or restarts workloads when supported health and failure conditions indicate that action is needed.
- **Scaling:** Changes the number of application instances manually or automatically, depending on the platform and configured metrics or policies.
- **Service discovery and load balancing:** Helps clients reach the application without needing to know every individual container address.
- **Rolling updates and rollbacks:** Coordinates application updates to reduce disruption and can support returning to a previous version.
- **Multi-host coordination:** Manages workloads across multiple machines and can help maintain availability when a node fails.
- **Configuration and secrets management:** Provides mechanisms for distributing configuration and sensitive values, with details varying by platform.
- **Resource controls:** Uses CPU and memory requirements, limits, and scheduling policies to manage resource use.

Orchestration does not make an application automatically highly available. The application, data layer, network, storage, replica placement, health checks, and control plane must also be designed appropriately.

## Desired State and Reconciliation

A central orchestration concept is **desired state**.

For example, an operator may request three replicas of a web application. The orchestrator compares the requested state with the current state and takes action when they differ.

```text
Desired state: 3 application replicas
                 |
                 v
       Orchestrator observes state
                 |
        +--------+--------+
        |                 |
   3 replicas running   Only 2 running
        |                 |
   No action needed    Create/replace a replica
```

This is a simplified model. The exact behavior depends on the orchestrator, workload configuration, and type of failure. A controller may replace a failed container, while a node failure may require the scheduler to place a replacement on another eligible node.

## Docker Swarm

Docker Swarm is Docker Engine's native clustering and orchestration mode. It groups Docker hosts into a swarm and manages services across its nodes.

Swarm distinguishes between:

- **Manager nodes:** Maintain the swarm's desired state, coordinate scheduling, and manage cluster operations. A manager can also run workloads unless configured otherwise.
- **Worker nodes:** Run assigned workloads and report status to the swarm managers.

Swarm services describe the desired state of an application, including the image to run and the number of replicas.

### Example: Create a replicated service

The following command must be run on a Swarm manager after a swarm has been initialized and the required nodes have joined. On a single-node lab, initialize the swarm first:

```bash
docker swarm init
```

Create a service:

```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish published=8080,target=80 \
  nginx:alpine
```

Inspect the service and its tasks:

```bash
docker service ls
docker service ps web
docker service inspect web
```

Change the desired replica count:

```bash
docker service scale web=5
```

Remove the service:

```bash
docker service rm web
```

The `--replicas 3` setting asks Swarm to maintain three service tasks. The actual placement depends on available nodes and constraints. If only one node exists, all replicas run on that node; this does not provide host-level high availability.

The command:

```bash
docker service create --replicas 100 nodejs
```

is a Swarm service command, not a generic `docker run` command. `nodejs` must refer to a valid image reference available to the Swarm nodes. Creating 100 replicas also does not guarantee the cluster has enough resources to run them successfully.

## Kubernetes

Kubernetes is a widely adopted open-source container orchestration platform. It originated at Google and is now developed as a community project under the Cloud Native Computing Foundation (CNCF).

Kubernetes provides an extensible API and control plane for declaring and managing workloads across a cluster. It is commonly used for production applications and is supported by managed services from major cloud providers, including Amazon EKS, Azure Kubernetes Service (AKS), and Google Kubernetes Engine (GKE).

Common Kubernetes concepts include:

- **Cluster:** The overall Kubernetes environment.
- **Control plane:** Components that expose the API, make scheduling decisions, and maintain cluster state.
- **Node:** A machine that runs application workloads.
- **Pod:** The smallest deployable unit in Kubernetes, containing one or more closely related containers.
- **Deployment:** Manages replicated, stateless application Pods and supports rolling updates.
- **Service:** Provides a stable network endpoint for a set of Pods.
- **ConfigMap and Secret:** Provide mechanisms for supplying configuration and sensitive data to workloads.
- **Horizontal Pod Autoscaler (HPA):** Adjusts the number of Pod replicas based on configured metrics when the necessary metrics and configuration are available.

Kubernetes is highly extensible and supports a large ecosystem of networking, storage, security, observability, and deployment integrations. That flexibility also brings more concepts and operational complexity than a small Docker-only setup.

Kubernetes does not automatically scale every part of an application. For example, the HPA scales workload replicas; scaling the underlying cluster nodes is a separate capability that requires suitable infrastructure and configuration.

## Docker Swarm vs. Kubernetes

Both can orchestrate containerized applications, but they differ in ecosystem, flexibility, and operational complexity.

| Area | Docker Swarm | Kubernetes |
|---|---|---|
| Setup and learning | Generally simpler for Docker users | More concepts and configuration to learn |
| Integration | Closely integrated with Docker Engine | Broad cloud-native ecosystem and integrations |
| Workload model | Swarm services and tasks | Pods, Deployments, Services, and other API resources |
| Scaling | Manual service scaling; automation can be added externally | Manual scaling and built-in autoscaling mechanisms such as HPA when configured |
| Networking | Swarm networking and service discovery | Cluster networking model, Services, and optional ingress/gateway components |
| Ecosystem | Smaller ecosystem | Broad ecosystem and widespread managed-cloud support |
| Typical fit | Simpler clustered Docker workloads and learning | A broad range of container platforms, including complex production environments |

It is too broad to say that Swarm is never suitable for production. Suitability depends on the application's requirements, team expertise, support needs, and operational environment. Kubernetes is more widely adopted in modern cloud-native ecosystems, but its additional capabilities also introduce complexity.

Other container platforms and managed services exist too, including Amazon ECS and HashiCorp Nomad. Docker Swarm and Kubernetes are useful examples, not the only available options.

## Where Docker Compose Fits

Docker Compose defines and runs multi-container applications using a YAML file. It is useful for local development, testing, and many single-host deployments.

Compose is **not a substitute for a multi-node orchestrator** when you need cluster-wide scheduling, node-failure recovery, and coordinated placement across a fleet of hosts.

For example:

```bash
docker compose up -d
```

starts the services described in the Compose file. It does not, by itself, create a cluster of Docker hosts or provide Kubernetes-style scheduling across nodes.

See [`docker-compose.md`](docker-compose.md) for Compose files, service-name networking, application lifecycle, and examples.

## How Orchestration Relates to Other Docker Concepts

- **Images and containers:** Orchestrators run containers from images. See [`docker-images-and-containers.md`](docker-images-and-containers.md).
- **Networking:** Multi-container and multi-host workloads need service discovery, connectivity, and appropriate exposure controls. See [`docker-networking.md`](docker-networking.md).
- **Volumes and storage:** Stateful workloads need deliberate storage and recovery design. See [`docker-volumes-and-bind-mounts.md`](docker-volumes-and-bind-mounts.md) and [`docker-storage-architecture.md`](docker-storage-architecture.md).
- **Registries:** Cluster nodes must be able to pull the images required by their workloads, with suitable access and authentication. See [`docker-registry.md`](docker-registry.md).
- **Resource isolation:** Linux namespaces and cgroups underpin important aspects of container isolation and resource control. See [`docker-linux-namespaces-and-cgroups.md`](docker-linux-namespaces-and-cgroups.md).

## Key Takeaways

- Running containers manually is useful for small workloads, but becomes difficult to manage as the number of instances and hosts increases.
- Orchestration manages desired state, scheduling, scaling, recovery, service discovery, and coordinated updates.
- Docker Swarm is Docker's native clustering and orchestration mode; `docker service` commands operate on Swarm services.
- Kubernetes is a widely adopted, extensible orchestration platform with a broad cloud-native ecosystem.
- Docker Compose manages multi-container applications but does not provide full multi-host cluster orchestration.
- More replicas on one host do not protect against failure of that host.
- Orchestration does not eliminate the need for application health checks, secure configuration, observability, capacity planning, and reliable data storage.

## Further Learning

- [Docker Swarm mode overview](https://docs.docker.com/engine/swarm/)
- [Docker services](https://docs.docker.com/engine/swarm/services/)
- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Azure Kubernetes Service](https://learn.microsoft.com/azure/aks/what-is-aks)
- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs)

