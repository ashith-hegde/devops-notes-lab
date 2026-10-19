# Docker Health Checks and Restart Policies

## Overview

Docker health checks and restart policies solve different problems:

- A **health check** periodically tests whether an application inside a container is responding as expected.
- A **restart policy** controls whether Docker restarts a container after its main process exits.

A container can be running but unhealthy. An unhealthy status does not, by itself, cause Docker to restart the container.

## Health Checks

### Health states

Docker reports one of these health states when a health check is configured:

| State | Meaning |
|---|---|
| `starting` | The health check has not yet reached a final healthy or unhealthy state. |
| `healthy` | The configured health-check command has succeeded according to the health-check settings. |
| `unhealthy` | The health check has failed enough times in a row to exceed the configured retry threshold. |

The health state is separate from the container's process state. For example, `running unhealthy` means the container's main process is still running, but the health check is failing.

### Configure a health check with `docker run`

This lab used Nginx with a health check that requests the local web page:

```bash
docker run -d --name ops-healthcheck \
  --health-cmd='wget -q -O /dev/null http://127.0.0.1/ || exit 1' \
  --health-interval=5s \
  --health-timeout=3s \
  --health-retries=2 \
  nginx:alpine
```

Options used:

| Option | Purpose | Lab value |
|---|---|---|
| `--health-cmd` | Command Docker runs to check application health | `wget` requests the local Nginx page; a failed request returns a non-zero status |
| `--health-interval` | Time between checks | `5s` |
| `--health-timeout` | Maximum time allowed for a check | `3s` |
| `--health-retries` | Consecutive failures before Docker marks the container unhealthy | `2` |

The health-check command must reflect what “healthy” means for the application. A TCP connection alone may not prove that the application is serving useful responses. Likewise, a check can be too strict or test the wrong endpoint and produce false alarms.

### Inspect health status

Show the container process state and health state:

```bash
docker inspect --format '{{.State.Status}} {{.State.Health.Status}}' ops-healthcheck
```

Example observed output:

```text
running healthy
```

Inspect recent health-check records, including exit codes and command output:

```bash
docker inspect --format '{{json .State.Health}}' ops-healthcheck
```

To inspect only the recorded check log:

```bash
docker inspect --format '{{json .State.Health.Log}}' ops-healthcheck
```

A successful health-check command normally has `ExitCode: 0`. A non-zero exit code indicates that the check failed; inspect its output to understand why.

## Lab Experiment: Healthy to Unhealthy and Back

### Make the health check fail

The lab temporarily moved the default Nginx index file out of the web root:

```bash
docker exec ops-healthcheck sh -c \
  'mv /usr/share/nginx/html/index.html /usr/share/nginx/html/index.html.lab-backup'
```

After the configured failures, the observed state was:

```text
running unhealthy
```

The health-check output included:

```text
wget: server returned error: HTTP/1.1 403 Forbidden
```

Nginx itself was still running. Because the index file was missing and directory listing was disabled, the request returned HTTP 403. The health-check command treated that unsuccessful HTTP response as a failure.

### Restore the index file

```bash
docker exec ops-healthcheck sh -c \
  'mv /usr/share/nginx/html/index.html.lab-backup /usr/share/nginx/html/index.html'
```

After the next successful checks, the container returned to `running healthy`. This demonstrates that a health state can recover without restarting the container.

> Only run the restore command if the file is currently named `index.html.lab-backup`. Do not run the move commands repeatedly without checking the current filenames.

## Restart Policies

Restart policies determine what Docker does after the container's main process exits. They do not automatically act as a health-based recovery mechanism.

| Policy | Behavior |
|---|---|
| `no` | Do not automatically restart the container. This is the default. |
| `on-failure` | Restart when the container exits with a non-zero exit status. An optional maximum retry count can be specified, for example `on-failure:5`. |
| `always` | Restart whenever the container exits, subject to Docker's manual-stop behavior. |
| `unless-stopped` | Restart after exits unless the container has been explicitly stopped; it remains stopped until manually started again. |

Configure a policy when creating a container:

```bash
docker run -d --restart unless-stopped --name example nginx:alpine
```

Update an existing container's policy:

