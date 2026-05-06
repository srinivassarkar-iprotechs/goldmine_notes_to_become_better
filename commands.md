# 🛠️ DevOps Cheat Sheet — 80/20 Edition
> Maximum real-world value. Minimum fluff. Skim in <30 seconds.

---

## 🐧 LINUX

---

### 🔹 `grep`

**What**
→ Search text patterns across files or streams

**Why**
→ The fastest way to locate errors, configs, or keywords in logs and codebases

**When to Use**
- Hunting `ERROR` / `WARN` in log files
- Checking if a config key exists
- Filtering command output in pipelines

**Core Usage**
```bash
grep -R "ERROR" /var/log/app/          # recursive search
grep -i "timeout" app.log              # case-insensitive
grep -n "failed" deploy.log            # show line numbers
grep -v "DEBUG" app.log                # exclude matches
grep -E "ERROR|WARN|FATAL" app.log     # regex OR match
```

**Pro Tips**
- Pair with `tail -f` for live log filtering: `tail -f app.log | grep ERROR`
- Use `--color=always` when piping to `less`: `grep --color=always "ERROR" app.log | less -R`
- `-l` lists only filenames, not matches — useful for bulk searches

---

### 🔹 `awk`

**What**
→ Field-based text processing — extract, transform, and compute from structured output

**Why**
→ Turns raw command output into actionable data without writing a script

**When to Use**
- Extract specific columns from `ps`, `df`, `netstat` output
- Sum or compute values from logs
- Transform CSV/TSV data in pipelines

**Core Usage**
```bash
awk '{print $1, $3}' file              # print columns 1 and 3
awk -F',' '{print $2}' data.csv        # CSV: use comma delimiter
ps aux | awk '$3 > 80 {print $0}'     # processes using >80% CPU
awk 'NR==5, NR==10 {print}' file      # print lines 5–10
awk '{sum += $4} END {print sum}' log  # sum column 4
```

**Pro Tips**
- `-F` sets the field separator (default: whitespace)
- `NF` = number of fields, `NR` = line number — both useful in conditions
- Combine with `grep` first to narrow data, then `awk` to shape it

---

### 🔹 `sed`

**What**
→ Stream editor — find/replace and transform text in-place or in pipelines

**Why**
→ Fastest way to bulk-edit configs or sanitize output without opening a file

**When to Use**
- In-place config substitution (CI/CD variable injection)
- Stripping or replacing patterns from log streams
- Transforming output before passing to next command

**Core Usage**
```bash
sed 's/old/new/g' file                     # replace all occurrences
sed -i 's/localhost/prod-db/g' config.env  # in-place edit
sed '/^#/d' file                           # delete comment lines
sed -n '10,20p' file                       # print lines 10–20
sed 's/ERROR/[ERROR]/g' app.log            # annotate log entries
```

**Pro Tips**
- Always backup before `-i` in production: `sed -i.bak 's/x/y/g' file`
- Chain multiple expressions: `sed -e 's/a/b/g' -e 's/c/d/g' file`
- Use with `find` + `xargs` for bulk file edits

**Common Mistakes**
- Forgetting `-i` behavior differs between macOS and Linux (GNU)
- Running in-place edits on symlinked config files — can break the link

---

### 🔹 `find`

**What**
→ Locate files by name, type, size, age, or permissions

**Why**
→ Essential for auditing, cleanup, and building targeted pipelines

**When to Use**
- Find logs older than N days for rotation/deletion
- Locate large files eating disk
- Find all files with specific extensions across a project

**Core Usage**
```bash
find /var/log -name "*.log" -mtime +7       # logs older than 7 days
find . -type f -size +100M                  # files > 100MB
find /etc -name "*.conf" -readable          # readable config files
find . -name "*.py" | xargs grep "TODO"    # search inside found files
find /tmp -mmin +60 -delete                 # delete files older than 60 min
```

**Pro Tips**
- `-exec` runs a command per result; `xargs` is faster for bulk ops
- `-maxdepth 2` prevents runaway recursion on large trees
- Combine `-type f -name` for precise targeting; avoid `-name "*"` alone

---

### 🔹 `xargs`

**What**
→ Convert stdin into arguments for another command

**Why**
→ Bridges pipelines — makes commands that don't read stdin work in chains

**When to Use**
- Delete/move a list of files from `find` output
- Run the same command on many inputs in parallel
- Pass grep results into another tool

**Core Usage**
```bash
find . -name "*.log" | xargs rm -f              # delete found files
cat hosts.txt | xargs -I {} ping -c1 {}         # ping each host
find . -name "*.py" | xargs wc -l               # count lines in all .py files
cat ids.txt | xargs -P 4 -I {} curl http://api/{} # parallel requests
```

