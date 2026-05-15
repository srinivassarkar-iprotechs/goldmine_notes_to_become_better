## A. FINAL — Ultimate Top 25 On-Call Commands

**(Battle-tested production order: diagnose → inspect → isolate → recover)**

| #      | Command                                     | Purpose                           |
| ------ | ------------------------------------------- | --------------------------------- |
| **1**  | `uptime`                                    | Check load / system stress        |
| **2**  | `top` / `htop`                              | Live CPU & memory pressure        |
| **3**  | `free -h`                                   | RAM / swap exhaustion             |
| **4**  | `vmstat 1`                                  | Real-time contention analysis     |
| **5**  | `df -h`                                     | Disk usage                        |
| **6**  | `du -sh /* 2>/dev/null \| sort -hr \| head` | Find disk hogs                    |
| **7**  | `ps aux --sort=-%cpu \| head`               | Top CPU consumers                 |
| **8**  | `ps aux --sort=-%mem \| head`               | Top memory consumers              |
| **9**  | `dmesg -T \| tail -50`                      | Kernel / OOM / system failures    |
| **10** | `journalctl -xe`                            | Critical system events            |
| **11** | `journalctl -u <service> -n 100 --no-pager` | Service-specific logs             |
| **12** | `tail -f /var/log/app.log`                  | Live app logs                     |
| **13** | `systemctl status <service>`                | Service state                     |
| **14** | `ss -tulpn`                                 | Open/listening ports              |
| **15** | `lsof -i :PORT`                             | Who owns a port                   |
| **16** | `curl -v http://localhost:PORT/health`      | Health endpoint validation        |
| **17** | `nc -zv host port`                          | Connectivity test                 |
| **18** | `docker ps -a`                              | Container state                   |
| **19** | `docker logs --tail=100 <container>`        | Container logs                    |
| **20** | `docker inspect <container>`                | Container restart reason / config |
| **21** | `kubectl get pods -A`                       | Cluster-wide pod health           |
| **22** | `kubectl describe pod <pod>`                | Events / scheduling / failures    |
| **23** | `kubectl logs <pod> --previous`             | Crash logs                        |
| **24** | `kubectl exec -it <pod> -- sh`              | Live pod inspection               |
| **25** | `kubectl rollout undo deployment/<name>`    | Emergency rollback                |

---

## If You Can Memorize Only 10

```bash
uptime
free -h
df -h
top
journalctl -xe
systemctl status <service>
curl localhost:PORT/health
docker logs <container>
kubectl describe pod <pod>
kubectl rollout undo deployment/<name>
```

---

## The Real 3AM Incident Flow

### 1. Node health

```bash
uptime
free -h
df -h
vmstat 1
```

---

### 2. Process pressure

```bash
top
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

---

### 3. Logs

```bash
dmesg -T | tail -50
journalctl -xe
journalctl -u <service> -n 100
```

---

### 4. Service reachability

```bash
ss -tulpn
lsof -i :PORT
curl -v localhost:PORT/health
```

---

### 5. Container checks

```bash
docker ps -a
docker logs <container>
docker inspect <container>
```

---

### 6. Kubernetes checks

```bash
kubectl get pods -A
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
```

---

### 7. If deploy caused issue

```bash
kubectl rollout undo deployment/<name>
```

