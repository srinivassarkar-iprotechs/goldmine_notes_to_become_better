# 🐳 Docker Troubleshooting Guide — Top 10 All Time
> **The definitive reference for every Docker issue you'll ever face in production. Study it once. Use it forever.**

---

## #1 — Container Exits Immediately After Start

**What it means:** The container starts and stops within seconds — `docker ps` shows nothing, but `docker ps -a` shows it exited.

**Diagnose:**
```bash
docker ps -a                                  # See exit code
docker logs <container-id>                    # See what happened before death
docker inspect <container-id> | grep -A 5 "State"
```

**Exit codes explained:**

| Exit Code | Meaning |
|---|---|
| `0` | Container ran successfully and finished (not a crash) |
| `1` | Application error |
| `137` | OOM killed (`128 + 9` = SIGKILL) |
| `139` | Segfault (`128 + 11` = SIGSEGV) |
| `143` | Graceful shutdown (`128 + 15` = SIGTERM) |
| `126` | Permission denied on entrypoint |
| `127` | Entrypoint command not found |

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| No long-running foreground process | App must run in **foreground**, not background. Remove `&` or `daemon` flags from CMD |
| Script exits with error | Debug the entrypoint script; add `set -e` to catch failures |
| Missing dependency / wrong binary | Verify the CMD binary exists in the image: `docker run --rm <image> which <cmd>` |
| Wrong working directory | Check `WORKDIR` in Dockerfile |

**Override entrypoint to debug:**
```bash
docker run -it --entrypoint /bin/sh <image>
docker run -it --entrypoint /bin/bash <image>
```

---

## #2 — `Permission Denied` Inside Container

**What it means:** A process inside the container can't read/write files or execute binaries.

**Diagnose:**
```bash
docker exec -it <container> ls -la /path/to/problem
docker exec -it <container> id                 # What user is the process running as?
docker inspect <container> | grep -i user
```

**Root causes & fixes:**

**Fix 1 — Volume mount permission mismatch:**
```bash
# Host file owned by root, container runs as non-root user
# Option A: Change host file ownership
sudo chown -R 1000:1000 /host/path

# Option B: Set user in docker run
docker run -u $(id -u):$(id -g) <image>

# Option C: Fix in Dockerfile
RUN chown -R appuser:appuser /app
USER appuser
```

**Fix 2 — Read-only filesystem:**
```bash
# Don't mount volumes as read-only if app needs to write
docker run -v /host/path:/container/path:rw <image>   # ensure :rw, not :ro
```

**Fix 3 — Executing a script:**
```bash
# Inside the image build
RUN chmod +x /app/entrypoint.sh

# Or fix on host before build
chmod +x entrypoint.sh
```

**Best practice — always define a non-root user in Dockerfile:**
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

---

## #3 — Container Can't Connect to Host or External Services

**What it means:** `curl`, DB connections, API calls fail from inside the container.

**Diagnose:**
```bash
docker exec -it <container> ping 8.8.8.8               # internet reachability
docker exec -it <container> curl http://google.com     # HTTP
docker exec -it <container> nc -zv <host> <port>       # TCP port check
docker network inspect <network-name>
```

**Root causes & fixes:**

| Scenario | Fix |
|---|---|
| Connect to host from container | Use `host.docker.internal` (Mac/Windows) or `172.17.0.1` (Linux default bridge) |
| Container-to-container (same compose) | Use **service name** as hostname — Docker Compose creates DNS for you |
| Container-to-container (different network) | Connect both to same network: `docker network connect <network> <container>` |
| Port not exposed | Add `-p <host>:<container>` in `docker run` or `ports:` in Compose |
| Firewall / iptables blocking | Check host firewall: `iptables -L -n` or `ufw status` |
| No internet (air-gapped or proxy) | Set `HTTP_PROXY`, `HTTPS_PROXY` env vars or configure Docker daemon proxy |

**Multi-container networking in Compose:**
```yaml
services:
  app:
    image: myapp
    networks:
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  backend:
    driver: bridge
```
> `app` can reach `db` at hostname `db` — Docker Compose DNS handles it.

---

## #4 — `docker build` Fails

**What it means:** Image build errors out mid-build.

**Diagnose:**
```bash
docker build -t myimage . 2>&1 | tee build.log    # capture full output
docker build --no-cache -t myimage .               # rule out cache corruption
docker build --progress=plain -t myimage .         # verbose output (BuildKit)
```

**Common failures & fixes:**

| Error | Fix |
|---|---|
| `COPY failed: no such file or directory` | Check `.dockerignore` — file may be excluded. Check relative path in `COPY` |
| `RUN` command fails (exit code 1) | The shell command errored. Add `RUN echo $?` or split the command to isolate |
| `npm install` / `pip install` network failure | Check proxy settings. Add `--network=host` to the build step for testing |
| Base image not found | Check image name/tag: `docker pull <base-image>` separately first |
| Out of disk space during build | `docker system prune` to free space |
| `apt-get` fails with `404` | Run `apt-get update` before `apt-get install` in the **same** `RUN` layer |

