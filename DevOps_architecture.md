# DevOps Architecture – Deep Interview & Implementation Prep
---

## 1. Docker Architecture (VERY COMMON)

### Senior-Level Mental Model

> Docker is **not** a VM manager. It is a **process isolation and lifecycle manager** built on top of Linux kernel primitives.

If you understand Docker as "a structured way to run Linux processes with isolation + limits", you are already thinking like a senior.

---

### What Problem Docker *Actually* Solves

* Eliminates environment drift
* Standardizes build → run pipeline
* Enables immutable infrastructure
* Makes CI/CD reproducible

Docker does **not** solve:

* Security by default
* High availability
* Orchestration (Kubernetes does that)

---

### Docker Architecture – Deep View

```
User Space
+-------------------+
| Docker CLI        |
+-------------------+
          |
          | REST API
          v
+-------------------+
| Docker Daemon     |  ← orchestration, networking, volumes
| (dockerd)         |
+-------------------+
          |
          v
+-------------------+
| containerd        |  ← container lifecycle (create/start/stop)
+-------------------+
          |
          v
+-------------------+
| runc              |  ← OCI runtime (syscalls)
+-------------------+

Kernel Space
+-------------------+
| Linux Kernel      |
| namespaces        |
| cgroups           |
| overlayfs         |
+-------------------+
```

---

### Isolation: What Isolated vs What Is Shared

| Layer         | Isolated?         |
| ------------- | ----------------- |
| Process Tree  | ✅ PID namespace   |
| Network Stack | ✅ Net namespace   |
| Filesystem    | ✅ Mount namespace |
| CPU / Memory  | ✅ cgroups         |
| Kernel        | ❌ Shared          |

> **Interview Insight:** Containers are isolated **from each other**, not from the kernel.

---

### `docker run` – Kernel-Level Breakdown

1. dockerd validates request
2. Image layers mounted via OverlayFS
3. New namespaces created (clone syscall)
4. cgroups applied (cpu, mem, pids)
5. Entrypoint process becomes PID 1

This is why:

* PID 1 signal handling matters
* Containers exit when PID 1 exits

---

### Failure & Production Reality

| Scenario                | Outcome                 |
| ----------------------- | ----------------------- |
| dockerd crash           | Containers keep running |
| container process crash | Container exits         |
| kernel panic            | Everything dies         |

Senior takeaway:

> Docker is **not** a supervisor. External orchestration is required.

---

---

### Docker High‑Level Architecture

```
Docker CLI
   |
   | REST API
   v
Docker Daemon (dockerd)
   |
   v
containerd
   |
   v
runc
   |
   v
Linux Kernel
(namespaces + cgroups + fs)
```

---

### Core Components

| Component     | Responsibility                           |
| ------------- | ---------------------------------------- |
| Docker Client | User interface (CLI/API)                 |
| dockerd       | Manages images, containers, networks     |
| containerd    | Container lifecycle management           |
| runc          | Creates containers using kernel features |
| Image         | Read‑only template                       |
| Container     | Running instance of image                |

---

### How Docker Achieves Isolation

| Mechanism  | Purpose                                |
| ---------- | -------------------------------------- |
| Namespaces | Process, network, PID, mount isolation |
| Cgroups    | CPU, memory, IO limits                 |
| OverlayFS  | Layered filesystem                     |

> **Key Insight:** Containers share the **host kernel**, unlike VMs.

---

### `docker run` – What Actually Happens

1. CLI sends request to dockerd
2. Image pulled (if missing)
3. containerd prepares container
4. runc creates namespaces & cgroups
5. Process starts inside container

---

### Failure Scenarios

* **dockerd crashes** → running containers continue
* **host crashes** → containers die
* Restart policy decides recovery

---

## 2. Kubernetes Architecture (MOST IMPORTANT)

### Senior-Level Mental Model

> Kubernetes is a **distributed control system**, not a container runtime.

It does **state reconciliation**, not imperative execution.

---

### Core Kubernetes Contract

* You declare **desired state**
* Kubernetes **continuously reconciles**
* No guarantees about *how*, only *eventual correctness*

This is why:

* Controllers retry forever
* APIs are idempotent
* Failures are expected

---

### Control Plane – Responsibility Split

```
API Server  → Validation + Auth + Admission
Scheduler  → Decision making (placement)
Controllers→ State reconciliation
etcd       → Single source of truth
```

> **Key Rule:** Nothing talks to etcd directly except the API server.

---

### Pod Lifecycle – With Control Loops

```
User submits spec
   ↓
API Server stores intent
   ↓
Scheduler assigns node
   ↓
kubelet observes state
   ↓
Runtime starts containers
   ↓
Controllers verify health
```

Every arrow is **event-driven**, not synchronous.

---

### Why Pods Exist (Senior Insight)

Pods exist to:

* Share network namespace
* Share storage volumes
* Enable sidecar patterns

A pod is **not** a container wrapper.

---

### Deployment vs ReplicaSet vs Pod

