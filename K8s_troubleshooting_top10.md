# ⚙️ Kubernetes Troubleshooting Guide — Top 10 All Time
> **Your holy book for production-grade K8s debugging. Bookmark it. Memorize it. Live by it.**

---

## #1 — Pod Stuck in `CrashLoopBackOff`

**What it means:** The container starts, crashes, and Kubernetes keeps restarting it in an exponential backoff loop.

**Diagnose:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous     # logs from last crashed container
kubectl logs <pod-name> -n <namespace> -c <container> # specific container in multi-container pod
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Application crash on startup | Check app logs. Fix the bug. |
| Missing environment variable / secret | Verify `env`, `envFrom`, and `secretKeyRef` refs exist |
| Wrong command / entrypoint in Dockerfile | Override `command` in pod spec temporarily to `["sleep","3600"]` and exec in |
| OOMKilled (out of memory) | Increase `resources.limits.memory` or fix memory leak |
| Liveness probe too aggressive | Increase `initialDelaySeconds` to give app time to start |
| Config file missing | Check `ConfigMap` mount path matches what the app expects |

**Golden debug trick:**
```bash
# Override entrypoint to keep pod alive and exec into it
kubectl run debug --image=<your-image> --command -- sleep 3600
kubectl exec -it debug -- /bin/sh
```

---

## #2 — Pod Stuck in `Pending`

**What it means:** The scheduler can't place the pod on any node.

**Diagnose:**
```bash
kubectl describe pod <pod-name> -n <namespace>
# Look at "Events" section at the bottom — it tells you everything
```

**Root causes & fixes:**

| Cause | Symptom in Events | Fix |
|---|---|---|
| Insufficient CPU/Memory | `Insufficient cpu` / `Insufficient memory` | Scale cluster or reduce `resources.requests` |
| No nodes match `nodeSelector` | `didn't match node selector` | Fix label selectors or label a node |
| `PersistentVolumeClaim` not bound | `pod has unbound PVC` | Check PVC/PV status, StorageClass |
| Taints with no matching tolerations | `had taints that the pod didn't tolerate` | Add `tolerations` to pod spec |
| Affinity/anti-affinity rules unsatisfiable | `node(s) didn't match pod affinity` | Relax affinity rules |
| Image pull backoff blocking scheduling | — | Fix image name/tag, check imagePullSecrets |

**Check node capacity:**
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl get pods -A --field-selector=spec.nodeName=<node-name>  # what's running on a node
```

---

## #3 — `ImagePullBackOff` / `ErrImagePull`

**What it means:** Kubernetes can't pull the container image.

**Diagnose:**
```bash
kubectl describe pod <pod-name> | grep -A 10 Events
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Wrong image name or tag | `kubectl set image deployment/<name> <container>=<correct-image>` |
| Private registry, no pull secret | Create `docker-registry` secret + add to `imagePullSecrets` |
| Registry is down or rate-limited | Docker Hub has rate limits for anonymous pulls — use authenticated pull secret |
| Digest mismatch (SHA pinned) | Update the digest or switch to a mutable tag |

**Create image pull secret:**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```

**Add to pod spec:**
```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

---

## #4 — Service Not Routing Traffic to Pods

**What it means:** A Service exists but requests aren't reaching pods.

**Diagnose:**
```bash
kubectl get svc <svc-name> -n <namespace>
kubectl describe svc <svc-name> -n <namespace>
kubectl get endpoints <svc-name> -n <namespace>   # CRITICAL: If "none" — selector is broken
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| Selector mismatch (labels don't match) | `kubectl get pods --show-labels` vs `svc.spec.selector` — must be identical |
| Pod not ready (failing readiness probe) | Fix readiness probe or fix what it checks |
| Wrong `targetPort` | Match `targetPort` to the port the app **actually** listens on inside the container |
| NetworkPolicy blocking traffic | Check `NetworkPolicy` resources in the namespace |
| Service port vs container port mismatch | Re-check `port`, `targetPort`, and `containerPort` |

**End-to-end connectivity debug:**
```bash
# Spawn a debug pod and curl the service
kubectl run curl-debug --image=curlimages/curl -it --rm -- \
  curl http://<service-name>.<namespace>.svc.cluster.local:<port>

