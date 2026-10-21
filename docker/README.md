# Docker Development Guide

This guide provides instructions for building and running the LiteLLM application using Docker and Docker Compose.

## Prerequisites

- Docker
- Docker Compose

## Building and Running the Application

The root `docker-compose.yml` starts a pre-incident image pinned by digest. Use `compose.source.yml` to build checked-out source, or `docker-compose.hardened.yml` for the non-root hardening/QA build.

### 1. Set the Master Key

The application requires a `LITELLM_MASTER_KEY` for signing and validating tokens. You must set this key as an environment variable before running the application.

Create a `.env` file in the root of the project and add the following line:

```
LITELLM_MASTER_KEY=your-secret-key
```

Replace `your-secret-key` with a strong, randomly generated secret.

### 2. Run the Pinned Image or Build from Source

The default stack uses the pre-incident LiteLLM v1.106.2 image pinned by digest.
Once you have set `LITELLM_MASTER_KEY`, start it with:

```bash
docker compose up -d
```

This starts the `litellm`, `litellm_db`, and `prometheus` services in detached mode.
To build the checked-out source with the root Dockerfile instead, use the explicit override:

```bash
docker compose -f docker-compose.yml -f compose.source.yml up -d --build
```

The source override uses a local image tag and always builds it. Source-build validation
remains outstanding: CI currently reports Python 3.15 is incompatible with the project's
`>=3.10,<3.14` requirement. The pinned image path does not use that source build.
The hardened stack below also keeps its own local image tag and build configuration.

### 3. Verifying the Application is Running

You can check the status of the running containers with the following command:

```bash
docker compose ps
```

To view the logs of the `litellm` container, run:

```bash
docker compose logs -f litellm
```

### 4. Stopping the Application

To stop the running containers, use the following command:

```bash
docker compose down
```

## Hardened / Offline Testing

To ensure changes are safe for non-root, read-only root filesystems and restricted egress, always validate with the hardened compose file:

```bash
docker compose -f docker-compose.yml -f docker-compose.hardened.yml build --no-cache
docker compose -f docker-compose.yml -f docker-compose.hardened.yml up -d
```

This setup:
- Builds from `docker/Dockerfile.non_root` with Prisma engines and Node toolchain baked into the image.
- Runs the proxy as a non-root user with a read-only rootfs and only writable tmpfs mounts:
  - `/app/cache` (Prisma/NPM cache; backing `PRISMA_BINARY_CACHE_DIR`, `NPM_CONFIG_CACHE`, `XDG_CACHE_HOME`)
  - `/app/migrations` (Prisma migration workspace; backing `LITELLM_MIGRATION_DIR`)
- Pre-builds and serves the admin UI from read-only paths:
  - `/var/lib/litellm/ui` (pre-restructured Next.js UI with `.litellm_ui_ready` marker)
  - `/var/lib/litellm/assets` (UI logos and assets)
- Routes all outbound traffic through a local Squid proxy that denies egress, so Prisma migrations must use the cached CLI and engines.

You should also verify offline Prisma behaviour with:

```bash
docker run --rm --network none --entrypoint prisma litellm-hardened:local --version
```

This command should succeed (showing engine versions) even with `--network none`, confirming that Prisma binaries are available without network access.

## Troubleshooting

-   **`build_admin_ui.sh: not found`**: This error can occur if the Docker build context is not set correctly. Ensure that you are running the `docker-compose` command from the root of the project.
-   **`Master key is not initialized`**: This error means the `LITELLM_MASTER_KEY` environment variable is not set. Make sure you have created a `.env` file in the project root with the `LITELLM_MASTER_KEY` defined.
