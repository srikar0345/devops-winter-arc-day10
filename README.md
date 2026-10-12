# Day 10 — Docker Multi-Stage Builds

## 1. Project Overview

This project demonstrates how Docker multi-stage builds can reduce image size and improve container security for a Go HTTP service.

The application exposes two endpoints:

- `GET /` — returns `Hello from a multi-stage build!`
- `GET /health` — returns `ok`

## 2. Problem Statement

The single-stage Docker image includes the Go development toolchain, source code, and compiled application. The compiler and build tools are not required to run the compiled application.

The objective is to separate the build environment from the runtime environment, reduce the final image size, and run the application as a non-root user.

## 3. Implementation

### Single-stage build

`Dockerfile.single` uses `golang:1.22`, copies the project files, compiles the application, and runs the executable in the same image environment.

### Multi-stage build

`Dockerfile` uses two stages:

1. **Builder:** Uses `golang:1.22-alpine` to compile the Go application into a standalone binary.
2. **Runtime:** Uses `alpine:3.20`, copies only the compiled binary from the builder stage, creates an `app` user, and runs the application without root privileges.

The instruction `COPY --from=builder` transfers the compiled executable into the final image.

## 4. Image Size Comparison

Measurements reported by Docker on my machine:

| Image | Disk Usage | Content Size |
|---|---:|---:|
| `demo:single` | 1.31 GB | 319 MB |
| `demo:build` | 441 MB | 91.3 MB |
| `demo:multi` | 19 MB | 5.72 MB |

Using the displayed disk-usage figures, the multi-stage image is approximately 98.6% smaller than the single-stage image. The result is approximate because the reported sizes are rounded.

Calculation:

`((1310 - 19) / 1310) × 100 ≈ 98.55%`

## 5. Validation Results

- Root endpoint returned: `Hello from a multi-stage build!`
- Health endpoint returned: `ok`
- `docker exec demo id` returned: `uid=100(app) gid=101(app) groups=101(app)`
- The builder stage was built successfully using `docker build --target builder`.
- Image history and image metadata were inspected.

## 6. Key Learnings

- Multi-stage builds separate compilation from runtime execution.
- `COPY --from=builder` copies only the required build artifact.
- Smaller runtime images reduce image transfer and storage requirements.
- Running as a non-root user follows the principle of least privilege.
- `docker history` shows image build instructions and layers.
- `docker image inspect` shows the resulting image configuration.
- `docker build --target builder` builds a specific stage.

## 7. Commands to Reproduce

Build the single-stage image:

```bash
docker build -f Dockerfile.single -t demo:single .
```

Build the multi-stage image:

```bash
docker build -f Dockerfile -t demo:multi .
```

Build only the builder stage:

```bash
docker build --target builder -t demo:build -f Dockerfile .
```

Compare images:

```bash
docker images demo
```

Run the application:

```bash
docker run -d --name demo -p 8080:8080 demo:multi
```

Test endpoints:

```bash
curl http://localhost:8080/
curl http://localhost:8080/health
```

Verify the runtime user:

```bash
docker exec demo id
```

Inspect the image:

```bash
docker history demo:multi
docker image inspect demo:multi
```

## 8. Conclusion

The multi-stage build successfully runs the Go service in a substantially smaller image while keeping the Go build toolchain out of the final runtime stage. The application endpoints work, and the container runs as a non-root user.

The measured results demonstrate the benefits of separating build and runtime environments in Docker.
