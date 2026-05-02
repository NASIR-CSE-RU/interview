# 🎯 Docker Interview Preparation — Complete Study Guide

A comprehensive interview-focused Docker study material covering everything from basics to advanced concepts.

---

## 📋 Table of Contents

1. [Core Concepts](#-part-1-core-concepts)
2. [Top 60+ Interview Questions](#-part-2-top-60-interview-questions--answers)
3. [Command Cheat Sheet](#️-part-3-command-cheat-sheet)
4. [Scenario-Based Questions](#-part-4-scenario-based-questions)
5. [Tricky Questions (FAANG-Level)](#-part-5-tricky-questions-faang-level)
6. [Hands-On Practice Tasks](#-part-6-hands-on-practice-tasks)
7. [System Design with Docker](#️-part-7-system-design-with-docker)
8. [Common Mistakes to Avoid](#️-part-8-common-mistakes-to-avoid)
9. [Final Interview Tips](#-part-9-final-interview-tips)

---

## 🧠 PART 1: CORE CONCEPTS

### What is Docker?

Docker is a **containerization platform** that packages applications with all dependencies into lightweight, portable containers.

### Container vs Virtual Machine

| Feature      | Container             | Virtual Machine    |
| ------------ | --------------------- | ------------------ |
| OS           | Shares host OS kernel | Full OS included   |
| Size         | MBs                   | GBs                |
| Boot time    | Seconds               | Minutes            |
| Performance  | Near-native           | Overhead           |
| Isolation    | Process-level         | Hardware-level     |

### Docker Architecture

```
[Docker CLI] → [Docker API] → [Docker Daemon] → [Containerd] → [runc] → [Container]
```

### Key Components

- **Docker Daemon (dockerd)** → background service
- **Docker Client (CLI)** → user interface
- **Docker Registry** → stores images (Docker Hub)
- **Docker Objects** → Images, Containers, Networks, Volumes

---

## ❓ PART 2: TOP 60+ INTERVIEW QUESTIONS & ANSWERS

### 🟢 Beginner Level (Q1–Q20)

---

#### Q1. What is Docker and why is it used?

Docker is an open-source containerization platform that allows developers to package applications with dependencies into containers. It's used for:

- Consistent environments (dev, test, prod)
- Faster deployments
- Better resource utilization
- Microservices architecture
- Solving "It works on my machine" problem

---

#### Q2. What is a Docker Image?

A Docker image is a **read-only template** (blueprint) used to create containers. It contains:

- Application code
- Runtime
- Libraries
- Environment variables
- Configuration files

Built using a `Dockerfile` and stored in a registry.

---

#### Q3. What is a Docker Container?

A container is a **running instance** of a Docker image. It's:

- Isolated, lightweight, and executable
- Has its own filesystem, networking, and process space
- Mutable (you can change runtime state)

> **Image = Class, Container = Object** (OOP analogy)

---

#### Q4. What is a Dockerfile?

A text file with instructions to build a Docker image automatically.

Common instructions:

| Instruction       | Purpose                       |
| ----------------- | ----------------------------- |
| `FROM`            | Base image                    |
| `WORKDIR`         | Set working directory         |
| `COPY` / `ADD`    | Copy files                    |
| `RUN`             | Execute commands at build time|
| `CMD`             | Default command at runtime    |
| `ENTRYPOINT`      | Main executable               |
| `EXPOSE`          | Document ports                |
| `ENV`             | Environment variables         |
| `ARG`             | Build-time arguments          |
| `VOLUME`          | Mount points                  |

---

#### Q5. Difference between CMD and ENTRYPOINT?

| CMD                                  | ENTRYPOINT                  |
| ------------------------------------ | --------------------------- |
| Provides default arguments           | Defines main executable     |
| Can be overridden by `docker run`    | Cannot be easily overridden |
| Used for default behavior            | Used for fixed behavior     |

**Best practice:** Use `ENTRYPOINT` for the executable, `CMD` for default arguments.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

---

#### Q6. Difference between COPY and ADD?

| COPY                  | ADD                                       |
| --------------------- | ----------------------------------------- |
| Copies files only     | Copies + extracts tar + supports URLs     |
| Simple, predictable   | More features but less explicit           |
| **Recommended**       | Use only when needed                      |

---

#### Q7. Difference between ARG and ENV?

| ARG                          | ENV                          |
| ---------------------------- | ---------------------------- |
| Build-time variable          | Runtime variable             |
| Not available after build    | Available in container       |
| Set via `--build-arg`        | Set via `-e` or in Dockerfile|

---

#### Q8. What is Docker Hub?

Cloud-based registry service for sharing Docker images. Acts as the default public registry where you can:

- Pull official images
- Push your own images
- Set up automated builds

---

#### Q9. Difference between `docker run` and `docker start`?

- `docker run` → creates a NEW container from an image
- `docker start` → starts an EXISTING stopped container

---

#### Q10. What does `docker ps` do?

Lists running containers. Use `-a` to show all (including stopped).

---

#### Q11. How do you stop and remove a container?

```bash
docker stop <container_id>
docker rm <container_id>
docker rm -f <container_id>   # force remove running
```

---

#### Q12. What is the purpose of `-d` flag?

Runs the container in **detached mode** (background), freeing up the terminal.

---

#### Q13. How do port mappings work in Docker?

```bash
docker run -p 8080:80 nginx
```

- Host port 8080 → Container port 80
- Traffic to `localhost:8080` is forwarded to container's port 80

---

#### Q14. What are Docker Volumes?

Persistent storage mechanism that:

- Survives container deletion
- Can be shared between containers
- Managed by Docker

**Types:**

1. **Named volumes** → managed by Docker
2. **Bind mounts** → host directory mounted
3. **tmpfs** → in-memory storage

---

#### Q15. Difference between Volume and Bind Mount?

| Named Volume          | Bind Mount              |
| --------------------- | ----------------------- |
| Managed by Docker     | User-managed path       |
| Stored in Docker area | Anywhere on host        |
| Better for production | Better for development  |
| Portable              | Host-specific           |

---

#### Q16. What is Docker Compose?

A tool to define and run **multi-container** applications using a YAML file (`docker-compose.yml`).

```bash
docker compose up -d
docker compose down
```

---

#### Q17. What is the use of `.dockerignore`?

Lists files/folders to exclude from build context (similar to `.gitignore`).

- Reduces image size
- Speeds up builds
- Prevents secrets from leaking

---

#### Q18. How do you check container logs?

```bash
docker logs <container>
docker logs -f --tail 50 <container>
```

---

#### Q19. How to enter a running container?

```bash
docker exec -it <container> bash
docker exec -it <container> sh   # if no bash
```

---

#### Q20. Difference between `docker exec` and `docker attach`?

- `exec` → runs a NEW process inside the container
- `attach` → connects to the MAIN running process

---

### 🟡 Intermediate Level (Q21–Q40)

---

#### Q21. What is a Docker Network? Types?

1. **Bridge** (default) → single host networking
2. **Host** → no isolation, uses host's network
3. **None** → no network
4. **Overlay** → multi-host (Swarm)
5. **Macvlan** → assigns MAC address to container

---

#### Q22. How do containers communicate with each other?

- Same network → use container name as hostname
- Different networks → connect container to multiple networks
- Via published ports
- Custom bridge networks have built-in DNS

---

#### Q23. What are Docker Image Layers?

Each Dockerfile instruction creates a layer. Layers are:

- **Cached** (faster builds)
- **Read-only** (except top container layer)
- **Shared** between images

> Tip: Order Dockerfile instructions from **least to most frequently changing**.

---

#### Q24. What is multi-stage build? Why use it?

Building an image in multiple stages where only the final artifacts move to the final image.

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**Benefits:**

- Smaller final image
- No build tools in production
- More secure

---

#### Q25. How to reduce Docker image size?

1. Use **smaller base images** (alpine, distroless)
2. **Multi-stage builds**
3. **Combine RUN commands** with `&&`
4. **Clean up cache** (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`)
5. Use **.dockerignore**
6. Avoid unnecessary packages
7. Use specific tags (avoid `latest`)

---

#### Q26. What is Docker Swarm?

Docker's native **clustering and orchestration** tool. Turns multiple Docker hosts into a single virtual host.

**Features:**

- Service discovery
- Load balancing
- Rolling updates
- Scaling
- High availability

---

#### Q27. Docker Swarm vs Kubernetes?

| Feature         | Swarm        | Kubernetes        |
| --------------- | ------------ | ----------------- |
| Setup           | Easy         | Complex           |
| Learning curve  | Low          | High              |
| Auto-scaling    | Manual       | Built-in HPA      |
| Load balancing  | Built-in     | Requires services |
| Community       | Smaller      | Massive           |
| Use case        | Small/medium | Enterprise        |

---

#### Q28. Difference between Docker Stack and Docker Compose?

| Compose             | Stack                  |
| ------------------- | ---------------------- |
| Single host         | Multi-host (Swarm)     |
| Development         | Production             |
| `docker compose up` | `docker stack deploy`  |

---

#### Q29. What are Docker Secrets?

Encrypted way to store sensitive data (passwords, API keys, certs) in Swarm mode.

```bash
echo "mypassword" | docker secret create db_pass -
```

Mounted as a file at `/run/secrets/<name>` (not env var, for security).

---

#### Q30. What are Healthchecks?

Defines how Docker checks if a container is healthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost/health || exit 1
```

States: `starting`, `healthy`, `unhealthy`.

---

#### Q31. Difference between EXPOSE and `-p`?

- `EXPOSE` → **documentation only**, doesn't publish ports
- `-p` (publish) → **actually maps** host port to container port

---

#### Q32. What is the role of `init` process in containers?

By default, containers run a single process. The `--init` flag adds a tiny init process that:

- Reaps zombie processes
- Forwards signals properly

```bash
docker run --init <image>
```

---

#### Q33. What happens when you run `docker run`?

1. Docker checks if image exists locally
2. If not, pulls from registry
3. Creates a writable container layer
4. Allocates IP, network interface
5. Executes the command (`CMD`/`ENTRYPOINT`)
6. Captures stdout/stderr to logs

---

#### Q34. How does Docker achieve isolation?

Using Linux kernel features:

- **Namespaces** → process, network, mount, IPC, user, UTS isolation
- **cgroups** → resource limits (CPU, memory, I/O)
- **Union file systems** → layered images
- **Capabilities** → restricted privileges

---

#### Q35. Difference between containerd and runc?

- **containerd** → high-level container runtime (manages lifecycle)
- **runc** → low-level runtime (creates containers per OCI spec)

Hierarchy: `Docker → containerd → runc → container`

---

#### Q36. What is OCI?

**Open Container Initiative** — standardizes container formats and runtimes for interoperability across Docker, Podman, etc.

---

#### Q37. Restart Policies in Docker?

```bash
docker run --restart=<policy> <image>
```

- `no` (default)
- `on-failure[:max-retries]`
- `always`
- `unless-stopped`

---

#### Q38. How to limit container resources?

```bash
docker run --memory="512m" --cpus="1.5" <image>
```

---

#### Q39. What is a dangling image?

Untagged images (`<none>:<none>`) that result from rebuilds.

```bash
docker image prune          # remove dangling
docker image prune -a       # remove all unused
```

---

#### Q40. How to share an image without Docker Hub?

```bash
docker save -o myimage.tar myimage:v1
# Transfer file
docker load -i myimage.tar
```

---

### 🔴 Advanced Level (Q41–Q60)

---

#### Q41. How to ensure security in Docker containers?

1. Use **minimal/official images**
2. Don't run as **root** (use `USER` directive)
3. **Scan images** (Trivy, Snyk, Docker Scout)
4. Enable **Docker Content Trust**
5. Use **secrets** (not ENV vars)
6. **Limit capabilities** (`--cap-drop ALL`)
7. **Read-only filesystems** (`--read-only`)
8. **Resource limits** (cgroups)
9. **Network segmentation**
10. Regular updates

---

#### Q42. What is rootless Docker?

Running Docker daemon and containers without root privileges, reducing attack surface.

---

#### Q43. What is BuildKit?

Next-gen Docker builder offering:

- Parallel build steps
- Better caching
- Build secrets (`--secret`)
- Multi-platform builds

```bash
DOCKER_BUILDKIT=1 docker build .
```

---

#### Q44. What is Docker Buildx?

CLI plugin for **multi-architecture** builds (AMD64, ARM64) using BuildKit.

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
```

---

#### Q45. How does Docker handle logs in production?

- Configurable **logging drivers**: `json-file`, `syslog`, `journald`, `fluentd`, `gelf`, `awslogs`
- Use centralized logging: **ELK / Loki / Splunk**
- Log to `stdout`/`stderr`, not files inside container

---

#### Q46. Common Docker performance issues and solutions?

| Issue            | Solution                |
| ---------------- | ----------------------- |
| Large images     | Multi-stage, alpine     |
| Slow builds      | Layer caching, BuildKit |
| Memory leaks     | Set resource limits     |
| Disk full        | Prune unused resources  |
| Network latency  | Use host network if needed |

---

#### Q47. Stateless vs Stateful containers?

- **Stateless** → no persistent data (web servers, APIs)
- **Stateful** → persistent data (databases) → needs **volumes**

---

#### Q48. How does Docker handle DNS?

- Default bridge → uses host's DNS
- Custom networks → built-in DNS server resolves container names
- Override with `--dns` flag

---

#### Q49. Difference between `docker pause` and `docker stop`?

- `pause` → freezes processes (using cgroup freezer), instant
- `stop` → sends SIGTERM, then SIGKILL after grace period

---

#### Q50. How do you debug a container that won't start?

1. Check logs: `docker logs <container>`
2. Inspect: `docker inspect <container>`
3. Check exit code: `docker ps -a`
4. Override entrypoint: `docker run -it --entrypoint sh <image>`
5. Check events: `docker events`

---

#### Q51. What is Docker Content Trust (DCT)?

Image signing system that ensures only signed/trusted images can be pulled or pushed.

```bash
export DOCKER_CONTENT_TRUST=1
```

---

#### Q52. Explain copy-on-write (CoW) in Docker.

- Image layers are read-only
- When a file is modified, it's **copied** to the container's writable layer
- Saves disk space and speeds up startup

---

#### Q53. How does container networking work internally?

- Each container gets a **virtual ethernet pair (veth)**
- Connected to a Linux **bridge (docker0)** by default
- Uses **iptables** for NAT and port forwarding
- DNS resolution via **embedded DNS server**

---

#### Q54. What are init containers (in Kubernetes context)?

Run BEFORE main containers. Used for:

- Setup tasks
- Wait for dependencies
- Database migrations

---

#### Q55. What is sidecar pattern?

Running a helper container alongside main app container in the same Pod (or shared network).

Examples: logging agent, proxy, monitoring.

---

#### Q56. What is Docker Scout?

Modern vulnerability management:

- AI-driven CVE prioritization
- Image analysis across orgs
- CI/CD integration
- Suggests safer base images

---

#### Q57. What is GitOps and how does Docker fit?

GitOps = Git as single source of truth for deployments.

- Push code → Build image → Push to registry → ArgoCD/Flux syncs cluster

---

#### Q58. How do you handle zero-downtime deployments?

- **Rolling updates** (Swarm/K8s)
- **Blue-Green deployment**
- **Canary releases**
- Use **healthchecks** + **readiness probes**

---

#### Q59. How does Docker work on Windows/Mac?

Uses a lightweight **Linux VM** (HyperKit on Mac, WSL2/Hyper-V on Windows) since containers need Linux kernel.

---

#### Q60. What is distroless image?

Stripped-down images with only the application and runtime — no shell, package manager, or extra libraries.

**Benefits:** smaller, more secure, fewer CVEs.

---

## 🛠️ PART 3: COMMAND CHEAT SHEET

### Container Lifecycle

```bash
docker run -d --name web -p 80:80 nginx
docker ps
docker ps -a
docker stop <container>
docker start <container>
docker restart <container>
docker rm <container>
docker rm -f <container>
docker logs -f <container>
docker exec -it <container> bash
docker inspect <container>
docker stats
```

### Image Management

```bash
docker images
docker pull <image>:<tag>
docker build -t myapp:v1 .
docker tag myapp:v1 user/myapp:v1
docker push user/myapp:v1
docker rmi <image>
docker image prune -a
```

### Volumes & Networks

```bash
docker volume create myvol
docker volume ls
docker volume inspect myvol
docker volume rm myvol
docker network create mynet
docker network connect mynet <container>
docker network ls
```

### Compose

```bash
docker compose up -d
docker compose down -v
docker compose logs -f
docker compose ps
docker compose exec <service> bash
docker compose build --no-cache
```

### System

```bash
docker system df
docker system prune -a
docker info
docker version
```

---

## 🎬 PART 4: SCENARIO-BASED QUESTIONS

---

### S1. Your container keeps crashing immediately. How do you debug?

1. `docker logs <container>` → check errors
2. `docker ps -a` → check exit code (137=OOM, 1=app error, 139=segfault)
3. `docker inspect <container>` → check state
4. Run interactively: `docker run -it --entrypoint sh <image>`
5. Check resource limits, missing env vars, missing files

---

### S2. Your image size is 2GB. How do you reduce it?

1. Switch to alpine/distroless base
2. Use multi-stage builds
3. Combine RUN commands
4. Clean cache files
5. Add `.dockerignore`
6. Remove dev dependencies in production

---

### S3. App in container can't connect to DB in another container?

- Are they on the same network? `docker network inspect`
- Use container name as hostname (not `localhost`)
- Check DB is listening on `0.0.0.0` (not `127.0.0.1`)
- Verify ports and firewall rules

---

### S4. How would you deploy a Node.js + Postgres + Redis app?

Use **Docker Compose**:

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    environment:
      DB_HOST: db
      REDIS_HOST: redis

  db:
    image: postgres:15-alpine
    volumes:
      - dbdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:alpine

volumes:
  dbdata:
```

---

### S5. How to migrate database data when changing containers?

1. Always use **named volumes** for DB
2. To backup: `docker exec db pg_dump > backup.sql`
3. To restore: `docker exec -i newdb psql < backup.sql`
4. Or copy volume data with `docker cp`

---

### S6. App needs HTTPS. How would you implement it?

- Use **Traefik** or **Nginx** as reverse proxy
- Auto SSL with **Let's Encrypt**
- Or terminate SSL at load balancer (ALB/NGINX)

---

### S7. Container is consuming too much memory. Solution?

```bash
docker run --memory="512m" --memory-swap="1g" <image>
```

- Add resource limits in Compose/Swarm
- Profile app with `docker stats`
- Check for memory leaks

---

### S8. How do you implement zero-downtime deployment?

- Build new image with new tag
- Use Swarm rolling updates: `docker service update --image newimg`
- Configure: `--update-parallelism 1 --update-delay 10s`
- Have proper healthchecks

---

## 🧩 PART 5: TRICKY QUESTIONS (FAANG-Level)

---

### T1. Can a container have multiple processes?

Technically yes, but **best practice = one process per container**. Use `supervisord` or sidecar containers if needed.

---

### T2. What happens if you `docker rm` a running container without `-f`?

Error: "You cannot remove a running container." Must stop it first.

---

### T3. Are containers immutable?

**Images** are immutable. **Containers** have a writable layer on top — but it's discarded on removal. Best practice: treat containers as ephemeral.

---

### T4. What is the order of execution in a Dockerfile?

Top to bottom. Each instruction creates a new cached layer.

---

### T5. Can two containers share the same volume?

Yes! Multiple containers can mount the same volume for data sharing.

---

### T6. What if Docker daemon crashes?

- Running containers continue (with `live-restore` enabled in `daemon.json`)
- Without it, all containers stop

---

### T7. Why is `latest` tag dangerous?

- It's mutable — same tag can point to different images
- Can break production unexpectedly
- Always use **specific version tags** in production

---

### T8. How does Docker layer caching work?

- Each instruction = a layer hash based on content
- If content unchanged, cached layer is reused
- Once a layer changes, ALL subsequent layers rebuild

> **Tip:** Place frequently changing instructions (like `COPY . .`) AFTER `RUN npm install`.

---

### T9. Can you run Docker inside Docker (DinD)?

Yes, two ways:

1. Mount Docker socket: `-v /var/run/docker.sock:/var/run/docker.sock`
2. Run Docker daemon inside container (DinD image)

⚠️ Security risk — be cautious.

---

### T10. What's the difference between docker stop signals?

- `docker stop` → sends `SIGTERM`, waits 10s, then `SIGKILL`
- `docker kill` → immediate `SIGKILL`
- Configure grace period: `docker stop -t 30`

---

## 💪 PART 6: HANDS-ON PRACTICE TASKS

### Beginner

- [ ] Run nginx, expose on port 8080, mount custom HTML
- [ ] Create a Dockerfile for a simple Node.js/Python app
- [ ] Push your image to Docker Hub
- [ ] Use volumes to persist MySQL data

### Intermediate

- [ ] Multi-stage build for a React/Go app
- [ ] Docker Compose: app + database + Redis
- [ ] Set up custom networks and verify connectivity
- [ ] Implement healthchecks for a web app
- [ ] Limit container resources (CPU/memory)

### Advanced

- [ ] Initialize a Swarm cluster, deploy stack
- [ ] Implement rolling updates with zero downtime
- [ ] Use Docker secrets for credentials
- [ ] Build multi-arch image with Buildx
- [ ] Set up Prometheus + Grafana for container monitoring
- [ ] Scan images with Trivy

---

## 🏗️ PART 7: SYSTEM DESIGN WITH DOCKER

### Q: Design a microservices architecture using Docker

**Components:**

- **API Gateway** (Traefik/Nginx) → routes traffic
- **Auth Service** → handles login/JWT
- **User Service** → manages users
- **Payment Service** → processes payments
- **Database** (per service) → PostgreSQL
- **Cache** → Redis
- **Message Queue** → RabbitMQ/Kafka
- **Logging** → ELK / Loki
- **Monitoring** → Prometheus + Grafana
- **Orchestration** → Kubernetes / Swarm

**Best Practices:**

- One container = one responsibility
- Use service discovery (DNS)
- Centralized logging
- Health checks for all services
- Circuit breakers (resilience)
- API versioning
- Secrets management

---

### Q: Design CI/CD pipeline with Docker

```
[Code Push] → [GitHub Actions]
              ↓
         [Run Tests]
              ↓
         [Build Image]
              ↓
        [Scan with Trivy]
              ↓
       [Push to Registry]
              ↓
      [Deploy to Staging]
              ↓
     [Integration Tests]
              ↓
       [Deploy to Prod]
              ↓
        [Monitor + Alert]
```

---

## ⚠️ PART 8: COMMON MISTAKES TO AVOID

1. ❌ Using `latest` tag in production
2. ❌ Running containers as root
3. ❌ Storing data in container filesystem
4. ❌ Hardcoding secrets in Dockerfile/images
5. ❌ Not using `.dockerignore`
6. ❌ Multiple processes in one container without supervisor
7. ❌ Not setting resource limits
8. ❌ Ignoring image vulnerabilities
9. ❌ Forgetting to clean up unused resources
10. ❌ Using `ADD` instead of `COPY` unnecessarily
11. ❌ Not using multi-stage builds
12. ❌ Mounting Docker socket carelessly
13. ❌ Exposing unnecessary ports
14. ❌ Skipping healthchecks
15. ❌ Long Dockerfile RUN commands without `&&` chain

---

## 🎯 PART 9: FINAL INTERVIEW TIPS

### 🗣️ How to Answer Effectively

1. **Start with definition**, then **example**, then **use case**
2. Use real-world analogies (containers = shipping containers)
3. Mention **best practices** in every answer
4. Talk about **trade-offs** (Swarm vs K8s, Volume vs Bind)
5. If unsure, walk through your **thought process**

### 📝 Topics to Revise the Night Before

- Dockerfile instructions and order
- CMD vs ENTRYPOINT
- Volumes vs Bind mounts
- Networking modes
- Multi-stage builds
- Image optimization
- Docker Compose syntax
- Security best practices
- Common debugging commands

### 🎤 Questions YOU Should Ask Interviewer

- "What container orchestration do you use?"
- "How do you handle CI/CD with Docker?"
- "What's your image scanning/security strategy?"
- "How do you monitor containers in production?"
- "Do you use Swarm, Kubernetes, or managed services like ECS?"

---

## 📊 QUICK REVISION CARDS

| Concept    | Key Point                  |
| ---------- | -------------------------- |
| Image      | Read-only template         |
| Container  | Running instance           |
| Dockerfile | Build instructions         |
| Volume     | Persistent storage         |
| Network    | Container communication    |
| Compose    | Multi-container manager    |
| Swarm      | Native orchestration       |
| Layer      | Cached build step          |
| Registry   | Image storage              |
| Daemon     | Background service         |

---

## ✅ PRE-INTERVIEW CHECKLIST

- [ ] Practiced 30+ Docker commands
- [ ] Built at least 3 Dockerfiles from scratch
- [ ] Created multi-stage builds
- [ ] Set up a Compose project (3+ services)
- [ ] Understand layer caching deeply
- [ ] Know networking modes
- [ ] Understand volumes vs bind mounts
- [ ] Can explain CMD vs ENTRYPOINT
- [ ] Know security best practices
- [ ] Familiar with Swarm basics
- [ ] Comfortable debugging containers
- [ ] Reviewed scenario-based questions

---

## 🚀 You've Got This!

Master these concepts, practice hands-on, and you'll be **interview-ready** for any Docker role — from Junior DevOps to Senior Cloud Engineer roles at FAANG companies! 🐳💪

**Good luck with your interview!** 🎯