**Dockerfile best practices that prevent failures:**
```dockerfile
# Always combine apt-get update + install in ONE layer
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Pin versions to avoid surprise breakage
RUN pip install requests==2.31.0

# Use multi-stage to isolate build failures
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

---

## #5 — Out of Disk Space (Docker eating your disk)

**What it means:** Docker accumulates unused images, stopped containers, volumes, and build cache — silently consuming gigabytes.

**Diagnose:**
```bash
docker system df                          # See exactly where space is used
docker system df -v                       # Verbose breakdown
du -sh /var/lib/docker                    # Total Docker disk usage on Linux
```

**Nuclear option (removes everything unused):**
```bash
docker system prune -a --volumes          # Removes: stopped containers, unused images, volumes, networks, build cache
```

**Surgical cleanup:**
```bash
# Remove stopped containers only
docker container prune

# Remove unused images (not referenced by any container)
docker image prune -a

# Remove unused volumes (CAREFUL — data loss risk)
docker volume prune

# Remove build cache
docker builder prune -a

# Remove specific old images
docker images | grep "months ago" | awk '{print $3}' | xargs docker rmi
```

**Prevent it — automate cleanup:**
```bash
# Add to cron (daily cleanup of dangling images)
0 2 * * * docker image prune -f >> /var/log/docker-prune.log 2>&1
```

**Change Docker's data root (if disk is wrong partition):**
```json
// /etc/docker/daemon.json
{
  "data-root": "/mnt/large-disk/docker"
}
```

---

## #6 — Container Running But App Not Accessible

**What it means:** `docker ps` shows the container as `Up`, but you can't reach the app.

**Diagnose:**
```bash
docker ps                                          # Check PORTS column
docker inspect <container> | grep -A 10 Ports
docker exec -it <container> ss -tlnp              # What ports is the app ACTUALLY listening on?
curl http://localhost:<port>
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Port not published | Add `-p 8080:80` to `docker run` |
| App listening on wrong interface | App must listen on `0.0.0.0`, not `127.0.0.1` inside the container |
| Port published but to wrong host port | Check: `docker port <container>` |
| App crashed silently | Check `docker logs <container>` — app may have died after starting |
| Firewall on host blocking the port | `ufw allow <port>` or check cloud security groups |
| Published on wrong network interface | Use `--publish 0.0.0.0:8080:80` to bind all interfaces |

**Verify app is actually listening inside the container:**
```bash
docker exec -it <container> ss -tlnp
# Or:
docker exec -it <container> netstat -tlnp
# Should show: 0.0.0.0:80 or :::80 — NOT 127.0.0.1:80
```

---

## #7 — `docker-compose up` Fails / Services Won't Start

**What it means:** Compose fails to bring up services cleanly — dependency issues, port conflicts, or config errors.

**Diagnose:**
```bash
docker-compose up                          # Run in foreground to see all logs
docker-compose up --build                  # Force rebuild
docker-compose logs <service-name>         # Logs for specific service
docker-compose config                      # Validate compose file syntax
docker-compose ps                          # Status of all services
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Port already in use | `lsof -i :<port>` to find conflicting process, kill it |
| Service dependency not ready | Use `healthcheck` + `depends_on: condition: service_healthy` |
| `.env` file missing | Create `.env` or pass vars with `docker-compose --env-file` |
| Volume mount path doesn't exist on host | Create the host directory first |
| Image not found locally | Add `build:` section or `docker pull` the image first |
| Compose file version incompatibility | Check `version:` field vs Docker Compose version installed |

**Proper dependency handling (not just `depends_on`):**
```yaml
services:
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy   # Wait for HEALTHY, not just started

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

**Nuke and restart cleanly:**
```bash
docker-compose down -v        # Stops + removes containers AND volumes
docker-compose up --build     # Rebuild everything from scratch
```

---

## #8 — Slow Docker Build Times

**What it means:** Builds take minutes when they should take seconds — usually a cache invalidation problem.

**Diagnose:**
```bash
docker build --progress=plain -t myimage .    # See timing of each layer
```

**Root causes & fixes:**

**Problem: Cache busted too early — copying source code before installing dependencies**
```dockerfile
# ❌ BAD — Any code change invalidates npm install cache
COPY . .
RUN npm install

# ✅ GOOD — Dependency install cached separately from code changes
COPY package*.json ./
RUN npm install          # Only re-runs when package.json changes
COPY . .                 # Code changes don't bust dependency cache
```