**Pro Tips**
- `-P N` runs N processes in parallel — great for I/O-bound tasks
- `-I {}` defines a placeholder for each input item
- Always quote `{}` when input may contain spaces

---

### 🔹 `ps` / `top` / `htop`

**What**
→ Inspect running processes, CPU, and memory usage

**Why**
→ First stop when a host is slow, a service is dead, or resources are maxed

**When to Use**
- Service not responding — is it running?
- CPU/memory spike on a node
- Find the PID of a process to kill or trace

**Core Usage**
```bash
ps aux --sort=-%cpu | head -10        # top CPU-consuming processes
ps aux --sort=-%mem | head -10        # top memory consumers
ps -ef | grep nginx                   # find nginx PID
kill -9 <pid>                         # force kill
htop                                  # interactive, color-coded view
```

**Pro Tips**
- `ps aux` is the universal one-liner; `htop` is better for interactive use
- Use `pgrep nginx` instead of `ps | grep | awk` — cleaner
- `kill -15` (SIGTERM) before `kill -9` — let the process clean up first

---

### 🔹 `curl`

**What**
→ Make HTTP requests from the terminal

**Why**
→ Fastest way to test APIs, health endpoints, and auth flows — no browser needed

**When to Use**
- Verify service is up (`/health`, `/ready`)
- Test auth tokens and headers
- Debug webhooks or API responses in CI

**Core Usage**
```bash
curl -i http://localhost:3000/health                      # include response headers
curl -s -o /dev/null -w "%{http_code}" http://url         # just the status code
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"val"}' http://api/endpoint               # POST with JSON body
curl -H "Authorization: Bearer $TOKEN" http://api/me      # auth header
curl -L -O https://example.com/file.tar.gz                # follow redirects, download
```

**Pro Tips**
- `-s` silences progress output — essential in scripts
- `-w` + format string = extract specific response metadata
- Pipe to `jq` for readable JSON: `curl -s http://api/data | jq`

---

### 🔹 `tail` / `less`

**What**
→ `tail` streams the end of a file; `less` lets you scroll through it

**Why**
→ Log watching and post-incident review without loading entire files into memory

**When to Use**
- Live log monitoring during deployments
- Reading large log files without `cat`
- Checking the last N lines of a rotating log

**Core Usage**
```bash
tail -f app.log                        # follow live output
tail -f app.log | grep ERROR           # live filtered stream
tail -n 100 app.log                    # last 100 lines
less +F app.log                        # less with follow mode (Ctrl+C to scroll)
less -S app.log                        # no line wrapping
```

**Pro Tips**
- `less +F` is underrated — you can pause, search, then resume following
- `tail -f` multiple files at once: `tail -f app.log error.log`
- Use `multitail` for side-by-side log streams

---

### 🔹 `ssh`

**What**
→ Securely connect to remote hosts; tunnel ports; proxy commands

**Why**
→ The foundation of remote ops — debugging, deployment, and emergency access

**When to Use**
- Direct server access for debugging
- Port forwarding to access internal services
- Running remote commands non-interactively in scripts

**Core Usage**
```bash
ssh user@host                                   # basic login
ssh -i ~/.ssh/key.pem user@host                 # specify key
ssh -L 5432:db-host:5432 user@bastion           # local port forward
ssh -N -D 1080 user@host                        # SOCKS proxy
ssh user@host "df -h && free -h"               # run remote commands
```

**Pro Tips**
- Use `~/.ssh/config` for aliases — avoids repeating IPs/keys
- `ssh -A` forwards your agent (useful for git over SSH on remote hosts)
- `ssh -o StrictHostKeyChecking=no` in automation (use with caution)

---

### 🔹 `tar`

**What**
→ Archive and compress/decompress files

**Why**
→ Standard format for backups, artifact packaging, and log archival

**Core Usage**
```bash
tar -czf archive.tar.gz /path/to/dir      # create compressed archive
tar -xzf archive.tar.gz                   # extract
tar -xzf archive.tar.gz -C /target/dir   # extract to specific path
tar -tzf archive.tar.gz                   # list contents without extracting
```

**Pro Tips**
- `z` = gzip, `j` = bzip2, `J` = xz — match to your size/speed tradeoff
- Always verify archives with `-t` before deleting originals
- Pipe directly: `tar -czf - /path | ssh user@host "tar -xzf - -C /dst"`

---

## 🔥 Linux Power Combos

```bash
# Find top 10 largest files on disk
find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10

# Kill all processes matching a name
pgrep -f "node server" | xargs kill -9

# Live error rate from log
tail -f app.log | grep --line-buffered ERROR | pv -l -r > /dev/null

# Replace env var in all config files
find ./config -name "*.yaml" | xargs sed -i 's/DB_HOST=localhost/DB_HOST=prod-db/g'

# Check which process owns a port
lsof -i :8080 | awk 'NR>1 {print $1, $2}'
```

