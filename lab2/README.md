# Lab 2 – Containerize It, Then Debug It

## Student
GitHub: zaidh2003

## Container Image
`ghcr.io/zaidh2003/course-api:lab2`

The image was built for:
- linux/amd64
- linux/arm64

## Image Optimization

Naive image:
- Image: `course-api:naive`
- Disk usage: 1.75 GB
- Content size: 434 MB

Final optimized image:
- Image: `course-api:final`
- Disk usage: 248 MB
- Content size: 62.5 MB

The optimized image is approximately 86% smaller by disk usage.

## Improvements

The final Docker image uses:
- `node:24-alpine`
- Multi-stage build
- Production-only dependencies with `npm ci --omit=dev`
- BuildKit npm cache
- `.dockerignore`
- Non-root `node` user
- Docker HEALTHCHECK
- `/healthz` endpoint
- Graceful SIGTERM handling
- Pinned Node Alpine image digest
- Multi-platform build

## Base Image Digest

`node:24-alpine`

Digest:

`sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1`

## Verification

Health status:

`healthy`

Container user:

`uid=1000(node) gid=1000(node) groups=1000(node)`

The container runs as a non-root user.

## Build and Push

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/zaidh2003/course-api:lab2 --push .