# Check DNS resolution
kubectl run dns-debug --image=busybox -it --rm -- nslookup <service-name>.<namespace>
```

---

## #5 — OOMKilled — Container Out of Memory

**What it means:** The container exceeded its memory limit and the kernel killed it.

**Diagnose:**
```bash
kubectl describe pod <pod-name> | grep -i "OOMKilled\|Last State\|Exit Code"
# Exit Code 137 = OOMKilled
kubectl top pod <pod-name> -n <namespace>   # requires metrics-server
```

**Fixes:**

```yaml
# In your deployment/pod spec
resources:
  requests:
    memory: "256Mi"   # what scheduler reserves
  limits:
    memory: "512Mi"   # hard ceiling — exceed this = OOMKilled
```

**Best practices:**
- Set `requests` based on observed usage (from `kubectl top`)
- Set `limits` at ~2x `requests` for headroom
- Never omit limits in production — one bad pod can starve the node
- Use VPA (Vertical Pod Autoscaler) to auto-tune over time
- Check for memory leaks with heap profiling (`pprof`, `jmap`, etc.)

---

## #6 — Node Not Ready (`NotReady`)

**What it means:** A node is unreachable or unhealthy — pods may be evicted.

**Diagnose:**
```bash
kubectl get nodes
kubectl describe node <node-name>        # Check "Conditions" section
kubectl get events -n kube-system        # Cluster-level events
ssh <node>                               # If possible — check system health
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| kubelet crashed or stopped | `systemctl restart kubelet` on the node |
| Disk pressure (`DiskPressure: True`) | Free disk space. Delete old images: `docker system prune` or `crictl rmi --prune` |
| Memory pressure (`MemoryPressure: True`) | Evict non-critical pods, scale down workloads |
| Network partition | Check node networking, VPC/subnet routing, security groups |
| Node kernel panic / hardware failure | Drain + terminate + replace the node |
| Certificate expired | Renew kubelet certs: `kubeadm certs renew all` |

**Safe node drain (before maintenance/replacement):**
```bash
kubectl cordon <node-name>              # Stop new pods from scheduling
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force
```

---

## #7 — Deployment Not Rolling Out (Stuck)

**What it means:** A rolling update is stuck — new pods won't come up, old ones won't terminate.

**Diagnose:**
```bash
kubectl rollout status deployment/<name> -n <namespace>
kubectl describe deployment <name> -n <namespace>
kubectl get rs -n <namespace>           # Check ReplicaSets
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| New pods crashlooping (see #1) | Fix the underlying pod issue |
| Insufficient cluster resources (see #2) | Scale cluster |
| `maxUnavailable: 0` + `maxSurge: 0` | Impossible config — fix rollout strategy |
| Readiness probe failing on new version | Fix probe or fix what it checks |
| PodDisruptionBudget blocking it | Check PDB: `kubectl get pdb -n <namespace>` |
| Quota exceeded | `kubectl describe quota -n <namespace>` |

**Rollback immediately when things go wrong:**
```bash
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> --to-revision=3 -n <namespace>  # specific revision
kubectl rollout history deployment/<name> -n <namespace>               # see revisions
```

---

## #8 — PersistentVolumeClaim (PVC) Stuck in `Pending`

**What it means:** Storage can't be provisioned for the pod.

**Diagnose:**
```bash
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get storageclass
kubectl get pv                          # Check available PersistentVolumes
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| No matching StorageClass | Specify correct `storageClassName` or create the SC |
| No available PV (static provisioning) | Create a matching PV manually |
| StorageClass provisioner not installed | Install CSI driver (e.g., EBS CSI, NFS provisioner) |
| Capacity mismatch | PV must be >= PVC requested size |
| AccessMode mismatch | PV must support the requested `accessMode` (`ReadWriteOnce`, etc.) |
| Wrong namespace | PV is cluster-scoped, PVC is namespace-scoped — check binding |