---

## ⚡ POWER MOVES

---

### 🔹 `jq`

**What**
→ JSON processor for the terminal — parse, filter, transform

**Why**
→ Kubernetes, Docker, and APIs all speak JSON. `jq` makes it human-readable and scriptable.

**Core Usage**
```bash
cat data.json | jq '.'                          # pretty print
kubectl get pod -o json | jq '.status.phase'    # extract field
jq '.[] | select(.status == "failed")' data.json  # filter array
jq -r '.items[].metadata.name' pods.json        # raw output (no quotes)
curl -s http://api/data | jq '.data | length'   # count items
```

**Pro Tips**
- `-r` strips quotes from output — essential when piping into other commands
- Use `@base64`, `@csv`, `@sh` for format conversion inside jq
- `jq 'keys'` instantly shows the schema of an unknown JSON blob

---

### 🔹 `watch`

**What**
→ Re-runs a command every N seconds and shows the diff

**Why**
→ Passive monitoring during rollouts, migrations, and scaling events

**Core Usage**
```bash
watch kubectl get pods                    # default: every 2s
watch -n 5 df -h                         # every 5 seconds
watch -d kubectl get pods                # highlight changes
watch "docker stats --no-stream"         # snapshot container stats
```

**Pro Tips**
- `-d` highlights what changed between refreshes — crucial for rollouts
- Not available everywhere; `while true; do clear; cmd; sleep 2; done` as fallback

---

## 🐳 DOCKER

---

### 🔹 Docker Core Workflow

**What**
→ Build, run, debug, and clean up containers

**Why**
→ Containers are the unit of deployment — knowing these cold saves hours in incidents

**When to Use**
- Spin up services locally
- Debug a failing container
- Build and tag images for CI/CD

**Core Usage**
```bash
docker build -t app:1.0 .                        # build image
docker run -d -p 3000:3000 --name api app:1.0    # run detached
docker exec -it api sh                           # shell into running container
docker logs -f api                               # follow logs
docker logs --tail=100 api                       # last 100 lines
docker inspect api | jq '.[0].NetworkSettings'  # inspect networking
docker stats                                     # live resource usage
docker system prune -af                          # full cleanup
```

**Pro Tips**
- Always name containers with `--name` — makes logs/exec much faster
- `docker exec` into a crashed container? Use `docker run --entrypoint sh image`
- `docker cp` copies files in/out of containers without exec

**Common Mistakes**
- Using `latest` tag in production — always pin versions
- Not setting resource limits (`--memory`, `--cpus`) on long-running containers

---

### 🔹 Docker Compose

**What**
→ Multi-container orchestration via declarative YAML

**Core Usage**
```bash
docker compose up -d                    # start all services detached
docker compose down -v                  # stop and remove volumes
docker compose logs -f service-name    # follow specific service
docker compose exec service sh         # shell into service
docker compose ps                      # service status
docker compose build --no-cache        # force rebuild
```

**Pro Tips**
- `--no-cache` prevents stale layer issues during debugging
- Override values without editing YAML: `docker compose -f docker-compose.yml -f override.yml up`

---

## 🔥 Docker Power Combos

```bash
# Shell into the latest running container
docker exec -it $(docker ps -q | head -1) sh

# Clean up all stopped containers + dangling images
docker rm $(docker ps -aq -f status=exited) && docker image prune -f

# Copy file from container to host
docker cp api:/app/config.json ./config.json

# Build and immediately run for testing
docker build -t test . && docker run --rm test
```

---

## ☸️ KUBERNETES

---

### 🔹 Context Safety ⭐

**What**
→ Switch between clusters safely — dev, staging, prod

**Why**
→ Wrong context = production accident. This is the highest-leverage habit in k8s.

**Core Usage**
```bash
kubectl config get-contexts                      # list all contexts
kubectl config current-context                   # what am I connected to?
kubectl config use-context dev-cluster           # switch context
kubectl config use-context prod-cluster          # switch to prod (be careful)
```

**Pro Tips**
- Use `kubectx` and `kubens` tools — much faster context/namespace switching
- Add context name to your shell prompt (`kube-ps1`) to prevent mistakes
- Never run `apply` or `delete` without verifying context first

---

### 🔹 Pods & Debugging (Most Used)

**What**
→ Inspect, log, and shell into pods — the core debugging loop

**Core Usage**
```bash
kubectl get pods -A                                  # all namespaces
kubectl get pods -n backend -w                       # watch for changes
kubectl describe pod <pod> -n backend                # full event history
kubectl logs <pod> -c <container> --tail=100         # specific container logs
kubectl logs <pod> --previous                        # logs from crashed container
kubectl exec -it <pod> -n backend -- sh              # shell into pod
kubectl cp <pod>:/path/to/file ./local-file          # copy file out
```

