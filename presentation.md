# Docker Compose in Depth

Multi-container orchestration, configuration, and best practices

**Tags:** Compose | Services | YAML | Orchestration

**Author:** Brendan Lynskey | 2026

---

## 01 — What is Docker Compose?

A tool for defining and running **multi-container Docker applications** using a single YAML configuration file.

### Without Compose
- Manual `docker run` per container
- Separate network/volume creation
- Complex startup ordering
- Hard to reproduce environments

### With Compose
- Single `docker compose up` command
- Declarative YAML configuration
- Automatic networking between services
- Reproducible across environments

```bash
# One command to rule them all
docker compose up -d
```

---

## 02 — Compose File Structure

A `compose.yaml` file has three top-level keys: **services**, **networks**, and **volumes**.

```yaml
services:          # Container definitions
  web:
    image: nginx:alpine
    ports: ["80:80"]
    networks: [frontend]
    volumes: [static:/usr/share/nginx/html]

  api:
    build: ./api
    networks: [frontend, backend]

  db:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [backend]

networks:          # Custom networks
  frontend:
  backend:
    driver: bridge

volumes:           # Named volumes
  pgdata:
  static:
```

---

## 03 — Service Configuration Deep Dive

Each service supports dozens of configuration keys. Here are the most important ones.

### Container Basics
- `image` — base image
- `build` — build context
- `container_name`
- `command` / `entrypoint`
- `working_dir`

### Networking
- `ports` — host:container
- `expose` — internal only
- `networks`
- `hostname`
- `dns`

### Runtime
- `environment` / `env_file`
- `volumes` / `tmpfs`
- `restart` policy
- `deploy.resources`
- `healthcheck`

```yaml
services:
  app:
    image: node:20-alpine
    container_name: my-app
    working_dir: /app
    command: ["node", "server.js"]
    restart: unless-stopped
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
```

---

## 04 — Build vs Image

Choose between pulling a **pre-built image** or **building from a Dockerfile**.

### Using image:

```yaml
services:
  db:
    image: postgres:16-alpine
    # Pulls from Docker Hub or a private registry
```

- Fast startup — no build step
- Ideal for databases, proxies, caches

### Using build:

```yaml
services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
      target: runtime
      cache_from:
        - myregistry/api:cache
```

- Custom application images
- Supports multi-stage targets

> **Tip:** Combine both — set `build` and `image` together to tag the built image for pushing to a registry.

---

## 05 — Environment Variables & .env Files

Three ways to inject environment variables into services.

### Inline

```yaml
environment:
  - DB_HOST=db
  - DB_PORT=5432
  - DEBUG=false
```

### env_file

```yaml
env_file:
  - .env
  - .env.local
```

Loaded in order; later files override earlier ones.

### Shell / .env

```yaml
# compose.yaml
services:
  db:
    image: postgres:${PG_VER}

# .env file
PG_VER=16-alpine
```

### Priority (highest to lowest)

| Source | Scope | Priority |
|--------|-------|----------|
| CLI `-e` | Single run | 1 (highest) |
| Shell environment | Host process | 2 |
| `environment:` key | Compose file | 3 |
| `env_file:` | External file | 4 |
| `.env` file | Variable substitution | 5 (lowest) |

---

## 06 — depends_on & Healthchecks

Control startup order and wait for services to be truly **ready**, not just started.

### Basic depends_on

```yaml
services:
  api:
    depends_on:
      - db
      - redis
  # Starts db and redis first, but does NOT wait for them to be "ready"
```

### With Healthcheck Condition

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

### Healthcheck Definition

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
```

**Flow:** db starts → healthcheck passes → api starts

---

## 07 — Networking in Compose

Compose creates a **default bridge network** for your project. Services resolve each other by **service name**.

### Default Behaviour
- Auto-creates `<project>_default` network
- All services join it automatically
- DNS resolution by service name
- `api` can reach `db:5432`

### Custom Networks

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # no internet

services:
  proxy:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]
```