**Check dynamic provisioning works:**
```bash
kubectl get storageclass
# Look for (default) annotation — if none, PVCs without storageClassName won't auto-provision
kubectl patch storageclass <name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## #9 — DNS Resolution Failing Inside Pods

**What it means:** Pods can't resolve service names or external domains.

**Diagnose:**
```bash
# Run a debug pod
kubectl run dns-test --image=busybox:1.28 -it --rm -- /bin/sh

# Inside the pod:
nslookup kubernetes.default
nslookup <service>.<namespace>.svc.cluster.local
cat /etc/resolv.conf
nslookup google.com        # test external DNS

# Check CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Root causes & fixes:**

| Cause | Fix |
|---|---|
| CoreDNS pods not running | Restart: `kubectl rollout restart deployment/coredns -n kube-system` |
| CoreDNS ConfigMap misconfigured | `kubectl edit configmap coredns -n kube-system` |
| NetworkPolicy blocking UDP/TCP port 53 to CoreDNS | Add egress rule for port 53 |
| Node-level iptables corruption | Restart kube-proxy or flush iptables on the node |
| Wrong `dnsPolicy` on pod | Use `ClusterFirst` (default) unless you have a specific reason |
| ndots setting causing slow lookups | Tune `ndots` in pod's `dnsConfig` |

**CoreDNS ConfigMap check:**
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

---

## #10 — RBAC `Forbidden` — Permission Denied Errors

**What it means:** A pod, service account, or user is trying to do something they're not authorized to do.

**Diagnose:**
```bash
# See the exact error
kubectl describe pod <pod-name>   # look for "Forbidden" in events

# Test permissions as a specific service account
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount>

# List all permissions for a role
kubectl describe role <role-name> -n <namespace>
kubectl describe clusterrole <role-name>

# Check what's bound to a service account
kubectl get rolebindings,clusterrolebindings -A \
  -o custom-columns='NAME:.metadata.name,SUBJECTS:.subjects[*].name' | grep <sa-name>
```

**Fix — full RBAC setup pattern:**
```yaml
# 1. ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
---
# 2. Role (namespace-scoped) or ClusterRole (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: my-namespace
rules:
  - apiGroups: [""]
    resources: ["pods", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch"]
---
# 3. RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-rolebinding
  namespace: my-namespace
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-namespace
roleRef:
  kind: Role
  name: my-app-role
  apiGroup: rbac.authorization.k8s.io
```

**Quick permission audit:**
```bash
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa> -n <ns>
```

---

## 🧰 Universal Debugging Toolkit

```bash
# All pods across all namespaces — spot unhealthy ones instantly
kubectl get pods -A | grep -v Running | grep -v Completed

# Events sorted by time (your best friend)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Resource usage across pods
kubectl top pods -A --sort-by=memory

# Exec into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Port-forward to debug a service locally
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>

# Copy files to/from a pod
kubectl cp <pod-name>:/app/logs/app.log ./app.log -n <namespace>

# Watch pod status in real-time
kubectl get pods -n <namespace> -w

# Get all resource YAML (great for diffing)
kubectl get deployment <name> -n <namespace> -o yaml
```

---

## 📐 Mental Model: K8s Debugging Decision Tree

```
Pod not working?
├── Status = Pending     → #2 Scheduling issue (resources, taints, PVC)
├── Status = ImagePull*  → #3 Registry/image issue
├── Status = CrashLoop   → #1 App crash (check --previous logs)
├── Status = OOMKilled   → #5 Memory limits too low
├── Status = Running but no traffic → #4 Service selector / endpoints
├── Node = NotReady      → #6 Node health issue
└── Permission error     → #10 RBAC
```

---

*Master these 10 and you'll resolve 95% of production K8s incidents faster than anyone else in the room.*
