# 26 — Docker for Java/Spring Developers

## 1. Why containers?
- "Works on my machine" → package app + runtime + OS libs into an **image** that runs the same everywhere.
- Lightweight isolation (namespaces + cgroups) sharing the host kernel — starts in seconds, far less overhead than VMs.
- Standard unit for CI/CD and orchestration (Kubernetes).

| | Virtual Machine | Container |
|--|--|--|
| Isolation | Full guest OS + hypervisor | Process-level (namespaces, cgroups) |
| Size | GBs | MBs |
| Startup | Minutes | Seconds |
| Density | Low | High |
| Security boundary | Stronger | Weaker (shared kernel) |

## 2. Core concepts
| Term | Meaning |
|------|---------|
| **Image** | Read-only template made of **layers** (each Dockerfile instruction adds a layer); identified by `repo:tag` / digest |
| **Container** | Running (or stopped) instance of an image with a thin writable layer |
| **Dockerfile** | Recipe to build an image |
| **Registry** | Stores images: Docker Hub, GHCR, AWS ECR, GCP Artifact Registry, Azure ACR, Harbor |
| **Volume** | Persistent data managed by Docker, survives container removal |
| **Bind mount** | Host directory mounted into container |
| **Network** | Bridge (default), host, none, overlay; user-defined bridge networks give DNS by container name |
| **Docker Engine** | `dockerd` daemon + containerd + runc; CLI talks to daemon |
| **OCI** | Open Container Initiative — standard image/runtime format (Podman, containerd compatible) |

## 3. Essential commands
```bash
# images
docker pull eclipse-temurin:21-jre
docker images                         # list
docker build -t acme/orders:1.0.0 .
docker tag acme/orders:1.0.0 registry.example.com/acme/orders:1.0.0
docker push registry.example.com/acme/orders:1.0.0
docker rmi <image>
docker image history acme/orders:1.0.0    # see layers & sizes
docker image prune -a

# containers
docker run -d --name orders -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod -e DB_PASSWORD=secret \
  --memory=1g --cpus=1.5 \
  --restart unless-stopped \
  acme/orders:1.0.0
docker ps / docker ps -a
docker logs -f --tail 100 orders
docker exec -it orders sh             # shell inside
docker stop orders && docker rm orders
docker inspect orders                 # JSON details (IP, mounts, env)
docker stats                          # live CPU/memory
docker cp orders:/app/logs/app.log .
docker system df / docker system prune

# volumes & networks
docker volume create pgdata
docker run -d --name pg -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw postgres:16
docker network create shop-net
docker run --network shop-net --name orders ...      # reach postgres at hostname "pg"
```
`-p host:container` publishes ports; `EXPOSE` in Dockerfile is documentation only.

## 4. Dockerfile instructions
| Instruction | Purpose |
|------------|---------|
| `FROM image AS name` | Base image (multi-stage name) |
| `WORKDIR /app` | Working directory |
| `COPY src dest` | Copy files from build context (prefer over ADD) |
| `ADD` | Like COPY + URLs + auto-extract tar |
| `RUN cmd` | Execute at build time (creates a layer) |
| `ENV KEY=value` | Environment variable (build & runtime) |
| `ARG NAME=default` | Build-time variable only |
| `EXPOSE 8080` | Document port |
| `USER app` | Run as non-root |
| `ENTRYPOINT ["java","-jar","app.jar"]` | Main executable (exec form → receives signals) |
| `CMD ["--spring.profiles.active=prod"]` | Default args (overridable) |
| `HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1` | Container health |
| `LABEL` | Metadata |
| `VOLUME` | Declare mount point |

**ENTRYPOINT vs CMD**: ENTRYPOINT defines the executable; CMD supplies default arguments (or the command if no ENTRYPOINT). `docker run image arg` replaces CMD. Use **exec form** (JSON array) so Java is PID 1 and receives `SIGTERM` for graceful shutdown; shell form wraps in `/bin/sh -c` which may not forward signals.

## 5. Dockerizing Spring Boot — three ways

### 5.1 Simple Dockerfile (jar built beforehand)
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY target/orders-1.0.0.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

### 5.2 Multi-stage + layered jar (best practice)
```dockerfile
# ---------- build stage ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B dependency:go-offline              # cached unless pom.xml changes
COPY src ./src
RUN mvn -B package -DskipTests

# ---------- extract layers ----------
FROM eclipse-temurin:21-jre AS layers
WORKDIR /layers
COPY --from=build /src/target/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted

# ---------- runtime stage ----------
FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 1001 app
COPY --from=layers /layers/extracted/dependencies/ ./
COPY --from=layers /layers/extracted/spring-boot-loader/ ./
COPY --from=layers /layers/extracted/snapshot-dependencies/ ./
COPY --from=layers /layers/extracted/application/ ./      # changes most often → last
USER 1001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "org.springframework.boot.loader.launch.JarLauncher"]
```
Why: final image contains only the JRE + app (no Maven/JDK/sources) → smaller & safer. Dependencies layer is cached across builds, so pushing a code change uploads only a few KB.

