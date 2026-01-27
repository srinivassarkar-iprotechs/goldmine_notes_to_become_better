# 🐧 Linux / 🐳 Docker / ☸️ Kubernetes — One-Page Ops Cheat Sheet

## 🐧 Linux

### Files & Navigation

```bash
ls -lah    cd /path    pwd    tree -L 2
```

### File Operations

```bash
cp -r src dst   mv old new   rm -rf dir
touch file     mkdir -p a/b/c
```

### View / Search

```bash
cat file     less file     tail -f app.log
grep -R "ERROR" .
```

### Processes & Ports ⭐

```bash
ps aux     top / htop
kill -9 <pid>
lsof -i :3000     netstat -tulpn
```

### Permissions

```bash
chmod +x script.sh   chmod 644 file
chown user:group file
```

### Disk & Memory

```bash
df -h     du -sh *     free -h
```

---

## ⚡ Power Moves (Daily Drivers)

### curl — API reality check

```bash
curl -i http://localhost:3000/health
curl -X POST -H "Authorization: Bearer <token>" url
```

### jq — Human-readable JSON

```bash
cat response.json | jq
kubectl get pod -o json | jq '.status.phase'
```

### watch — Observe behavior

```bash
watch kubectl get pods
watch df -h
```

### env — Config sanity

```bash
printenv | grep DB
```

---

## 🐳 Docker

### Images & Containers

```bash
docker ps      docker ps -a
docker images  docker pull node:18
```

### Run & Debug

```bash
docker run -p 3000:3000 image
docker exec -it <container> sh
docker logs <container>   docker logs -f <container>
```

### Build & Cleanup

```bash
docker build -t app:1.0 .
docker stop <id>   docker rm <id>
docker rmi <image>
docker system prune
```

### Inspect

```bash
docker inspect <container>
docker stats
```

---

## ☸️ Kubernetes

### Context Safety ⭐

```bash
kubectl config get-contexts
kubectl config use-context dev
```

### Pods & Debugging (Most Used)

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod> [-c <container>]
kubectl exec -it <pod> -- sh
```

### Namespaces

```bash
kubectl get ns
kubectl get pods -n backend
```

### Deployments

```bash
kubectl get deploy
kubectl describe deploy <name>
kubectl rollout status deploy/<name>
kubectl rollout restart deploy/<name>
```

### Services & Networking

```bash
kubectl get svc
kubectl describe svc <name>
kubectl port-forward svc/api 8080:80
```

### YAML > Imperative

```bash
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl diff -f deployment.yaml
```

### Events & Resources

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods     kubectl top nodes
kubectl api-resources
kubectl explain deployment.spec.template.spec.containers.resources
kubectl get all
```

---

## 🌐 Networking

```bash
ping service-name
traceroute google.com
nc -zv localhost 5432
```

---