| Feature | Default Network | Custom Network |
|---------|----------------|----------------|
| Isolation | All services see each other | Fine-grained segmentation |
| Internet access | Yes | Configurable via `internal` |
| Subnet control | Auto-assigned | `ipam` config supported |

---

## 08 — Volume Management in Compose

Persist data beyond container lifecycle using **named volumes**, **bind mounts**, or **tmpfs**.

### Named Volume

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Managed by Docker. Survives `compose down`.

### Bind Mount

```yaml
services:
  app:
    volumes:
      - ./src:/app/src
      - ./config:/app/config:ro
```

Maps host path directly. Great for development.

### tmpfs

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run:size=64M
```

In-memory filesystem. Lost on restart.

> **Warning:** `docker compose down -v` deletes named volumes. Use with caution in production.

---

## 09 — Compose Profiles

Selectively start services based on **profiles** — perfect for optional tools, debug containers, or test infrastructure.

```yaml
services:
  web:
    image: nginx:alpine       # Always starts (no profile)

  api:
    build: ./api              # Always starts (no profile)

  debug:
    image: busybox
    profiles: [debug]         # Only with --profile debug

  test-db:
    image: postgres:16
    profiles: [testing]       # Only with --profile testing

  mailhog:
    image: mailhog/mailhog
    profiles: [debug, testing] # Starts with either profile
```

```bash
# Start specific profiles
docker compose --profile debug up -d
docker compose --profile testing up -d

# Multiple profiles
docker compose --profile debug --profile testing up -d

# Or use environment variable
COMPOSE_PROFILES=debug,testing
```

---

## 10 — Compose Watch (Hot Reload)

Automatically **sync**, **rebuild**, or **restart** services when source files change.

```yaml
services:
  web:
    build: ./web
    develop:
      watch:
        - action: sync           # Copy changed files into container
          path: ./web/src
          target: /app/src
          ignore:
            - node_modules/

        - action: rebuild        # Full rebuild on Dockerfile changes
          path: ./web/Dockerfile

        - action: sync+restart   # Sync then restart the service
          path: ./web/config
          target: /app/config
```

### Actions

| Action | Behaviour |
|--------|-----------|
| `sync` | Hot-swap files without restart. Best for interpreted languages. |
| `rebuild` | Triggers full image rebuild. Use for dependency or Dockerfile changes. |
| `sync+restart` | Copy files then restart container. For config changes needing a reload. |

```bash
docker compose watch    # Start watching for changes
```

---

## 11 — Multi-Environment Configs

Use **override files** to layer environment-specific configuration on top of a base.

### compose.yaml (base)

```yaml
services:
  api:
    build: ./api
    environment:
      - NODE_ENV=production
    restart: always
```

### compose.override.yaml (dev)

```yaml
services:
  api:
    environment:
      - NODE_ENV=development
      - DEBUG=true
    volumes:
      - ./api/src:/app/src
    ports:
      - "3000:3000"
    restart: "no"
```

Compose **automatically** merges `compose.yaml` + `compose.override.yaml`.

```bash
# Dev (auto-loads override)
docker compose up -d

# Production (explicit files, skips override)
docker compose -f compose.yaml -f compose.prod.yaml up -d

# Staging
docker compose -f compose.yaml -f compose.staging.yaml up -d
```

---

## 12 — Scaling Services

Run multiple instances of a service for **load balancing** and **high availability**.

### CLI Scaling

```bash
# Scale to 3 instances
docker compose up -d --scale api=3

# Scale multiple services
docker compose up -d --scale api=3 --scale worker=5
```

> Cannot use `container_name` or fixed `ports` with scaling.

### Deploy Replicas

```yaml
services:
  api:
    build: ./api
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
    ports:
      - "3000-3002:3000"
```

### Load Balancing Example

```yaml
services:
  proxy:
    image: nginx:alpine
    ports: ["80:80"]
    depends_on: [api]
  api:
    build: ./api
    deploy:
      replicas: 3
    expose: ["3000"]     # Internal only, proxy routes traffic