| Object     | Responsibility       |
| ---------- | -------------------- |
| Pod        | Execution            |
| ReplicaSet | Quantity             |
| Deployment | Versioning + rollout |

> You never manage ReplicaSets directly in production.

---

### Failure Handling Philosophy

| Failure             | Kubernetes Response            |
| ------------------- | ------------------------------ |
| Pod crash           | Restart                        |
| Node crash          | Reschedule                     |
| Control plane crash | API unavailable, workloads run |

Senior takeaway:

> Kubernetes assumes **everything will fail**.

---

---

### Kubernetes Cluster Architecture

```
        Control Plane
+-----------------------------+
| API Server                  |
| Scheduler                   |
| Controller Manager          |
| etcd (cluster state)        |
+-----------------------------+
            |
---------------------------------
            |
        Worker Nodes
+-----------------------------+
| kubelet                     |
| kube-proxy                  |
| container runtime           |
+-----------------------------+
```

---

### Pod Creation Flow (VERY COMMON)

```
kubectl apply
   ↓
API Server (validate)
   ↓
etcd (store desired state)
   ↓
Scheduler (pick node)
   ↓
kubelet (node agent)
   ↓
container runtime
   ↓
Pod Running
```

---

### Core Objects Explained

| Object     | Purpose                    |
| ---------- | -------------------------- |
| Pod        | Smallest deployable unit   |
| Deployment | Desired replica management |
| ReplicaSet | Ensures pod count          |
| Service    | Stable networking          |
| Ingress    | HTTP routing               |

---

### How Kubernetes Maintains Desired State

* Controllers watch etcd
* Detect drift (pod crash, node failure)
* Automatically reconcile

---

## 3. CI/CD Architecture (Jenkins / GitHub Actions)

### Core CI/CD Goals

* Fast feedback
* Repeatable builds
* Automated deployments

---

### Jenkins Architecture

```
Git Push
   ↓ (Webhook)
Jenkins Controller
   ↓
Dynamic Agent (Docker / K8s)
   ↓
Build → Test → Artifact
   ↓
Deploy
```

---

### Jenkins Components

| Component     | Role              |
| ------------- | ----------------- |
| Controller    | Orchestration     |
| Agent         | Executes jobs     |
| SCM           | Source code       |
| Artifact Repo | Versioned outputs |

---

### Modern Best Practices

* No static agents
* Ephemeral Docker/K8s agents
* Immutable artifacts
* Secrets via vaults

---

## 4. Kubernetes Networking (ADVANCED)

### Core Networking Model

* Every Pod has unique IP
* No NAT between pods
* Flat cluster network

---

### Networking Components

| Component     | Function                  |
| ------------- | ------------------------- |
| CNI           | Networking implementation |
| kube-proxy    | Service routing           |
| iptables/IPVS | Traffic forwarding        |

---

### Service Types

| Type         | Use Case         |
| ------------ | ---------------- |
| ClusterIP    | Internal access  |
| NodePort     | External (basic) |
| LoadBalancer | Cloud LB         |
| Ingress      | HTTP routing     |

---

> **Interview Gold:** Kubernetes defines networking rules; CNI plugins implement them.

---

## 5. AWS High Availability Architecture

### Typical Production Setup

```
Route53
   ↓
ALB (Multi‑AZ)
   ↓
Auto Scaling Group (EC2/EKS)
   ↓
RDS (Multi‑AZ)
   ↓
S3 + CloudFront
```

---

### Key AWS Concepts

* Stateless compute
* Auto scaling
* IAM roles (no access keys)
* Multi‑AZ by default

---

## 6. Terraform Architecture

### How Terraform Works

```
terraform plan
   ↓
Provider APIs
   ↓
State Comparison
   ↓
Execution Graph
   ↓
terraform apply
```

---

### Terraform State

| Concept        | Meaning                 |
| -------------- | ----------------------- |
| State file     | Source of truth         |
| Remote backend | Team collaboration      |
| Locking        | Prevent race conditions |

---

## 7. DevOps System Design (Senior Level)

### Common Design Topics

* CI/CD for microservices
* Zero downtime deployment
* Rollback strategies

---

### Deployment Strategies

| Strategy   | When to Use      |
| ---------- | ---------------- |
| Blue‑Green | Instant rollback |
| Canary     | Gradual risk     |
| Rolling    | Default K8s      |

---

## 8. Security Architecture (DevSecOps)

### Core Security Layers

| Layer  | Controls               |
| ------ | ---------------------- |
| Cloud  | IAM, Security Groups   |
| CI/CD  | Secrets, scanning      |
| K8s    | RBAC, NetworkPolicy    |
| Images | Vulnerability scanning |

---

## How to Use This README

1. Read section
2. Redraw diagram from memory
3. Explain flow out loud
4. Implement minimal lab

---

## Final Interview Mindset

> **Architecture questions test thinking, not tools.**

Explain:

* Why a component exists
* What problem it solves
* What happens when it fails

---

**End of README**