**Problem: Not using BuildKit**
```bash
# Enable BuildKit for parallel builds + better caching
DOCKER_BUILDKIT=1 docker build -t myimage .
# Or set permanently in daemon.json:
# { "features": { "buildkit": true } }
```

**Problem: Large build context (slow COPY/ADD)**
```bash
# Check what you're sending to the build daemon
docker build . 2>&1 | head -1   # "Sending build context to Docker daemon  X.XXkB"

# Create .dockerignore
cat .dockerignore
node_modules
.git
*.log
dist
.env
__pycache__
```

**Use cache mounts for package managers:**
```dockerfile
# Cache pip downloads across builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## #9 — Container Uses Too Much Memory / CPU

**What it means:** Containers consuming runaway resources, impacting host or sibling containers.

**Diagnose:**
```bash
docker stats                               # Live resource usage (all containers)
docker stats <container-name>              # Single container
docker inspect <container> | grep -i memory
```

**Set resource limits:**
```bash
# Memory limit
docker run -m 512m --memory-swap 512m <image>    # 512MB RAM, no swap

# CPU limit
docker run --cpus="1.5" <image>                   # Max 1.5 CPU cores
docker run --cpu-shares=512 <image>               # Relative weight (default 1024)

# Both
docker run -m 512m --cpus="1.0" <image>
```

**In docker-compose:**
```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
```

**Identify memory leak pattern:**
```bash
# Watch memory grow over time
watch -n 5 docker stats --no-stream <container>

# Trigger OOM manually to confirm limit behavior
docker run -m 100m stress --vm 1 --vm-bytes 200M
```

---

## #10 — Docker Daemon Not Starting / `Cannot connect to Docker daemon`

**What it means:** The Docker daemon is dead, misconfigured, or you don't have permission to talk to the socket.

**Diagnose:**
```bash
systemctl status docker                    # Is the service running?
journalctl -xu docker                      # Full daemon logs
docker info                                # Quick connectivity test
ls -la /var/run/docker.sock               # Check socket exists + permissions
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Docker not running | `sudo systemctl start docker && sudo systemctl enable docker` |
| Current user not in `docker` group | `sudo usermod -aG docker $USER` then **log out and back in** |
| Corrupted daemon.json | `cat /etc/docker/daemon.json` → fix JSON syntax errors |
| Port conflict (dockerd startup failure) | Check logs: `journalctl -xu docker` |
| Storage driver corruption | Backup + clean `/var/lib/docker` — nuclear option |
| Out of disk space (daemon won't start) | Free disk space, then restart Docker |

**Full daemon restart procedure:**
```bash
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo systemctl start docker
sudo systemctl status docker
```

**Verify group membership (common beginner mistake):**
```bash
groups                          # Does 'docker' appear?
id                              # Full group listing
# If not listed, you haven't logged out since running usermod
newgrp docker                   # Activate group in current session without logout
```

**Daemon config validation:**
```bash
sudo dockerd --validate         # Validate daemon.json without starting
cat /etc/docker/daemon.json     # Check for JSON errors (trailing commas, etc.)
```

---

## 🧰 Universal Docker Debug Toolkit

```bash
# See ALL containers (including stopped)
docker ps -a

# Follow live logs
docker logs -f <container>
docker logs --tail=100 <container>           # Last 100 lines

# Get a shell in running container
docker exec -it <container> /bin/bash
docker exec -it <container> /bin/sh          # For Alpine-based images

# Inspect everything about a container
docker inspect <container>

# See real-time events from Docker daemon
docker events

# Check network configuration
docker network ls
docker network inspect bridge

# Run a throwaway debug container
docker run --rm -it --network container:<target> nicolaka/netshoot

# Copy files in/out
docker cp <container>:/path/to/file ./local-copy
docker cp ./local-file <container>:/path/in/container

# Check image layers and sizes
docker history <image>
docker image inspect <image>
```

---

## 📐 Mental Model: Docker Debugging Decision Tree

```
Problem?
├── Container exits immediately       → #1  Check logs + exit code + entrypoint
├── Permission denied                 → #2  User/ownership/chmod mismatch
├── Can't reach network/service       → #3  Networking — host.docker.internal, ports, networks
├── Build fails                       → #4  Layer error, cache issue, COPY path
├── Disk full                         → #5  docker system prune -a --volumes
├── Running but can't access          → #6  Port binding, app listening on 127.0.0.1
├── Compose won't start               → #7  Port conflict, missing deps, healthchecks
├── Build too slow                    → #8  .dockerignore, layer order, BuildKit
├── High memory/CPU                   → #9  Set --memory and --cpus limits
└── Daemon won't start                → #10 systemctl, group permissions, daemon.json
```

---

*Master these 10 and you will debug Docker faster than any Stackoverflow search. No exceptions.*