```

---

## 13 — Compose Commands Cheat Sheet

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start all services in detached mode |
| `docker compose down` | Stop and remove containers, networks |
| `docker compose down -v` | Also remove named volumes |
| `docker compose build` | Build or rebuild service images |
| `docker compose pull` | Pull latest images for services |
| `docker compose logs -f api` | Follow logs for a specific service |
| `docker compose ps` | List running containers |
| `docker compose exec api sh` | Shell into a running container |
| `docker compose run api npm test` | Run a one-off command |
| `docker compose config` | Validate and view merged config |
| `docker compose watch` | Start file-watching for hot reload |
| `docker compose top` | Display running processes |
| `docker compose cp` | Copy files to/from containers |

---

## 14 — Production vs Development Configs

| Concern | Development | Production |
|---------|-------------|------------|
| Build | Use `build:` with watch | Use `image:` from registry |
| Volumes | Bind mounts for live editing | Named volumes only |
| Ports | Expose debug ports | Expose only necessary ports |
| Restart | `restart: "no"` | `restart: always` |
| Logging | Debug/verbose level | Structured JSON logging |
| Resources | Unconstrained | CPU/memory limits set |
| Secrets | Inline env vars | `secrets:` or external vault |

### Development Tips
- Use `compose watch` for hot reload
- Include debug tools (mailhog, pgadmin)
- Mount source code as bind mounts

### Production Tips
- Pin image tags — never use `:latest`
- Set resource limits on every service
- Use healthchecks and restart policies

---

## 15 — Docker Compose vs Kubernetes

Both orchestrate containers, but at very different scales and complexity levels.

| Aspect | Docker Compose | Kubernetes |
|--------|---------------|------------|
| Scope | Single host | Multi-node cluster |
| Config | One YAML file | Multiple manifests (Deployments, Services, Ingress...) |
| Scaling | Manual replicas | Auto-scaling (HPA, VPA) |
| Networking | Bridge networks | Overlay, Service mesh, Ingress |
| Self-healing | Restart policies | Full reconciliation loop |
| Secrets | Environment / files | Native Secrets, external vaults |
| Learning curve | Low | High |
| Best for | Dev, CI/CD, small prod | Large-scale production |

> **Tip:** Use Compose for development and CI, then graduate to Kubernetes when you need multi-node orchestration. Tools like `kompose` help convert Compose files to K8s manifests.

---

## 16 — Compose Extensions & YAML Anchors

Eliminate duplication using **YAML anchors** and Compose **extension fields**.

```yaml
# Extension field (x-) ignored by Compose, used for anchors
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options: { max-size: "10m", max-file: "3" }

x-env: &default-env
  TZ: UTC
  LOG_LEVEL: info

services:
  api:
    <<: *common                 # Merge anchor
    build: ./api
    environment:
      <<: *default-env
      PORT: "3000"

  worker:
    <<: *common                 # Reuse same config
    build: ./worker
    environment:
      <<: *default-env
      QUEUE: jobs
```

| Syntax | Purpose |
|--------|---------|
| `&` (Anchor) | Defines a reusable YAML block. Placed on the source definition. |
| `*` (Alias) | References the anchor to reuse its content. |
| `<<:` (Merge) | Merges keys from the anchor into the current mapping. |

---

## 17 — Summary & Further Reading

### Key Takeaways
- Compose simplifies multi-container apps into one YAML file
- Use **healthchecks** with `depends_on` for proper ordering
- Profiles let you toggle optional services
- Compose Watch enables hot reload in development
- Override files separate dev/staging/prod configs
- YAML anchors eliminate configuration duplication
- Compose is ideal for dev and small prod; Kubernetes for scale

### Further Reading
- [Docker Compose Docs](https://docs.docker.com/compose/)
- [Compose Specification](https://docs.docker.com/compose/compose-file/)
- [Compose Profiles Guide](https://docs.docker.com/compose/profiles/)
- [Compose Watch Reference](https://docs.docker.com/compose/file-watch/)
- [Awesome Compose (examples)](https://github.com/docker/awesome-compose)
- [Compose in Production](https://docs.docker.com/compose/production/)

---

*Brendan Lynskey | 2026*
