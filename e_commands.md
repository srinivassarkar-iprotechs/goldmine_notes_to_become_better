
# 🐧 Linux

## 🔍 Search Logs / Code
```bash
grep -Ri "ERROR" .                  # Recursively search "ERROR" in current dir
grep -E "ERROR|WARN|FATAL" app.log  # Search multiple patterns
tail -f app.log | grep ERROR        # Live stream only ERROR logs
```

## ⚙️ Extract / Process
```bash
awk '{print $1,$3}' file            # Print column 1 and 3
ps aux | awk '$3>80'                # Show processes using >80% CPU
jq '.items[].metadata.name'         # Extract names from JSON
```

## ✏️ Replace / Edit
```bash
sed -i 's/old/new/g' file           # Replace all old -> new inside file
sed -n '10,20p' file                # Print only lines 10–20
```

## 📂 Find Files
```bash
find . -name "*.log"                # Find all .log files
find / -size +100M                  # Find files >100MB
find . -mtime +7                    # Files modified >7 days ago
```

## 🧠 Process Checks
```bash
ps aux --sort=-%cpu | head          # Top CPU-consuming processes
pgrep nginx                         # Find nginx process PID
kill -15 <pid>                      # Gracefully stop process
htop                                # Interactive process monitor
```

## 🌐 Port / Network
```bash
lsof -i :8080                       # Which process uses port 8080
ss -tulpn                           # Show listening ports
nc -zv host port                    # Test if port is reachable
ping host                           # Check network connectivity
```

---

# 🌐 HTTP / API

```bash
curl -i http://localhost:3000/health              # Get response + headers
curl -s -o /dev/null -w "%{http_code}" URL        # Only status code
curl -H "Authorization: Bearer $TOKEN" URL        # Authenticated request
curl -X POST -H "Content-Type: application/json" \
-d '{}' URL                                       # POST JSON request
```

---

# 🔐 Remote Access

```bash
ssh user@host                          # Connect to remote server
ssh -i key.pem user@host               # Connect using private key
ssh -L 5432:db:5432 user@bastion       # Port forward DB locally
ssh user@host "df -h && free -h"       # Run remote commands
```

---

# 📦 Files

```bash
tar -czf backup.tar.gz dir/            # Compress directory
tar -xzf backup.tar.gz                 # Extract archive
df -h                                  # Disk usage
du -sh *                               # Folder sizes
```

---

# ⚡ Useful Pipelines

## Find Largest Files
```bash
find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head
```

## Kill All Node Processes
```bash
pgrep -f node | xargs kill -9
```

## Search TODOs
```bash
find . -name "*.js" | xargs grep TODO
```

---

# 🐳 Docker

## Core
```bash
docker build -t app:v1 .               # Build image
docker run -d -p 3000:3000 --name api app:v1
docker ps                              # List running containers
docker logs -f api                     # Follow logs
docker exec -it api sh                 # Shell into container
docker inspect api                     # Full container details
docker stats                           # Live resource usage
docker system prune -af                # Remove unused resources
```

## Compose
```bash
docker compose up -d                   # Start all services
docker compose down -v                 # Stop + remove volumes
docker compose logs -f service         # Follow service logs
docker compose exec service sh         # Shell into service
docker compose ps                      # List compose services
```

## Debug
```bash
docker cp api:/app/file .              # Copy file from container
docker rm $(docker ps -aq -f status=exited)
```

---

# ☸️ Kubernetes

## Context Safety
```bash
kubectl config current-context         # Current cluster
kubectl config use-context dev         # Switch cluster
```

## Pods
```bash
kubectl get pods -A                    # All pods
kubectl get pods -w                    # Watch pod changes
kubectl describe pod <pod>             # Detailed pod info + events
kubectl logs <pod>                     # Current logs
kubectl logs <pod> --previous          # Previous crashed logs
kubectl exec -it <pod> -- sh           # Shell into pod
kubectl cp <pod>:/file .               # Copy file from pod
```

## Deployments
```bash
kubectl get deploy                     # List deployments
kubectl rollout status deploy/api      # Check rollout progress
kubectl rollout restart deploy/api     # Restart deployment
kubectl rollout undo deploy/api        # Rollback
kubectl scale deploy/api --replicas=5  # Scale replicas
kubectl set image deploy/api api=app:v2
```

## Services
```bash
kubectl get svc                        # List services
kubectl describe svc api               # Service details
kubectl port-forward svc/api 8080:80   # Access locally
```

## YAML
```bash
kubectl apply -f file.yaml             # Apply config
kubectl delete -f file.yaml            # Delete resource
kubectl diff -f file.yaml              # Preview changes
kubectl apply -f ./manifests/
```

## Cluster Health
```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
kubectl get all -A
kubectl explain deployment
```

## Incident Commands
```bash
kubectl get pods -A | grep -v Running
kubectl delete pod <pod> --force --grace-period=0
kubectl rollout status deploy/api && kubectl get pods -w
```

---

# 🧠 Incident Flow

| Problem | First Command |
|---------|--------------|
| Pod crash | `kubectl logs --previous` |
| Service down | `kubectl describe pod` |
| Connection issue | `nc -zv host port` |
| High CPU | `htop` |
| Disk full | `df -h` |
| Config issue | `printenv` |
| Broken deploy | `kubectl rollout undo` |

---

# ⭐ Golden Rules

## Linux
`grep → analyze → fix`

## Docker
`logs → exec → inspect`

## Kubernetes
`describe → logs --previous → events → rollback`