```bash
docker update --restart unless-stopped ops-healthcheck
```

Inspect the configured policy:

```bash
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' ops-healthcheck
```

Observed output:

```text
unless-stopped
```

### Important distinction: health status versus process exit

The lab deliberately caused the Nginx health check to fail while the Nginx process continued running. With `unless-stopped` configured, the observed state was:

```text
running unhealthy 0
```

The restart count remained `0`. Docker did not restart the container because the health check failed; the main process had not exited.

To monitor or act on unhealthy containers, use an appropriate external monitoring/orchestration mechanism or application-specific recovery process. Do not assume that adding a Docker restart policy will restart a still-running but unhealthy container.

## Lab Experiment: Verify a Process Exit Triggers a Restart

The lab created an Alpine container whose main process exits with status `1` after 15 seconds:

```bash
docker run -d \
  --name restart-test3 \
  --restart on-failure \
  alpine:3.22 \
  sh -c 'sleep 15; exit 1'
```

After waiting, the first inspection showed:

```text
Status=running ExitCode=0 RestartCount=1 Policy=on-failure
```

A later inspection showed:

```text
Status=running ExitCode=0 RestartCount=2 Policy=on-failure
```

The container was running at each inspection because it had restarted and entered another 15-second cycle. The increasing restart count confirmed that Docker restarted it after non-zero exits. `ExitCode=0` in these inspections describes the currently running container state and should not be mistaken for the exit code of a previous run.

### Manual stop and restart-policy behavior

The lab also tested `docker kill --signal=SIGKILL` against a container with `unless-stopped`. It exited with code `137` and did not automatically restart. An explicit Docker stop/kill operation is treated as a manual stop for restart-policy purposes.

To test an unexpected process exit, the lab later terminated the container's main process from the host PID namespace. After that process exited, Docker restarted the container: its host PID changed from `13828` to `19479`, and its restart count increased to `1`.

This is a diagnostic detail, not a recommended routine management technique. Prefer `docker stop`, `docker kill`, and other Docker commands to manage containers; do not directly signal host-side container processes in normal operations.

## PID 1 and Signal Handling in Containers

The process with PID `1` inside a container is the main process in that container's PID namespace. It is not the host's init process, but PID 1 in a PID namespace has special signal-handling behavior. In particular, a signal sent from inside the same PID namespace may not terminate PID 1 if it has no handler for that signal. A process outside that namespace can have different privileges and signal semantics.

The lab's `kill -TERM 1` and `kill -KILL 1` commands, run inside the Alpine container, did not terminate its main shell. The host-side process was then terminated with elevated privileges, after which Docker restarted the container. This distinction explains why “PID 1 can never be killed” is too broad, while “PID 1 behaves like any ordinary process” is also incorrect.

## Useful Troubleshooting Commands

| Command | Purpose |
|---|---|
| `docker ps` | Show running containers and health status when available. |
| `docker ps -a` | Include stopped containers. |
| `docker inspect --format '{{.State.Status}} {{.State.Health.Status}}' CONTAINER` | Show process and health states. |
| `docker inspect --format '{{.RestartCount}}' CONTAINER` | Show the number of Docker-managed restarts. |
| `docker inspect --format '{{.State.ExitCode}}' CONTAINER` | Show the recorded exit code for the current/last container state. |
| `docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' CONTAINER` | Show the configured restart policy. |
| `docker logs CONTAINER` | Read the container's stdout/stderr logs. |
| `docker events --since 5m --filter container=CONTAINER` | Inspect recent lifecycle and health-status events. |

When interpreting results, check several fields together. A health status alone does not tell you whether the main process is running, and a restart count alone does not explain why a restart did or did not occur.

## Key Takeaways

- Health checks report application health; restart policies respond to main-process exits.
- `running unhealthy` is a valid state: the process can run even when the application check fails.
- An unhealthy status alone does not trigger a Docker restart.
- `on-failure` restarts a container after non-zero process exits; `always` and `unless-stopped` have different restart semantics.
- An explicit manual stop/kill and an unexpected application process exit are not equivalent restart-policy tests.
- `RestartCount`, exit status, health-check logs, and lifecycle events should be read together when troubleshooting.