### 5.3 Cloud Native Buildpacks — no Dockerfile
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=acme/orders:1.0.0
```
```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <image>
      <name>acme/${project.artifactId}:${project.version}</name>
      <env><BP_JVM_VERSION>21</BP_JVM_VERSION></env>
    </image>
  </configuration>
</plugin>
```
Produces optimized layered images with a memory calculator, non-root user, SBOM. Alternative: **Jib** (Google) — builds images without Docker daemon.

## 6. JVM in containers
- JVM is container-aware (Java 10+): reads cgroup memory/CPU limits.
- Default max heap = 25% of container memory → too low for a dedicated container. Use `-XX:MaxRAMPercentage=70-75` (leave room for metaspace, threads, direct buffers, code cache).
- Set container memory **limit** ≥ heap + non-heap (~200–400MB extra for a typical Boot app).
- OOMKilled (exit 137) by the kernel ≠ Java `OutOfMemoryError` — it means the total process exceeded the container limit.
- CPU limits affect GC thread counts and `availableProcessors()`; very low CPU limits slow startup.
- Startup optimization: CDS/AOT cache, lazy init, GraalVM native images (tiny, ms startup), CRaC.

## 7. Image best practices
- Small, trusted base images (`eclipse-temurin:21-jre`, `-alpine`, distroless `gcr.io/distroless/java21-debian12`, Chainguard). Pin versions/digests; avoid `latest` in production.
- Multi-stage builds; `.dockerignore` (`target/`, `.git`, `*.md`, IDE files) to shrink build context.
- Order instructions from least to most frequently changing (cache efficiency).
- Combine `RUN` commands and clean package caches in the same layer.
- **Run as non-root**; read-only filesystem where possible; drop capabilities.
- No secrets in images or `ENV`/`ARG` (they're visible in history) — inject at runtime (env, mounted secrets, BuildKit `--secret` for build-time).
- One process per container; log to stdout/stderr.
- Scan images (Trivy, Grype, Docker Scout, Snyk) in CI; generate SBOMs; sign images (cosign).
- Tag with immutable versions (git SHA / semver).
- `HEALTHCHECK` for Compose/Swarm (Kubernetes uses probes instead).

## 8. Docker Compose — local multi-container environments
```yaml
# compose.yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shop
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: shop
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka:
    image: apache/kafka:3.8.0
    ports: ["9092:9092"]

  orders:
    build: ./orders                      # or image: acme/orders:1.0.0
    ports: ["8080:8080"]
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/shop     # service name as hostname!
      SPRING_DATASOURCE_USERNAME: shop
      SPRING_DATASOURCE_PASSWORD: shop
      SPRING_DATA_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
    deploy:
      resources:
        limits: { memory: 768m }

volumes:
  pgdata:
```
```bash
docker compose up -d --build
docker compose ps
docker compose logs -f orders
docker compose exec postgres psql -U shop
docker compose down            # keep volumes
docker compose down -v         # also delete volumes
```
- Compose creates a default network; services resolve each other by **service name**. `localhost` inside a container means the container itself.
- `depends_on` controls start order; with `condition: service_healthy` it waits for health checks.
- Spring Boot relaxed binding maps env vars (`SPRING_DATASOURCE_URL`) to properties.
- Profiles in compose (`profiles: [monitoring]`) for optional services; `.env` file for variables; override files (`compose.override.yaml`).
- Spring Boot's `spring-boot-docker-compose` module can start `compose.yaml` automatically during development.

## 9. Common issues
| Symptom | Cause / fix |
|---------|-------------|
| `Connection refused localhost:5432` from app container | Use service name `postgres`, not localhost |
| App starts before DB ready | `depends_on` with healthcheck, or retry on startup (Spring retries Hikari connections lazily) |
| Container exits immediately | Check `docker logs`; main process ended; wrong ENTRYPOINT |
| Exit code 137 | OOMKilled — raise memory limit or lower heap % |
| Exit code 143 | SIGTERM — normal stop |
| Huge images | Use JRE base, multi-stage, `.dockerignore` |
| Slow rebuilds | Copy pom first & cache dependencies; layered jars |
| Port already allocated | Another container/process using host port |
| Permission denied writing files | Non-root user needs writable dir (`/tmp`, volume) |

## 10. Interview Q&A
**Q. Image vs container?** Image: immutable layered template. Container: runtime instance with a writable layer.

**Q. Container vs VM?** Containers share the host kernel (light, fast); VMs virtualize hardware with full guest OS (heavier, stronger isolation).

**Q. What is a multi-stage build?** Multiple FROM stages; build tools in early stages, only artifacts copied to a slim final image.

**Q. CMD vs ENTRYPOINT?** ENTRYPOINT is the executable; CMD default args (or command). Use exec form for signal handling.

**Q. How do containers communicate in Compose?** Through the shared network using service names as DNS hostnames.

**Q. How do you persist data?** Volumes (or bind mounts); container filesystem is ephemeral.

**Q. How do you size the JVM in a container?** Container memory limit + `-XX:MaxRAMPercentage`, leaving headroom for non-heap memory.

**Q. How do you keep images secure?** Minimal base, non-root, no secrets, scan for CVEs, pin versions, sign images, rebuild regularly.