**Pro Tips**
- `--previous` is critical — gets logs from the crashed instance before restart
- `kubectl describe` shows Events at the bottom — always check them first
- Use `kubectl debug` (1.23+) to attach ephemeral debug containers

---

### 🔹 Deployments

**Core Usage**
```bash
kubectl get deploy -n backend
kubectl rollout status deploy/api -n backend         # watch rollout live
kubectl rollout history deploy/api                   # revision history
kubectl rollout undo deploy/api                      # rollback one revision
kubectl rollout restart deploy/api                   # rolling restart
kubectl scale deploy/api --replicas=5               # manual scale
kubectl set image deploy/api api=app:2.0            # update image
```

**Pro Tips**
- Always confirm `rollout status` after `apply` — don't just fire and forget
- `rollout undo` is your break-glass rollback; `rollout history` tells you what you're undoing
- Prefer `kubectl apply -f` over imperative commands for auditability

---

### 🔹 Services & Networking

**Core Usage**
```bash
kubectl get svc -n backend
kubectl describe svc api -n backend
kubectl port-forward svc/api 8080:80 -n backend     # local access to service
kubectl port-forward pod/<pod> 9090:9090            # direct pod tunnel
```

**Pro Tips**
- Port-forward is your best friend for debugging internal services locally
- Check `Endpoints` in `describe svc` — if empty, selector doesn't match any pods

---

### 🔹 YAML Operations

**Core Usage**
```bash
kubectl apply -f deployment.yaml                # apply (create or update)
kubectl delete -f deployment.yaml               # delete resources
kubectl diff -f deployment.yaml                 # preview changes before apply
kubectl apply -f ./manifests/                  # apply entire directory
kubectl get deploy api -o yaml > backup.yaml    # export current state
```

**Pro Tips**
- Always `kubectl diff` before `kubectl apply` in production
- Use `--dry-run=client` or `--dry-run=server` to validate without applying
- `--server-side` apply is safer for large manifests with field managers

---

### 🔹 Events, Resources & Cluster Health

**Core Usage**
```bash
kubectl get events --sort-by=.metadata.creationTimestamp -n backend
kubectl top pods -n backend                          # CPU/memory by pod
kubectl top nodes                                    # node resource usage
kubectl get all -n backend                           # all resource types
kubectl api-resources                                # list all resource types
kubectl explain deployment.spec.template.spec        # inline docs
```

**Pro Tips**
- Events are time-sorted, last = most recent. Start here during an incident.
- `kubectl top` requires metrics-server — verify it's installed if it errors
- `kubectl get all` misses CRDs — use `-A --all-namespaces` for full picture

---

## 🔥 Kubernetes Power Combos

```bash
# Get all failing pods across all namespaces
kubectl get pods -A | grep -v Running | grep -v Completed

# Force delete a stuck terminating pod
kubectl delete pod <pod> --grace-period=0 --force -n backend

# Watch rollout in real time
kubectl rollout status deploy/api -n backend && kubectl get pods -n backend -w

# Describe all pods with errors (quick triage)
kubectl get pods -A | awk '$4 ~ /Error|CrashLoop|OOMKilled/ {print $1, $2}' \
  | xargs -n2 sh -c 'kubectl describe pod $2 -n $1'

# Port-forward to a pod by label (no name needed)
kubectl port-forward $(kubectl get pod -l app=api -o name | head -1) 8080:3000
```

---

## 🌐 NETWORKING

---

### 🔹 Quick Checks

```bash
ping service-name                       # basic reachability
traceroute google.com                   # hop-by-hop path
nc -zv localhost 5432                   # TCP port check (no data sent)
curl -sv telnet://host:port             # curl-based TCP check
lsof -i :3000                           # what's on port 3000?
netstat -tulpn | grep LISTEN            # all listening ports
ss -tulpn                               # modern netstat replacement
```

**Pro Tips**
- `nc -zv` is the fastest port probe — use in container healthchecks
- `ss` is faster and richer than `netstat` on modern Linux
- Inside k8s pods, DNS issues are common: `nslookup service-name.namespace.svc.cluster.local`

---

## 🧠 Mental Models

| Situation | Start With |
|---|---|
| Service is down | `kubectl describe pod` → check Events |
| Container keeps restarting | `kubectl logs --previous` |
| Node is slow | `kubectl top nodes` → `ssh` → `htop` |
| Can't connect to service | `nc -zv host port` → `kubectl describe svc` → check Endpoints |
| Disk full | `df -h` → `du -sh *` → `find / -size +100M` |
| Unknown error in logs | `grep -E "ERROR|WARN|FATAL"` → `jq` if JSON |
| Config not loading | `printenv | grep VAR` → `kubectl exec -- env` |
| Rollout broken | `kubectl rollout undo` → `kubectl rollout history` |

---

