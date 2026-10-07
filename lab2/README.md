# Lab 2 · Containerize it, then debug it

**Student GitHub:** zaidh2003

- **Image:** `ghcr.io/zaidh2003/course-api:lab2`
- **Platforms:** `linux/amd64`, `linux/arm64`
- **Base image:** `node:24-alpine`
- **Base image digest:** `sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1`

---

## Part 1 · Images and layers

### Base image comparison

| Image | Size | Distro | Default user |
|---|---:|---|---|
| `node:24` | 1.66 GB | Debian 12 (bookworm) | root |
| `node:24-slim` | 332 MB | Debian 12 (bookworm) | root |
| `node:24-alpine` | 304 MB | Alpine Linux 3.24 | root |

### Copy-on-write experiment

- `cow:bad` = **65.4 MB**
- `cow:good` = **12.9 MB**
- Difference = **52.5 MB**

`cow:bad` is larger because Docker image layers are immutable. The large file was created in one layer and deleted in another layer. Deleting it only hides the file; its data still exists in the earlier layer.

In `cow:good`, the large file is created and removed in the same layer, so it is not preserved in the final image layers.

### Build cache experiment

| Build | RUN step CACHED? | Build time |
|---|---|---:|
| b · `app.txt` changed | Yes | ~0.4 s |
| c · `deps.txt` changed | No | 17.519 s |
| d · `app.txt` changed, wrong order | No | 16.901 s |

Changing only `app.txt` did not invalidate the dependency installation layer when the dependency file was copied first.

Changing `deps.txt` invalidated the dependency layer and caused the expensive command to run again.

With the wrong Dockerfile order, changing application code also invalidated the dependency layer.

---

## Part 2 · The course API image

### Image size

| Step | Image | Size |
|---|---|---:|
| Naive | `course-api:naive` | 1.75 GB |
| Final optimized | `course-api:lab2` | 248 MB |
| Reduction against naive | | ~86% |

The final Docker image uses:

- `node:24-alpine`
- Multi-stage build
- Production-only dependencies with `npm ci --omit=dev`
- BuildKit npm cache
- `.dockerignore`
- Non-root `node` user
- Exec-form `CMD`
- `/healthz` endpoint
- Docker `HEALTHCHECK`
- Graceful `SIGTERM` handling
- Pinned Node Alpine image digest
- Multi-platform build

### Container user

```text
uid=1000(node) gid=1000(node) groups=1000(node)
```

The application runs as a non-root user.

### Health check

```text
healthy
```

The API exposes the `/healthz` endpoint and Docker reports the container as healthy.

### Graceful shutdown

The final application contains a `SIGTERM` handler that closes the HTTP server and database pool before exiting.

This allows the application to terminate cleanly instead of waiting for Docker to force-kill the process.

---

## Part 3 · Linux drills

### 3.1 Userspace and kernel

Inside the Ubuntu container:

```text
Ubuntu 24.04.5 LTS (Noble Numbat)
```

Kernel:

```text
6.18.40.1-microsoft-standard
```

`/etc/os-release` describes the userspace distribution inside the container.

`uname -r` shows the kernel used by the Docker/WSL host environment. Containers have their own userspace but share the host kernel.

### 3.2 Text processing

Original file:

```text
chef tech
ansible tech
docker tech
```

After using `sed`:

```text
chef tools
ansible tools
docker tools
```

The command replaced the word `tech` with `tools` in the file.

### 3.3 Logs

Twenty requests were sent to a missing nginx path to generate HTTP 404 responses.

The logs can be filtered using:

```bash
docker logs web 2>/dev/null | grep -c '" 404 '
```

Result:

```text
20
```

This demonstrates how container logs can be filtered to count specific HTTP status codes.

### 3.4 File permissions

File permissions:

```text
-rwxr-x--- 1 root root 7 /lab/f
```

Attempting to read the file as the student user resulted in:

```text
cat: /lab/f: Permission denied
```

The file has mode `750` and is owned by `root:root`.

The owner has read, write, and execute permissions. The group has read and execute permissions. Other users have no permissions.

Because the student user is neither the owner nor a member of the root group, the student cannot read the file.

### 3.5 Environment variables

Before exporting the variable, a child process could not see it.

After:

```bash
export STAGE=staging
```

the child process showed:

```text
child sees: staging
```

A normal shell variable is not automatically inherited by child processes. `export` makes the variable part of the process environment so child processes can access it.

### 3.6 PID 1 and stopping containers

PID 1 inside the lab container was:

```text
bash
```

Stopping the container took:

```text
real 0m4.009s
```

PID 1 has special importance inside a container because Docker sends the termination signal to PID 1.

If PID 1 does not respond correctly to the termination signal, Docker waits for its configured stop timeout before forcing the process to terminate.

### 3.7 Container networking

Docker DNS resolved the nginx container name:

```text
172.18.0.2      web
```

A request from the lab container returned:

```text
HTTP/1.1 200 OK
```

nginx was listening on port 80 inside the container:

```text
0.0.0.0:80


1. cow:bad contains no /big.file, yet it is much bigger than cow:good. Why?
Docker image layers are immutable. In cow:bad, the large file was created in one layer and deleted in a later layer. The deletion only hides the file; the original data remains in the earlier layer. In cow:good, creation and deletion happen in the same layer, so the large file is not preserved in the final image.

2. Why does the order of COPY and RUN lines decide how long a rebuild takes?
Docker reuses cached layers until an instruction or one of its inputs changes. Once a layer is invalidated, subsequent dependent layers must be rebuilt. Copying dependency manifests before application source code allows the expensive dependency installation layer to stay cached when only application code changes.

3. Why can docker stop take several seconds before a container terminates?
Docker first sends a termination signal to the container's PID 1 process and waits for the configured stop timeout. If the process does not handle the signal and exit cleanly, Docker eventually force-kills it. Adding proper SIGTERM handling allows the application to close its server and resources and exit cleanly.

4. Name three things the naive image contained that course-api:lab2 does not.
The optimized image removes or avoids several unnecessary items, including:
1. Development dependencies.
2. Local .env files and secrets.
3. Build/development files such as Dockerfiles, documentation, and other files excluded by .dockerignore.