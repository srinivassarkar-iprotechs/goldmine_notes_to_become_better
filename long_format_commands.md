# ⚔️ DevOps Production Survival Sheet
### *Battle-Tested Operational Playbook — Master This, Handle Anything*

> **How to use this sheet:** Under pressure, go straight to the category. Read the "When" column first. If you're debugging a 2 AM outage, start with **Incident Response**, then cross-reference **Linux**, **Kubernetes**, or **Docker** as needed. Every command here has been chosen because it solves real production problems — not because it looks good in a tutorial.

---

## Table of Contents
1. [Linux & System Administration](#1-linux--system-administration)
2. [Docker](#2-docker)
3. [Kubernetes](#3-kubernetes)
4. [Networking](#4-networking)
5. [CI/CD](#5-cicd)
6. [Monitoring & Observability](#6-monitoring--observability)
7. [Debugging & Log Analysis](#7-debugging--log-analysis)
8. [Cloud (AWS-focused, with GCP/Azure notes)](#8-cloud-aws-focused)
9. [Git](#9-git)
10. [Security](#10-security)
11. [Incident Response Workflows](#11-incident-response-workflows)

---

## 1. Linux & System Administration

> **Guiding principle:** If you can't read a machine's health in under 60 seconds, you're not ready for production on-call.

---

### 1.1 Process & CPU

```bash
top -c
```
| Field | Detail |
|-------|--------|
| **What** | Real-time view of processes, CPU, and memory |
| **Why** | Fastest way to see if a process is eating CPU/RAM |
| **When** | App is slow, host is unresponsive, CPU spike alert fires |
| **Example** | `top -c` → Press `P` to sort by CPU, `M` to sort by memory |
| **Pitfall** | `top` shows snapshot averages — use `htop` for a cleaner view, or `pidstat` for history |

```bash
htop
```
| Field | Detail |
|-------|--------|
| **What** | Interactive process viewer with color coding |
| **Why** | Easier to read than `top`; supports mouse, filtering, tree view |
| **When** | Investigating which process is consuming resources |
| **Example** | `htop -u appuser` — show only processes for a specific user |
| **Pitfall** | May not be installed by default. On minimal containers: use `top` or `ps` |

```bash
ps aux --sort=-%cpu | head -20
```
| Field | Detail |
|-------|--------|
| **What** | List top 20 CPU-consuming processes |
| **Why** | Non-interactive; pipe-friendly; scriptable |
| **When** | Automated alerting scripts, SSH sessions without terminal control |
| **Example** | `ps aux --sort=-%mem | head -10` — top memory consumers |
| **Pitfall** | `%cpu` is cumulative since start, not current. For instantaneous: use `top` |

```bash
pidstat -u -p <PID> 1 5
```
| Field | Detail |
|-------|--------|
| **What** | Per-process CPU stats sampled every 1s for 5 intervals |
| **Why** | Tracks CPU trend over time for a specific process |
| **When** | You have a suspect PID and need historical CPU data |
| **Example** | `pidstat -u -p 1234 1 10` |
| **Pitfall** | Requires `sysstat` package. Install: `apt install sysstat` |

---

### 1.2 Memory

```bash
free -h
```
| Field | Detail |
|-------|--------|
| **What** | Shows total, used, free, and cached memory in human-readable format |
| **Why** | Quick memory health check |
| **When** | OOM alerts, app crashes, slow performance |
| **Example** | `watch -n 2 free -h` — refresh every 2 seconds |
| **Pitfall** | "available" column is what matters, not "free". Linux caches aggressively — low "free" is normal |

```bash
vmstat 1 10
```
| Field | Detail |
|-------|--------|
| **What** | Virtual memory stats: CPU, swap, I/O, memory per second |
| **Why** | Catches swap usage and I/O wait — two silent killers |
| **When** | App slow but CPU looks normal; OOM suspected; disk I/O alert |
| **Example** | Watch `si`/`so` columns — non-zero swap in/out = memory pressure |
| **Pitfall** | First row is system average since boot — ignore it, watch rows 2+ |

```bash
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapUsed"
```
| Field | Detail |
|-------|--------|
| **What** | Exact kernel memory breakdown |
| **Why** | More precise than `free`; useful in scripts |
| **When** | Investigating OOM killer events, memory limits |
| **Example** | Check `SwapUsed` — if non-zero in production, you have a problem |
| **Pitfall** | Values are in kB — always convert for readability |

---

### 1.3 Disk & I/O

```bash
df -h
```
| Field | Detail |
|-------|--------|
| **What** | Disk usage by mount point |
| **Why** | Catch disk-full before logs stop writing or databases crash |
| **When** | Services failing to write, log pipelines breaking, deployments failing |
| **Example** | `df -h /var/log /tmp /` — check the mounts that fill fastest |
| **Pitfall** | A full `/tmp` or `/var/log` will kill services even if `/` has space |

```bash
du -sh /var/log/* | sort -rh | head -20
```
| Field | Detail |
|-------|--------|
| **What** | Which directories/files are consuming the most space |
| **Why** | `df` tells you disk is full; `du` tells you *why* |
| **When** | After `df` shows high usage |
| **Example** | `du -sh /var/log/app/* | sort -rh` — find the fattest log files |
| **Pitfall** | Can be slow on large filesystems. Use `ncdu` for interactive navigation |

```bash
iostat -xz 1 5
```
| Field | Detail |
|-------|--------|
| **What** | Disk I/O stats: utilization, await time, throughput per device |
| **Why** | Diagnose disk bottlenecks — high `%util` or `await` = disk is the problem |
| **When** | Slow database queries, high I/O wait in `vmstat`, slow file operations |
| **Example** | `await` > 20ms on SSD = problem. On HDD > 100ms = serious problem |
| **Pitfall** | `%util` near 100% means saturation, but SSDs can saturate at 100% and still serve fast — check `await` |

```bash
lsof +D /var/log/app/
```
| Field | Detail |
|-------|--------|
| **What** | List all open file handles in a directory |
| **Why** | Diagnose "deleted but still consuming space" files and file handle leaks |
| **When** | Disk shows full but `du` shows it shouldn't be; log rotation failures |
| **Example** | `lsof | grep deleted` — files deleted but still open (consuming inodes) |
| **Pitfall** | A process holding a deleted file open keeps consuming disk space until the process restarts |

---

### 1.4 System Vitals (The 60-Second Health Check)

```bash
# Run this sequence when you land on a sick host:
uptime                          # Load average vs CPU count
dmesg -T | tail -50             # Kernel messages — OOM kills, hardware errors
journalctl -xe --since "1 hour ago"  # Systemd unit errors
ss -tulnp                       # Open ports and listening services
netstat -s | grep -i error      # Network stack errors
```

```bash
uptime
```
| Field | Detail |
|-------|--------|
| **What** | Load average over 1, 5, 15 minutes + uptime |
| **Why** | Load > number of CPU cores = system is overwhelmed |
| **When** | First command when landing on a sick host |
| **Example** | 8-core host with load `24.5` = 3x overloaded |
| **Pitfall** | Load includes I/O wait, not just CPU. A host with high I/O will show high load even with low CPU |

```bash
dmesg -T | tail -50
```
| Field | Detail |
|-------|--------|
| **What** | Kernel ring buffer — hardware errors, OOM kills, network issues |
| **Why** | OOM killer, segfaults, and disk errors show up here before anywhere else |
| **When** | Process mysteriously dies, host has hardware issues, unexplained crashes |
| **Example** | Look for: `Out of memory: Kill process`, `I/O error`, `EXT4-fs error` |
| **Pitfall** | `-T` flag adds human-readable timestamps — always use it |

```bash
journalctl -u <service> -n 100 --no-pager
```
| Field | Detail |
|-------|--------|
| **What** | Last 100 log lines for a systemd service |
| **Why** | Fastest way to see why a service is down |
| **When** | Service failed to start, restart loops, silent failures |
| **Example** | `journalctl -u nginx -f` — follow nginx logs live |
| **Pitfall** | `--no-pager` is critical in scripts. Without it, journalctl blocks on interactive pager |

```bash
ss -tulnp
```
| Field | Detail |
|-------|--------|
| **What** | Show all TCP/UDP sockets, listening ports, and owning PIDs |
| **Why** | Verify services are actually listening on expected ports |
| **When** | Service says "up" but connections fail; port conflicts; deployment verification |
| **Example** | `ss -tulnp | grep 8080` — is anything listening on 8080? |
| **Pitfall** | `netstat` is deprecated — use `ss`. On old systems without `ss`, fall back to `netstat -tulnp` |

---

### 1.5 File & Text Operations

```bash
tail -f /var/log/app/app.log
```
| Field | Detail |
|-------|--------|
| **What** | Follow a log file in real time |
| **Why** | Watch application behavior live during incidents or deployments |
| **When** | After a deployment, during active incidents, testing changes |
| **Example** | `tail -f app.log | grep -i error` — filter for errors only |
| **Pitfall** | If log rotates, `tail -f` stops following. Use `tail -F` (capital F) to follow across rotations |

```bash
grep -rn "ERROR\|FATAL\|Exception" /var/log/app/ --include="*.log"
```
| Field | Detail |
|-------|--------|
| **What** | Recursively search for error patterns in log files |
| **Why** | Find errors across multiple log files quickly |
| **When** | Post-incident analysis, searching for root cause across services |
| **Example** | `grep -rn "NullPointerException" /var/log/ -l` — list files containing NPEs |
| **Pitfall** | Without `--include`, grep scans binary files and slows drastically |

```bash
awk '{print $NF}' access.log | sort | uniq -c | sort -rn | head -20
```
| Field | Detail |
|-------|--------|
| **What** | Count and rank the last field (e.g., status codes or endpoints) |
| **Why** | Find the most frequent log patterns — top error codes, top IPs, top endpoints |
| **When** | Traffic analysis, identifying abuse, finding hot paths |
| **Example** | `awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn` — HTTP status code distribution |
| **Pitfall** | Know your log format's field positions. Nginx combined = field 9 for status code |

```bash
find /var/log -name "*.log" -mtime -1 -size +100M
```
| Field | Detail |
|-------|--------|
| **What** | Find log files modified in last 24h that are over 100MB |
| **Why** | Catch runaway logging before disk fills up |
| **When** | Disk usage alert, log pipeline issues |
| **Example** | `find /tmp -mtime +7 -delete` — clean up files older than 7 days (be careful!) |
| **Pitfall** | `-mtime -1` means "less than 1 day old" — the sign matters. Test with `-print` before `-delete` |

---

## 2. Docker

> **Guiding principle:** If you can't get into a container and read its logs within 30 seconds, you're debugging blind.

---

### 2.1 Container Inspection & Access

```bash
docker ps -a
```
| Field | Detail |
|-------|--------|
| **What** | List all containers including stopped ones |
| **Why** | See what's running, what crashed, and exit codes |
| **When** | First thing to run when a service is down |
| **Example** | `docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` |
| **Pitfall** | Without `-a`, stopped containers are invisible. Always use `-a` during incidents |

```bash
docker logs <container> --tail 200 -f
```
| Field | Detail |
|-------|--------|
| **What** | Stream last 200 lines and follow container logs |
| **Why** | See what the app is actually saying |
| **When** | Container is crashing, behaving unexpectedly, or you're deploying |
| **Example** | `docker logs api-service --tail 500 --since 30m` — last 30 minutes |
| **Pitfall** | Logs disappear when container is removed. Always ship logs to a centralized system in production |

```bash
docker exec -it <container> /bin/sh
```
| Field | Detail |
|-------|--------|
| **What** | Open an interactive shell inside a running container |
| **Why** | Inspect the runtime environment, check files, test connectivity |
| **When** | Debugging network issues, checking env vars, inspecting config files |
| **Example** | `docker exec -it nginx-container nginx -t` — test nginx config without full shell |
| **Pitfall** | Minimal images (alpine, distroless) may not have bash. Try `/bin/sh` first, then `busybox sh` |

```bash
docker inspect <container> | jq '.[0].NetworkSettings.Networks'
```
| Field | Detail |
|-------|--------|
| **What** | Full JSON metadata for a container — networks, mounts, env vars, health |
| **Why** | Ground truth for container configuration |
| **When** | Network issues, verifying mounts, checking health check config |
| **Example** | `docker inspect <id> | jq '.[0].Config.Env'` — read environment variables |
| **Pitfall** | Env vars with secrets appear in plain text here — treat `docker inspect` output as sensitive |

```bash
docker stats --no-stream
```
| Field | Detail |
|-------|--------|
| **What** | CPU, memory, network I/O, and block I/O for all containers |
| **Why** | Resource health check across all containers at once |
| **When** | Host is slow, need to identify which container is the culprit |
| **Example** | `docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"` |
| **Pitfall** | `--no-stream` gives a single snapshot. Without it, it refreshes continuously |

---

### 2.2 Image Management

```bash
docker images | sort -k7 -rh
```
| Field | Detail |
|-------|--------|
| **What** | List images sorted by size |
| **Why** | Find bloated images consuming disk |
| **When** | Disk pressure on Docker host, auditing image hygiene |
| **Example** | `docker images --filter "dangling=true"` — find untagged orphan images |
| **Pitfall** | Dangling images accumulate silently. Set up a cron job: `docker image prune -f` |

```bash
docker system prune -af --volumes
```
| Field | Detail |
|-------|--------|
| **What** | Remove all stopped containers, unused networks, dangling images, and volumes |
| **Why** | Reclaim disk space fast |
| **When** | Docker host disk is filling up |
| **Example** | Run without `--volumes` first if you're not sure about volume data |
| **Pitfall** | **DESTRUCTIVE**. Do not run on hosts with stateful containers unless you know what's in the volumes |

---

### 2.3 Docker Networking & Debugging

```bash
docker network ls
docker network inspect <network-name>
```
| Field | Detail |
|-------|--------|
| **What** | List networks and inspect their configuration and connected containers |
| **Why** | Diagnose container-to-container communication failures |
| **When** | Container can't reach another container by name |
| **Example** | Verify both containers are on the same network |
| **Pitfall** | Containers on different networks can't communicate by default — this is a feature, not a bug |

```bash
docker run --rm --network container:<id> nicolaka/netshoot
```
| Field | Detail |
|-------|--------|
| **What** | Attach a network debugging toolkit container to an existing container's network namespace |
| **Why** | Debug network issues in minimal containers that lack tools |
| **When** | Need to run `curl`, `dig`, `tcpdump` inside a container's network context |
| **Example** | Then run `curl http://other-service:8080/health` from inside |
| **Pitfall** | The target container must be running. Distroless containers are common — always have this trick ready |

---

## 3. Kubernetes

> **Guiding principle:** In Kubernetes, the pod is disposable. Diagnose fast, understand the root cause, fix the source — don't patch the pod.

---

### 3.1 The Kubernetes First-Responder Sequence

When a service is down in Kubernetes, run these in order:

```bash
# Step 1: What's broken?
kubectl get pods -n <namespace> -o wide

# Step 2: Why is it broken?
kubectl describe pod <pod-name> -n <namespace>

# Step 3: What did the app say?
kubectl logs <pod-name> -n <namespace> --previous --tail=200

# Step 4: What's the deployment state?
kubectl rollout status deployment/<name> -n <namespace>

# Step 5: Is it a config or secret issue?
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp' | tail -20
```

---

### 3.2 Pod Operations

```bash
kubectl get pods -n <namespace> -o wide
```
| Field | Detail |
|-------|--------|
| **What** | List pods with status, node assignment, restarts, and IP |
| **Why** | See pod health at a glance |
| **When** | First command in any Kubernetes incident |
| **Example** | Watch for: `CrashLoopBackOff`, `OOMKilled`, `Pending`, `ImagePullBackOff` |
| **Pitfall** | "Running" doesn't mean "healthy" — a pod can be running but failing health checks |

```bash
kubectl describe pod <pod-name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Full pod metadata: events, resource limits, conditions, probes, mounts |
| **Why** | The Events section shows exactly why a pod failed to start |
| **When** | Pod won't start, is pending, or keeps restarting |
| **Example** | Look for: `FailedScheduling` (no nodes), `ErrImagePull`, `Insufficient memory` |
| **Pitfall** | Events expire after ~1 hour. If the pod has been failing for a while, events may be gone |

```bash
kubectl logs <pod-name> -n <namespace> --previous
```
| Field | Detail |
|-------|--------|
| **What** | Logs from the previous (crashed) container instance |
| **Why** | After a crash loop, the current instance may not have useful logs yet |
| **When** | CrashLoopBackOff pods, OOMKilled pods |
| **Example** | `kubectl logs <pod> -c <container> --previous` — for multi-container pods |
| **Pitfall** | `--previous` only keeps one previous instance. If it's crashed 10 times, earlier logs are gone — ship to a log aggregator |

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```
| Field | Detail |
|-------|--------|
| **What** | Interactive shell inside a running pod |
| **Why** | Inspect environment, test connectivity, check config files at runtime |
| **When** | Debugging app behavior, testing service discovery, verifying secrets are mounted |
| **Example** | `kubectl exec -it <pod> -- env | grep DB_HOST` — check environment variables |
| **Pitfall** | Pod must be running. For crashed pods, use an ephemeral debug container (see below) |

```bash
kubectl debug -it <pod-name> --image=nicolaka/netshoot --copy-to=debug-pod -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Launch an ephemeral debug sidecar into a copy of a pod |
| **Why** | Debug distroless/minimal containers that have no shell |
| **When** | `kubectl exec` fails because the container has no shell or tools |
| **Example** | Use to run `curl`, `tcpdump`, `dig` in the same network namespace |
| **Pitfall** | Creates a copy of the pod — remember to delete it after: `kubectl delete pod debug-pod` |

---

### 3.3 Deployments & Rollouts

```bash
kubectl rollout status deployment/<name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Block and watch a deployment rollout until complete or failed |
| **Why** | Know definitively when a deployment succeeds or fails |
| **When** | After every `kubectl apply` or `kubectl set image` |
| **Example** | Use in CI/CD pipelines to gate the next step on successful rollout |
| **Pitfall** | If a deployment stalls (not enough nodes, image pull fails), this command hangs — set a timeout |

```bash
kubectl rollout undo deployment/<name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Roll back to the previous deployment revision |
| **Why** | Fastest way to recover from a bad deployment |
| **When** | New deployment is causing errors — roll back immediately, investigate later |
| **Example** | `kubectl rollout undo deployment/api --to-revision=3` — roll back to specific revision |
| **Pitfall** | This only rolls back the pod template. If the bad deployment ran a DB migration, rollback won't undo the schema change |

```bash
kubectl rollout history deployment/<name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | List all deployment revisions |
| **Why** | Identify which revision to roll back to |
| **When** | Before performing `rollout undo` |
| **Example** | `kubectl rollout history deployment/api --revision=3` — see what changed in revision 3 |
| **Pitfall** | History is limited by `revisionHistoryLimit` (default: 10). Old revisions are pruned |

```bash
kubectl set image deployment/<name> <container>=<image>:<tag> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Update the container image for a deployment |
| **Why** | Quick hotfix deployment without editing YAML |
| **When** | Emergency hotfix, fast image update during an incident |
| **Example** | `kubectl set image deployment/api api=myrepo/api:hotfix-1.2.3 -n prod` |
| **Pitfall** | This changes the live spec but not your source YAML — always sync your manifests afterward |

---

### 3.4 Services, ConfigMaps & Secrets

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | List services and inspect their selectors, endpoints, and ports |
| **Why** | Verify traffic is actually routed to the right pods |
| **When** | Requests not reaching pods, load balancer misconfiguration |
| **Example** | Check `Endpoints:` section — if empty, the selector doesn't match any running pod |
| **Pitfall** | A service with no endpoints is a silent failure — traffic goes nowhere |

```bash
kubectl get endpoints <service-name> -n <namespace>
```
| Field | Detail |
|-------|--------|
| **What** | Show the actual pod IPs backing a service |
| **Why** | Verify that service routing is wired to live pods |
| **When** | Service exists but requests fail |
| **Example** | `<none>` under ENDPOINTS = label mismatch between service and pods |
| **Pitfall** | Endpoints update asynchronously — give 10-30s after pod comes up |

```bash
kubectl get configmap <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d
```
| Field | Detail |
|-------|--------|
| **What** | Read ConfigMap or decode a specific secret value |
| **Why** | Verify config values are correct at the cluster level |
| **When** | App is using wrong config, environment variables unexpected |
| **Example** | `kubectl get secret db-creds -n prod -o jsonpath='{.data.password}' \| base64 -d` |
| **Pitfall** | Secrets are base64 encoded, not encrypted at rest by default — ensure etcd encryption is enabled in your cluster |

---

### 3.5 Resource Usage & Node Health

```bash
kubectl top pods -n <namespace> --sort-by=cpu
kubectl top nodes
```
| Field | Detail |
|-------|--------|
| **What** | Real-time CPU and memory usage for pods and nodes |
| **Why** | Identify resource hogs and node pressure |
| **When** | Performance degradation, OOMKilled pods, slow nodes |
| **Example** | `kubectl top pods --all-namespaces --sort-by=memory` |
| **Pitfall** | Requires metrics-server installed. If not installed: `kubectl top` will fail with "metrics API not available" |

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
```
| Field | Detail |
|-------|--------|
| **What** | Node health and capacity; `describe` shows pressure conditions and taints |
| **Why** | Understand why pods can't be scheduled |
| **When** | Pods stuck in `Pending`, nodes showing `NotReady` |
| **Example** | Look for: `MemoryPressure`, `DiskPressure`, `PIDPressure` in node conditions |
| **Pitfall** | A node can be `Ready` but have taints preventing pod scheduling — check `Taints:` section |

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -30
```
| Field | Detail |
|-------|--------|
| **What** | Cluster events sorted by time — scheduling failures, image pulls, health check failures |
| **Why** | Often the most informative log when `describe pod` isn't enough |
| **When** | Pods failing for unclear reasons |
| **Example** | `kubectl get events --field-selector reason=OOMKilling` — filter to OOM events |
| **Pitfall** | Events are namespace-scoped. Run across all namespaces: `kubectl get events --all-namespaces` |

---

### 3.6 Namespace & RBAC

```bash
kubectl auth can-i create pods --as=system:serviceaccount:<namespace>:<sa-name>
```
| Field | Detail |
|-------|--------|
| **What** | Check if a service account has permission to perform an action |
| **Why** | Diagnose RBAC failures without trial and error |
| **When** | Pod fails with "forbidden" errors, deployment controller can't create pods |
| **Example** | `kubectl auth can-i get secrets -n prod --as=system:serviceaccount:prod:my-app` |
| **Pitfall** | RBAC errors in pod logs often appear as vague HTTP 403s — always check service account permissions |

---

## 4. Networking

> **Guiding principle:** Network problems lie. The error message is rarely where the problem actually is. Trace the path.

---

### 4.1 Connectivity Testing

```bash
curl -v -o /dev/null -w "%{http_code} %{time_total}s" https://api.example.com/health
```
| Field | Detail |
|-------|--------|
| **What** | Test HTTP endpoint, show response code and total latency |
| **Why** | Verify endpoint is reachable, get timing breakdown |
| **When** | Service health checks, testing after deployments |
| **Example** | `-w` flag formats output: `200 0.342s` means healthy and fast |
| **Pitfall** | Without `-v`, SSL errors are silent. Always use `-v` during debugging |

```bash
curl -v --connect-timeout 5 --max-time 10 http://service:port/endpoint
```
| Field | Detail |
|-------|--------|
| **What** | HTTP request with explicit connection and total timeouts |
| **Why** | Prevent hung curl commands during debugging |
| **When** | Testing internal service endpoints, verifying timeouts are configured correctly |
| **Example** | Connection timeout = can't reach the host. Max-time = reached but slow |
| **Pitfall** | Default curl has no timeout — it will hang forever on a broken connection |

```bash
traceroute -n <hostname>
mtr --report --report-cycles 10 <hostname>
```
| Field | Detail |
|-------|--------|
| **What** | Trace network path hop by hop; `mtr` combines traceroute + ping with statistics |
| **Why** | Find where in the network path packets are dropping or latency is introduced |
| **When** | Intermittent connectivity issues, high latency between services |
| **Example** | `mtr --report 8.8.8.8` — look for hops with >1% packet loss |
| **Pitfall** | Some routers block ICMP — a `***` hop doesn't always mean packet loss to that destination |

```bash
dig +trace @8.8.8.8 api.example.com
nslookup api.example.com
```
| Field | Detail |
|-------|--------|
| **What** | DNS resolution trace from root to authoritative nameserver |
| **Why** | Diagnose DNS failures — wrong record, propagation delay, caching issues |
| **When** | Service unreachable by hostname, recent DNS change, SSL cert mismatch |
| **Example** | `dig api.example.com A` — just get the A record |
| **Pitfall** | Your machine's resolver caches results. Add `+nocache` or test with a different DNS server (`@1.1.1.1`) |

```bash
nc -zv <host> <port>
```
| Field | Detail |
|-------|--------|
| **What** | Test TCP connectivity to a specific port without sending data |
| **Why** | Verify port is open and service is listening, without needing curl or a browser |
| **When** | Firewall testing, verifying service ports, database connectivity checks |
| **Example** | `nc -zv db.internal 5432` — is PostgreSQL reachable? |
| **Pitfall** | `nc` tests TCP only. Use `nc -zu` for UDP. A response doesn't mean auth will work |

```bash
ss -s
ss -tulnp | grep <port>
```
| Field | Detail |
|-------|--------|
| **What** | Socket statistics; filter to a specific port |
| **Why** | See what's actually listening; diagnose port conflicts and connection state |
| **When** | Service won't bind to a port, too many connections, connection exhaustion |
| **Example** | `ss -s` shows summary: TIME_WAIT accumulation = connection exhaustion risk |
| **Pitfall** | High `TIME_WAIT` count is normal under load but can exhaust ephemeral ports — tune `ip_local_port_range` and `tcp_tw_reuse` |

---

### 4.2 Packet Analysis

```bash
tcpdump -i eth0 -nn port 80 -w /tmp/capture.pcap
```
| Field | Detail |
|-------|--------|
| **What** | Capture all traffic on port 80, save to file |
| **Why** | See actual network traffic — what's really being sent and received |
| **When** | Mysterious connection failures, debugging protocols, SSL handshake issues |
| **Example** | `tcpdump -i any -nn host 10.0.1.5 and port 443` — filter by host and port |
| **Pitfall** | Generates large files fast. Always use `-w` + a size limit, or you'll fill the disk |

---

### 4.3 Load Balancer & Firewall

```bash
iptables -L -n -v --line-numbers
```
| Field | Detail |
|-------|--------|
| **What** | List all firewall rules with packet counts |
| **Why** | Find rules blocking traffic |
| **When** | Connectivity blocked, security group changes, Kubernetes CNI issues |
| **Example** | High packet count on a REJECT rule = legitimate traffic being blocked |
| **Pitfall** | Output is long. Pipe through `grep` to find relevant chains quickly |

---

## 5. CI/CD

> **Guiding principle:** A CI/CD pipeline is only as trustworthy as its failure modes. Know how to unblock a pipeline and how to safely deploy without one.

---

### 5.1 Pipeline Operations

```bash
# GitHub Actions — trigger a workflow manually
gh workflow run deploy.yml --ref main -f environment=production

# View recent run status
gh run list --workflow=deploy.yml --limit 10

# Watch a specific run
gh run watch <run-id>
```
| Field | Detail |
|-------|--------|
| **What** | GitHub CLI commands to manage and monitor workflows |
| **Why** | Manage deployments without leaving the terminal |
| **When** | Manual deploys, investigating failed pipelines, triggering hotfixes |
| **Pitfall** | Always confirm the `--ref` branch. Accidentally deploying from `main` to prod when you meant `hotfix` is a common mistake |

```bash
# Jenkins — trigger a job via API
curl -X POST https://jenkins.internal/job/<job-name>/build \
  --user "user:api_token" \
  --data-urlencode "json={}"
```
| Field | Detail |
|-------|--------|
| **What** | Trigger a Jenkins build via REST API |
| **Why** | Automate deployments, recover stuck builds |
| **When** | Pipeline needs to be triggered from scripts, webhooks failed |
| **Pitfall** | API token must have appropriate permissions. CSRF protection may require a crumb token first |

---

### 5.2 Container Registry

```bash
# Build and push with proper tagging strategy
docker build -t myrepo/service:${GIT_SHA} -t myrepo/service:latest .
docker push myrepo/service:${GIT_SHA}
docker push myrepo/service:latest
```
| Field | Detail |
|-------|--------|
| **What** | Build image tagged with git SHA and latest |
| **Why** | `latest` is convenient; git SHA tag makes deployments traceable and rollback-safe |
| **When** | Every production build |
| **Example** | Always tag with a specific version. Never deploy `:latest` to production |
| **Pitfall** | `:latest` is dangerous in prod — it changes silently. Pin to SHA or semantic version tags |

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myrepo/service:1.0.0 --push .
```
| Field | Detail |
|-------|--------|
| **What** | Build multi-architecture images |
| **Why** | AWS Graviton (ARM), Apple Silicon devs — cross-platform images are increasingly required |
| **When** | Mixed-architecture clusters, deploying to Graviton nodes |
| **Pitfall** | QEMU emulation is slow — use native build nodes per architecture for production pipelines |

---

### 5.3 Deployment Patterns

```bash
# Blue/Green: verify green is healthy before cutting traffic
kubectl apply -f deployment-green.yaml
kubectl rollout status deployment/service-green -n prod
# If healthy, switch the service selector:
kubectl patch service my-service -p '{"spec":{"selector":{"version":"green"}}}'
```

```bash
# Canary: send 10% of traffic to new version
# (via Argo Rollouts or Istio VirtualService weight)
kubectl apply -f rollout-canary.yaml
kubectl argo rollouts set weight my-rollout 10
kubectl argo rollouts promote my-rollout  # promote to 100% when stable
```

---

## 6. Monitoring & Observability

> **Guiding principle:** Observability is not just metrics. You need metrics (what), logs (why), and traces (where) to solve production problems reliably.

---

### 6.1 Prometheus & Metrics

```bash
# Query Prometheus HTTP API directly
curl -s 'http://prometheus:9090/api/v1/query' \
  --data-urlencode 'query=rate(http_requests_total{service="api"}[5m])' | jq .

# Check metric exists
curl -s 'http://prometheus:9090/api/v1/label/__name__/values' | jq '.data | map(select(startswith("http_")))'
```
| Field | Detail |
|-------|--------|
| **What** | Query Prometheus directly via HTTP API |
| **Why** | Debug dashboards, verify metrics are being collected, test PromQL queries |
| **When** | Alert is firing but you don't know why; metric appears missing |
| **Pitfall** | Prometheus has a query timeout. Complex queries on large datasets will time out — add label filters |

**Essential PromQL Patterns:**
```promql
# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage trend
container_memory_working_set_bytes{pod=~"api-.*"} / container_spec_memory_limit_bytes

# CPU throttling (critical - often missed)
rate(container_cpu_throttled_seconds_total[5m])
```

---

### 6.2 Alerting Triage

When an alert fires, immediately answer these 4 questions:

```
1. What is the blast radius? (one service, one region, entire platform?)
2. Is this trending worse or stabilizing?
3. What changed in the last 30 minutes? (deployment, config, traffic spike?)
4. Is there customer impact right now?
```

```bash
# Check recent deployments across namespaces
kubectl get events --all-namespaces --field-selector reason=Pulled --sort-by='.lastTimestamp' | tail -20

# Check if traffic changed (nginx example)
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | sort | uniq -c
```

---

### 6.3 Log Aggregation (ELK / Loki)

```bash
# Loki via logcli
logcli query '{app="api", namespace="prod"}' --since=1h --limit=500 | grep -i error

# Elasticsearch via curl
curl -s "http://es:9200/logs-*/_search" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"bool":{"must":[{"match":{"level":"ERROR"}},{"range":{"@timestamp":{"gte":"now-1h"}}}]}},"size":50}' | jq '.hits.hits[]._source'
```
| Field | Detail |
|-------|--------|
| **When** | Investigating errors across multiple services or instances |
| **Pitfall** | Elasticsearch queries without a time range on large clusters can be catastrophically slow |

---

## 7. Debugging & Log Analysis

> **Guiding principle:** Reproduce → Isolate → Confirm → Fix → Verify. Never fix something you can't confirm is broken.

---

### 7.1 Application Debugging Workflow

```bash
# Step 1: Check if the service is responding at all
curl -s -o /dev/null -w "%{http_code}" http://service/health

# Step 2: Check error rate in the last 5 minutes
grep "$(date '+%Y-%m-%dT%H:%M' -d '5 minutes ago')" /var/log/app/app.log | grep -c ERROR

# Step 3: Find the first occurrence of the error (when did it start?)
grep -m 1 "ERROR: DatabaseConnectionException" /var/log/app/app.log

# Step 4: Get context around the error (what happened before?)
grep -n "ERROR: DatabaseConnectionException" /var/log/app/app.log | head -1 | cut -d: -f1 | xargs -I{} sed -n "$(({}  -10)),$(({}+5))p" /var/log/app/app.log

# Step 5: Check if it correlates with a deployment
git log --oneline --since="2 hours ago"
```

---

### 7.2 Performance Bottleneck Identification

```bash
# CPU profiling (Java)
jstack <pid> > /tmp/thread-dump.txt
# Look for "BLOCKED" threads and identify contention

# Strace — system call analysis (use carefully in prod)
strace -p <pid> -c -T -e trace=network,file
# -c = count/summarize, -T = time per call

# perf — kernel-level profiling
perf top -p <pid>
perf stat -p <pid> sleep 10
```

```bash
# Find slow SQL queries (PostgreSQL)
psql -c "SELECT pid, now() - query_start AS duration, query FROM pg_stat_activity WHERE state = 'active' AND now() - query_start > interval '30 seconds' ORDER BY duration DESC;"

# Find blocking queries
psql -c "SELECT blocked.pid, blocked.query, blocking.pid AS blocking_pid, blocking.query AS blocking_query FROM pg_stat_activity AS blocked JOIN pg_stat_activity AS blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));"
```
| Field | Detail |
|-------|--------|
| **When** | Slow database queries, high CPU in database, request latency spikes |
| **Pitfall** | `KILL` on a blocking query may cause a transaction rollback — confirm with the team before killing production queries |

---

### 7.3 Memory Leak Investigation

```bash
# Watch memory growth over time
watch -n 5 "ps aux --sort=-%mem | head -5"

# Track memory of a specific process over time
while true; do
  ps -o pid,vsz,rss,comm -p <PID>
  sleep 30
done | tee /tmp/memory-track.log

# Check for OOM kill history
dmesg -T | grep -i "killed process"
grep -i "oom" /var/log/syslog | tail -20
```

---

## 8. Cloud (AWS-focused)

> **Note:** These are the most critical production commands. GCP/Azure equivalents follow each section in parentheses.

---

### 8.1 AWS Core Operations

```bash
# Check EC2 instance health
aws ec2 describe-instance-status --instance-ids i-xxxxx --query 'InstanceStatuses[*].[InstanceState.Name,InstanceStatus.Status,SystemStatus.Status]'

# List instances by state (find stopped/terminated instances)
aws ec2 describe-instances --filters "Name=instance-state-name,Values=stopped" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],State.Name]' \
  --output table
```

```bash
# EKS — update kubeconfig for a cluster
aws eks update-kubeconfig --region us-east-1 --name my-cluster

# Check EKS node group status
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name ng-1 \
  --query 'nodegroup.[status,scalingConfig,health]'
```

```bash
# CloudWatch Logs — query without the console
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern "ERROR" \
  --query 'events[*].[timestamp,message]' \
  --output text | tail -50
```
| Field | Detail |
|-------|--------|
| **When** | Lambda errors, ECS task failures, any AWS service logging to CloudWatch |
| **Pitfall** | CloudWatch timestamps are in milliseconds epoch — multiply `date +%s` by 1000 |

```bash
# S3 — check bucket size and object count
aws s3 ls s3://my-bucket --recursive --human-readable --summarize | tail -2

# Sync a directory (dry-run first)
aws s3 sync ./data s3://my-bucket/data --dryrun
aws s3 sync ./data s3://my-bucket/data --delete
```

```bash
# RDS — check instance status and recent events
aws rds describe-db-instances --db-instance-identifier mydb \
  --query 'DBInstances[*].[DBInstanceStatus,Endpoint.Address,MultiAZ]'

aws rds describe-events --source-identifier mydb --source-type db-instance \
  --duration 60  # Last 60 minutes
```

---

### 8.2 IAM & Permissions Debugging

```bash
# Simulate an IAM policy — does this role allow this action?
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/my-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# Who am I? (Verify identity in scripts)
aws sts get-caller-identity
```
| Field | Detail |
|-------|--------|
| **When** | Access denied errors, verifying IAM roles before deployment |
| **Pitfall** | Always run `aws sts get-caller-identity` first in scripts to confirm you're using the right credentials/profile |

---

### 8.3 Cost & Resource Audit

```bash
# Find unattached EBS volumes (paying for nothing)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' --output table

# Find old AMIs
aws ec2 describe-images --owners self \
  --query 'sort_by(Images, &CreationDate)[-10:].[ImageId,Name,CreationDate]' --output table
```

---

## 9. Git

> **Guiding principle:** Git is append-only in practice. Almost everything is recoverable. Stay calm and think before you force-push.

---

### 9.1 Daily Operations

```bash
git log --oneline --graph --decorate --all -20
```
| Field | Detail |
|-------|--------|
| **What** | Compact visual branch history |
| **Why** | Understand branch topology before merging or rebasing |
| **When** | Before any complex merge, understanding what changed |
| **Pitfall** | Without `--all`, you only see the current branch |

```bash
git diff HEAD~1 HEAD -- path/to/file
```
| Field | Detail |
|-------|--------|
| **What** | Show exact changes in the last commit, optionally filtered to a file |
| **Why** | Understand exactly what a commit changed before deploying |
| **When** | Code review, investigating a regression |
| **Pitfall** | `HEAD~1` is the previous commit. `HEAD~5` is 5 commits back |

```bash
git stash push -m "WIP: debugging prod issue"
git stash pop
```
| Field | Detail |
|-------|--------|
| **What** | Temporarily save uncommitted changes |
| **Why** | Switch branches or pull without committing half-done work |
| **When** | You need to switch context quickly during an incident |
| **Pitfall** | Stashes are local — don't rely on them for long-term storage. Untracked files need `--include-untracked` |

---

### 9.2 Recovery Operations

```bash
git reflog
```
| Field | Detail |
|-------|--------|
| **What** | Log of every HEAD movement including lost commits |
| **Why** | Recover from accidental reset, lost commits, bad rebases |
| **When** | You've lost a commit or done an accidental `git reset --hard` |
| **Example** | `git checkout <sha-from-reflog>` to recover |
| **Pitfall** | Reflog is local only and expires (default 90 days). Remote doesn't have your reflog |

```bash
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
# Git checks out a midpoint commit — test it, then:
git bisect good  # or git bisect bad
# Repeat until git identifies the first bad commit
git bisect reset
```
| Field | Detail |
|-------|--------|
| **What** | Binary search through commit history to find which commit introduced a bug |
| **Why** | Narrow down regressions from hundreds of commits to the exact culprit |
| **When** | "This worked in last release but not now" — binary search the history |
| **Pitfall** | You need a reliable way to test good vs bad for each commit. Automate the test if possible: `git bisect run ./test.sh` |

```bash
git cherry-pick <commit-sha>
```
| Field | Detail |
|-------|--------|
| **What** | Apply a specific commit to the current branch |
| **Why** | Hotfix: apply a fix to main without merging the entire feature branch |
| **When** | Emergency fix that needs to go to production before a full release |
| **Pitfall** | Cherry-picking creates a new commit with a different SHA — can lead to conflicts when the branch eventually merges |

---

## 10. Security

> **Guiding principle:** Security in production is about blast radius. Know what's exposed, what has access to what, and detect anomalies fast.

---

### 10.1 Access & Authentication

```bash
# Check who's currently logged in
who
w
last -20  # Last 20 logins

# Check sudo history
grep sudo /var/log/auth.log | tail -30
journalctl _COMM=sudo --since "24 hours ago"
```

```bash
# SSH - test connection with verbose output (debug auth failures)
ssh -vvv user@host 2>&1 | grep -E "Authenticating|debug1: Trying|Authenticated"

# Check authorized keys
cat ~/.ssh/authorized_keys
# In production: audit who has SSH access regularly
```

---

### 10.2 Port & Service Exposure Audit

```bash
# What ports are exposed externally?
nmap -sV --open -p 1-65535 <external-ip>

# Quick check of what's listening
ss -tulnp | grep -v "127.0.0.1"

# Check for unexpected SUID binaries
find / -perm -4000 -type f 2>/dev/null
```
| Field | Detail |
|-------|--------|
| **When** | Security audit, post-incident review, new environment setup |
| **Pitfall** | Never run nmap against systems you don't own or without authorization — it's detectable and potentially illegal |

---

### 10.3 Secrets Management

```bash
# Scan git history for accidentally committed secrets (truffleHog)
trufflehog git file://. --since-commit HEAD~50

# Scan Docker image for secrets
trivy image myrepo/service:latest

# Check if secrets are in environment variables (careful who's watching)
env | grep -iE "password|secret|key|token|api" | sed 's/=.*/=REDACTED/'
```
| Field | Detail |
|-------|--------|
| **When** | Pre-deployment security check, post-incident audit |
| **Pitfall** | If secrets are found in git history, rotation is not enough — the entire history is compromised. You must rotate AND rewrite history or make the repo private |

```bash
# Rotate a Kubernetes secret without downtime
kubectl create secret generic db-creds \
  --from-literal=password="new-password" \
  --dry-run=client -o yaml | kubectl apply -f -

# Trigger a rolling restart to pick up the new secret
kubectl rollout restart deployment/<name> -n <namespace>
```

---

### 10.4 Certificate Management

```bash
# Check SSL cert expiry
echo | openssl s_client -connect api.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check cert SANs (Subject Alternative Names)
echo | openssl s_client -connect api.example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"

# Check cert chain
openssl s_client -connect api.example.com:443 -showcerts 2>/dev/null | grep "subject\|issuer"
```
| Field | Detail |
|-------|--------|
| **When** | SSL errors in production, cert expiry alerts, new cert deployment |
| **Pitfall** | A cert can be valid but not trusted if the intermediate CA isn't served. Always check the full chain |

---

## 11. Incident Response Workflows

> **This section is your 2 AM guide. Follow the sequence. Don't skip steps.**

---

### The Production Incident Playbook

```
SEVERITY LEVELS:
  P1 — Customer-facing service fully down or data loss occurring
  P2 — Significant degradation, partial outage, or high error rate
  P3 — Non-critical issue, one region/customer affected
  P4 — Cosmetic/minor, no customer impact
```

---

### 11.1 P1 Response Sequence (First 15 Minutes)

```bash
# T+0: Acknowledge the alert. Open incident channel. Declare IC (Incident Commander).

# T+1: Blast radius assessment
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
# How many services? How many pods? Which namespaces?

# T+2: Customer impact check
curl -s https://api.example.com/health
# Is it fully down or degraded?

# T+3: What changed in the last 30 minutes?
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -30
git log --oneline --all --since="30 minutes ago"  # Any recent commits/deploys?

# T+5: If a recent deployment caused it — ROLLBACK IMMEDIATELY
kubectl rollout undo deployment/<name> -n <namespace>
# Don't investigate while the customer is impacted. Restore first, investigate second.

# T+10: If no deployment, start triage:
# - Check metrics: CPU, memory, error rates, latency
# - Check logs: first ERROR occurrence
# - Check dependencies: is the database up? External APIs?
```

---

### 11.2 Service Down — Diagnosis Tree

```
Is the pod running?
├── NO (CrashLoopBackOff / Error)
│   ├── kubectl describe pod <pod>   → check Events section
│   ├── kubectl logs <pod> --previous → what did it say before crashing?
│   └── Common causes:
│       ├── OOMKilled → increase memory limits
│       ├── ImagePullBackOff → image doesn't exist or registry auth failed
│       ├── CrashLoopBackOff → app crashing, check app logs
│       └── Pending → no schedulable nodes, PVC not bound, insufficient resources
│
└── YES (Running)
    ├── Is it serving traffic?
    │   ├── curl the pod IP directly (bypass service/LB)
    │   │   kubectl get pod <pod> -o jsonpath='{.status.podIP}'
    │   │   kubectl exec -it <debug-pod> -- curl http://<pod-ip>:<port>/health
    │   └── If pod responds but service doesn't:
    │       ├── kubectl describe svc → check selector matches pod labels
    │       └── kubectl get endpoints → empty = selector mismatch
    │
    └── Is it healthy but slow?
        ├── kubectl top pod <pod>      → CPU/memory
        ├── Check downstream deps (DB, external APIs)
        └── Check error rate in logs
```

---

### 11.3 High Error Rate — Investigation Sequence

```bash
# 1. When did errors start?
grep "ERROR" /var/log/app/app.log | awk '{print $1, $2}' | head -1

# 2. What type of errors?
grep "ERROR" /var/log/app/app.log | grep -oP "(?<=ERROR: )[^\n]+" | sort | uniq -c | sort -rn | head -10

# 3. Is it all instances or just some?
# In Kubernetes: check error rate per pod
for pod in $(kubectl get pods -n prod -l app=api -o name); do
  echo "=== $pod ===";
  kubectl logs $pod -n prod --since=5m | grep -c ERROR
done

# 4. Is it a specific endpoint?
grep "ERROR" /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10

# 5. Is it a specific user/customer? (check for correlation)
grep "ERROR" /var/log/app/app.log | grep -oP "user_id=\K\d+" | sort | uniq -c | sort -rn | head -5
```

---

### 11.4 Database Issues

```bash
# PostgreSQL — is it accepting connections?
psql -h db.internal -U app -d mydb -c "SELECT 1;"

# Connection pool exhaustion
psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
# Too many "idle in transaction" = connection leak

# Long-running queries
psql -c "SELECT pid, now() - query_start, state, query FROM pg_stat_activity WHERE now() - query_start > interval '1 minute' ORDER BY 2 DESC;"

# Table bloat (after many updates/deletes without VACUUM)
psql -c "SELECT schemaname, tablename, n_dead_tup, n_live_tup FROM pg_stat_user_tables WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC;"

# Kill a specific blocking query (confirm first!)
psql -c "SELECT pg_terminate_backend(<pid>);"
```
| Field | Detail |
|-------|--------|
| **Pitfall** | Never `pg_terminate_backend` without confirming with the team. Terminating a long-running transaction may trigger a slow rollback that makes things worse |

---

### 11.5 Memory/OOM Incidents

```bash
# Was there an OOM kill?
dmesg -T | grep -i "killed process\|oom"
kubectl get events -n <namespace> | grep OOMKill

# Which container was OOM killed?
kubectl describe pod <pod> | grep -A5 "OOMKilled\|Last State"

# Current memory headroom
kubectl top pods -n <namespace> | sort -k3 -rn

# Check for memory leak pattern (usage always climbing, never releasing)
# Look at container_memory_working_set_bytes in Prometheus over 24h
```

**Recovery actions for OOM:**
```bash
# Immediate: restart the pod
kubectl rollout restart deployment/<name> -n <namespace>

# Short-term: increase memory limits
kubectl set resources deployment/<name> -c=<container> --limits=memory=2Gi

# Long-term: fix the memory leak in code
```

---

### 11.6 Post-Incident (Blameless)

After every P1/P2, document:

```
1. TIMELINE: Exact sequence of events (alert time, response time, fix time)
2. ROOT CAUSE: The actual technical cause (not "human error")
3. CONTRIBUTING FACTORS: What made it worse or harder to detect
4. CUSTOMER IMPACT: How long, how many users, what functionality
5. WHAT WORKED: Tools/runbooks that helped
6. ACTION ITEMS:
   - Detection improvement (better alerts/dashboards)
   - Prevention (code/config fix)
   - Response improvement (runbook updates, automation)
   Each item must have: owner, due date, and priority
```

---

## Quick Reference Cards

### Pod Status Decoder
| Status | Meaning | Action |
|--------|---------|--------|
| `Pending` | No node available to schedule | Check node resources, PVC, taints |
| `CrashLoopBackOff` | Container keeps crashing | Check `logs --previous`, fix app |
| `OOMKilled` | Container exceeded memory limit | Increase limits or fix memory leak |
| `ImagePullBackOff` | Can't pull container image | Check image name, tag, registry auth |
| `Error` | Container exited with error | Check logs |
| `Terminating` stuck | Pod won't delete | `kubectl delete pod <pod> --force --grace-period=0` |
| `Running` (but unhealthy) | App up but failing health checks | Check readiness probe config |

---

### The 10 Most Common Production Problems & First Commands

| Problem | First Command |
|---------|---------------|
| Service returns 5xx | `kubectl logs <pod> --tail=100 \| grep ERROR` |
| Service is slow | `kubectl top pods && curl -w "%{time_total}" <endpoint>` |
| Pod won't start | `kubectl describe pod <pod>` |
| Disk full | `df -h && du -sh /var/log/* \| sort -rh \| head -10` |
| High CPU | `top -c` then `ps aux --sort=-%cpu \| head -10` |
| Memory pressure | `free -h && dmesg \| grep -i oom` |
| Can't connect to service | `nc -zv <host> <port> && kubectl get endpoints <svc>` |
| DNS not resolving | `dig +short <hostname> && dig @8.8.8.8 <hostname>` |
| Deployment stuck | `kubectl rollout status deployment/<name>` |
| Database unreachable | `nc -zv db.host 5432 && psql -c "SELECT 1"` |

---

### Golden Signals (What to Always Monitor)
| Signal | What It Measures | Alert When |
|--------|-----------------|------------|
| **Latency** | How long requests take | P99 > SLA threshold |
| **Traffic** | Request rate | Sudden drop = possible outage |
| **Errors** | Error rate | >1% for critical services |
| **Saturation** | Resource headroom | CPU >80%, Memory >85% |

---

*"Speed comes from preparation. Under pressure, you don't rise to the occasion — you fall to your level of preparation."*

---

**Version:** 1.0 | **Last Updated:** 2026 | **Maintained by:** Principal DevOps
