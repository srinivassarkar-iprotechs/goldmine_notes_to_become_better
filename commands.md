## 🐧 Linux 

### Files & Navigation

```bash
ls -lah
cd /path
pwd
tree -L 2
```

### File Ops

```bash
cp -r src dst
mv old new
rm -rf dir
touch file
mkdir -p a/b/c
```

### View / Search

```bash
cat file
less file
tail -f app.log
grep -R "ERROR" .
```

### Processes & Ports (VERY IMPORTANT)

```bash
ps aux
top
htop
kill -9 <pid>
lsof -i :3000
netstat -tulpn
```

### Permissions

```bash
chmod +x script.sh
chmod 644 file
chown user:group file
```

### Disk & Memory

```bash
df -h
du -sh *
free -h
```


## (Power Moves)

### `curl` — API sanity check (MANDATORY for backend)

```bash
curl -i http://localhost:3000/health
curl -X POST -H "Authorization: Bearer <token>" url
```

If you can’t curl it, it doesn’t exist.

---

### `jq` — Read JSON like a human

```bash
cat response.json | jq
kubectl get pod -o json | jq '.status.phase'
```

Senior engineers don’t scroll JSON.

---

### `watch` — Observe behavior over time

```bash
watch kubectl get pods
watch df -h
```

Great for spotting flapping pods.

---

### `env` / `printenv`

```bash
printenv | grep DB
```

Half of prod bugs are env bugs.

---
---

## 🐳 Docker 

### Images & Containers

```bash
docker ps
docker ps -a
docker images
docker pull node:18
```

### Run & Debug

```bash
docker run -p 3000:3000 image
docker exec -it <container> sh
docker logs <container>
docker logs -f <container>
```

### Build & Cleanup

```bash
docker build -t app:1.0 .
docker stop <id>
docker rm <id>
docker rmi <image>
docker system prune
```

### Inspect (Senior move)

```bash
docker inspect <container>
docker stats
```

---

## ☸️ Kubernetes

### Context & Cluster Safety

```bash
kubectl config get-contexts
kubectl config use-context dev
```

> This alone prevents production disasters.

---

### Pods & Debugging (MOST USED)

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> -c <container>
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

### YAML > Imperative (Architect mindset)

```bash
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl diff -f deployment.yaml
```

### Events (underrated but 🔥)

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```



### `kubectl top` (resource pressure)

```bash
kubectl top pods
kubectl top nodes
```

Explains random restarts & throttling.

---

### `kubectl api-resources`

```bash
kubectl api-resources
```

When you forget what even exists in the cluster.

---

### `kubectl explain`

```bash
kubectl explain deployment.spec.template.spec.containers.resources
```

This is *documentation inside the cluster*. Huge CKA edge.

---

### `kubectl get all`

```bash
kubectl get all
```

Quick situational awareness.

---

## 🌐 Networking (Don’t skip these)

### `ping`, `traceroute`

```bash
ping service-name
traceroute google.com
```

Basic, but still relevant.

---

### `nc` (netcat) — Port-level truth

```bash
nc -zv localhost 5432
```

Kills “maybe the DB is up?” debates instantly.

---
