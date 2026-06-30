# Kamal 2 — Build Strategies & Builders

## Table of Contents

1. [How Kamal builds images](#how-kamal-builds-images)
2. [Builder drivers](#builder-drivers)
3. [Architecture and multiarch](#architecture-and-multiarch)
4. [Remote builder (cross-arch via SSH)](#remote-builder-cross-arch-via-ssh)
5. [Build context and Dockerfile](#build-context-and-dockerfile)
6. [Build args and secrets](#build-args-and-secrets)
7. [Build caching](#build-caching)
8. [Cloud Native Buildpacks (no Dockerfile)](#cloud-native-buildpacks-no-dockerfile)
9. [Asset bridging](#asset-bridging)
10. [Image tagging and versioning](#image-tagging-and-versioning)

---

## How Kamal builds images

By default, Kamal builds from a **clean temporary clone of the current git HEAD** (not your working directory). This ensures reproducible builds and prevents local uncommitted changes from sneaking into production.

```bash
kamal build push         # build and push to registry
kamal build deliver      # build, push, then pull on servers
kamal build dev          # build from working dir (dirty), push to local Docker store only
```

---

## Builder drivers

### `docker-container` (default)

Uses Docker Buildx with the `docker-container` driver. Supports:
- Multi-architecture builds (amd64 + arm64)
- Remote builder over SSH
- Layer caching to registry
- Build secrets

```yaml
builder:
  # no 'driver' needed — docker-container is the default
```

### `docker` (simple, local-only)

Uses the local Docker daemon directly. Faster for simple cases but does NOT support:
- Multi-architecture builds
- Remote builders
- Registry caching

```yaml
builder:
  driver: docker
  arch: amd64    # single arch only
```

---

## Architecture and multiarch

### Single architecture (most common)

```yaml
builder:
  arch:
    - amd64     # your servers run on amd64 (Intel/AMD)
```

or simply:
```yaml
builder:
  arch: amd64
```

### Multiple architectures

Build for both `amd64` (Intel/AMD servers) and `arm64` (AWS Graviton, ARM servers):

```yaml
builder:
  arch:
    - amd64
    - arm64
```

The default `docker-container` driver handles multiarch via QEMU emulation (slow) or native remote builders (fast).

---

## Remote builder (cross-arch via SSH)

**When you need this:** Developing on Apple Silicon (arm64) but deploying to amd64 servers. Cross-compilation via QEMU is painfully slow; a remote Linux/amd64 builder is much faster.

```yaml
builder:
  remote: ssh://user@builder.example.com       # amd64 Linux builder
  arch:
    - amd64
```

### With both arches (hybrid)

Build arm64 locally, amd64 remotely:

```yaml
builder:
  remote: ssh://user@builder.example.com
  # local: true is the default — arm64 built locally, amd64 on remote
  arch:
    - amd64
    - arm64
```

Disable local building entirely (always use remote):
```yaml
builder:
  remote: ssh://user@builder.example.com
  local: false
  arch:
    - amd64
```

Custom port:
```yaml
builder:
  remote: ssh://user@builder.example.com:2222
```

### Remote builder prerequisites

- A Linux server (any cloud VM works)
- Docker installed on the remote server
- Passwordless SSH from your local machine to the builder
- The Docker daemon accessible over SSH (`ssh user@builder docker info` should work)
- Registry credentials available on the remote (Kamal handles this via `kamal registry login`)

Images are pushed directly from the remote builder to the registry — they never transit your local machine.

---

## Build context and Dockerfile

```yaml
builder:
  context: .                      # use local working directory instead of git clone
  dockerfile: Dockerfile.production   # default: Dockerfile
  target: production              # multi-stage build target
```

### Dockerfile expectations

- Your `Dockerfile` must `EXPOSE` the port your app listens on (or set `proxy.app_port` in config)
- Use multi-stage builds to minimize final image size
- The `WORKDIR` should match where your app runs from

Generic multi-stage example:
```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Final stage
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Build args and secrets

### Build args (visible in image layers — safe for non-sensitive values)

```yaml
builder:
  args:
    NODE_VERSION: "20"
    BUILD_ENV: production
    APP_VERSION: <%= `git describe --tags` %>    # ERB supported
```

In Dockerfile:
```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
```

### Build secrets (mounted at build time, NOT baked into layers)

For sensitive tokens (GitHub, NPM, etc.) needed during the build:

```yaml
builder:
  secrets:
    - GITHUB_TOKEN
    - NPM_TOKEN
```

Values come from `.kamal/secrets`. In Dockerfile:
```dockerfile
RUN --mount=type=secret,id=GITHUB_TOKEN \
    GITHUB_TOKEN=$(cat /run/secrets/GITHUB_TOKEN) \
    npm install --prefer-offline

RUN --mount=type=secret,id=NPM_TOKEN \
    npm config set //registry.npmjs.org/:_authToken $(cat /run/secrets/NPM_TOKEN) && \
    npm install && \
    npm config delete //registry.npmjs.org/:_authToken
```

---

## Build caching

Layer caching significantly speeds up builds by reusing unchanged layers.

### Registry cache (recommended)

```yaml
builder:
  cache:
    type: registry
    options: mode=max        # max = cache all layers, not just final stage
    image: myapp-build-cache  # optional: custom cache image name (default: auto-named)
```

### GitHub Actions / CI cache

```yaml
builder:
  cache:
    type: gha           # GitHub Actions cache
    options: mode=max
```

### Local cache

```yaml
builder:
  cache:
    type: local
    options: mode=max
```

---

## Cloud Native Buildpacks (no Dockerfile)

If you don't have a Dockerfile, Kamal can use Buildpacks to auto-detect and build your app:

```yaml
builder:
  pack:
    builder: heroku/builder:24
    buildpacks:
      - heroku/nodejs
      # - heroku/python
      # - heroku/go
```

Requires `pack` CLI installed locally.

---

## Asset bridging

### The problem

During rolling deploys, old containers are still serving some requests while new containers serve others. If your frontend assets are versioned by hash (`app.abc123.js`, `app.def456.js`), a user who loaded an old HTML page with `app.abc123.js` will get 404s after the new container replaces the old one (which had `app.abc123.js`).

### The solution: `asset_path`

```yaml
asset_path: /app/public      # path inside the container where static assets live
```

What Kamal does:
1. Before deploying a new container, extract the new container's assets to the host
2. Mount the host's combined asset directory into both old and new containers
3. Old container serves both old and new assets
4. New container also serves both
5. Users on old HTML pages never hit 404s

### Per-role asset_path

```yaml
asset_path: /app/public       # global default

servers:
  web:
    hosts: [192.168.0.1]
    # inherits global asset_path
  admin:
    hosts: [192.168.0.2]
    asset_path: /app/admin/public    # override for this role
```

### Read-only option

```yaml
asset_path: /app/public:ro    # mount as read-only
```

---

## Image tagging and versioning

Images are automatically tagged with the **git commit SHA** (first 7 characters).

```bash
# Default: uses git commit SHA
kamal deploy                       # tags image as e.g. myapp:abc123d

# Override version
VERSION=v1.2.3 kamal deploy       # tags as myapp:v1.2.3

# Uncommitted changes add -dirty suffix (from kamal build dev)
kamal build dev                    # tags as myapp:abc123d-dirty
```

Inspect available versions:
```bash
kamal app containers    # lists version tags on all hosts
kamal app images        # lists images on all hosts
```
