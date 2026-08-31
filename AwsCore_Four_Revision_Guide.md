# AWS Core Four Revision Master Guide: EC2, LB, Lambda & IAM

> **Purpose:** A first-principles, deep-dive mental model for the 4 core AWS services. After reading this, you will be able to answer **any** variation, cross-question, or scenario thrown at you in an interview.

---

# 🖥️ MODULE 1: Amazon EC2 (Elastic Compute Cloud)

## 1. First-Principles Mental Model
EC2 is **virtualized compute** running on top of AWS Nitro (or Xen/KVM hypervisors). When you launch an EC2 instance, you are provisioning:
1. **CPU & RAM** (Instance Type family)
2. **Boot Disk** (EBS Volume or Instance Store)
3. **Network Card** (Elastic Network Interface - ENI with private/public IP)
4. **Firewall** (Security Group at the ENI level)

---

## 2. Instance Types & Families (The Cheat Code)
Remember the acronym **`C R M I T G`**:
* **`C` (Compute Optimized - `c5`, `c6g`):** High CPU-to-RAM ratio (2:1). Use for batch processing, media encoding, ML inference (CPU).
* **`R` (Memory Optimized - `r5`, `r6g`):** High RAM-to-CPU ratio (8:1). Use for In-memory caches (Redis), relational databases, real-time analytics.
* **`M` (General Purpose - `m5`, `m6i`):** Balanced CPU and RAM (4:1). Use for standard application servers, microservices, backend APIs.
* **`T` (Burstable - `t3`, `t4g`):** Baseline performance with **CPU Credits** for bursty workloads (Dev/staging, low-traffic web).
* **`I` (Storage Optimized - `i3`, `i3en`):** NVMe SSD storage attached directly. High IOPS (Cassandra, Elasticsearch, NoSQL).
* **`G / P` (Accelerated Computing - `g4dn`, `p4d`):** GPU instances for Deep Learning training, LLM serving.

---

## 3. Storage: EBS vs Instance Store vs EFS

| Feature | EBS (Elastic Block Store) | Instance Store (Ephemeral) | EFS (Elastic File System) |
|---|---|---|---|
| **Type** | Network-attached Block Storage | Physically attached NVMe SSD | Network-attached File System (NFS) |
| **Persistence** | **Persistent** (survives instance reboot/stop) | **Ephemeral** (data lost on instance STOP/terminate) | **Persistent** |
| **Multi-Attach** | Single instance (unless io1/io2 multi-attach in same AZ) | Single instance | **Shared across 1,000s of EC2s & EKS pods** |
| **Performance** | Up to 64k IOPS (`gp3`, `io2`) | **Ultra-low latency, millions of IOPS** | Scalable throughput (POSIX compliant) |
| **Use Case** | Boot volumes, databases | Temporary buffers, caches, scratch space | Shared web content, K8s RWX volumes |

### EBS Volume Types to Know:
* **`gp3` (Default standard):** Baseline 3,000 IOPS and 125 MB/s throughput independent of storage size.
* **`io2` / `io2 Block Express`:** Highest durability (99.999%) and up to 256,000 IOPS for mission-critical databases.

---

## 4. EC2 Pricing & Purchasing Models
1. **On-Demand:** Maximum flexibility, pay by the second. Highest price. Use for short-term, unpredictable workloads.
2. **Savings Plans / Reserved Instances (RI):** 1-year or 3-year commitment in exchange for up to **72% discount**. Use for steady-state production workloads.
3. **Spot Instances:** Bid on spare AWS compute capacity for up to **90% discount**. AWS can reclaim instances with a **2-minute termination notice**. Use for stateless batch jobs, CI/CD runners, EKS worker nodes (with Karpenter/Cluster Autoscaler).

---

## 5. Security Groups (SG) vs Network ACLs (NACL) — *Classic Interview Trap!*

| Feature | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| **Level** | **Instance / ENI level** (Virtual firewall) | **Subnet level** (Subnet boundary guard) |
| **State** | **Stateful** (Inbound return traffic allowed automatically) | **Stateless** (Must explicitly allow inbound & outbound) |
| **Rules** | **ALLOW rules only** (Everything else implicitly denied) | **ALLOW and DENY rules** (Evaluated in numbered order 1-32766) |
| **Evaluation** | Evaluates **all rules** before deciding | Evaluates in **sequential order** (stops at first match) |

---

# ⚖️ MODULE 2: AWS Load Balancers (ALB, NLB, GWLB)

## 1. First-Principles Mental Model
A Load Balancer distributes incoming network traffic across a group of backend targets (EC2, ECS, EKS Pods, Lambda). It acts as the single point of entry, providing **high availability, SSL termination, and traffic isolation**.

---

## 2. The 3 Types of Modern Elastic Load Balancers

### A. Application Load Balancer (ALB) — Layer 7
* **Understands:** HTTP, HTTPS, HTTP/2, gRPC, WebSockets.
* **Key Capabilities:**
  * **Layer 7 Request Routing:** Routes based on HTTP Path (`/api` $\rightarrow$ TargetGroup A), Host Header (`app.xyz.com`), Query parameters, HTTP Headers, or Source IP.
  * **Redirects & Fixed Responses:** Redirect HTTP $\rightarrow$ HTTPS automatically or return custom 403/503 JSON without hitting backend.
  * **Native WAF Integration:** Directly attach AWS WAF for Layer-7 DDoS, SQLi, and Bot Control.
  * **IP Mechanism:** Resolves to **dynamic IP addresses** managed by AWS DNS. Always reference ALB via its DNS name.

### B. Network Load Balancer (NLB) — Layer 4
* **Understands:** TCP, UDP, TLS.
* **Key Capabilities:**
  * **Ultra-High Performance:** Handles tens of millions of requests per second with **microsecond latency**.
  * **Static & Elastic IPs:** Provides one **Static IP (or Elastic IP)** per Availability Zone. Essential when client firewalls require strict IP whitelisting.
  * **TLS Passthrough:** Can forward encrypted TLS traffic directly to the backend without decrypting it on the load balancer (strict compliance/banking).
  * **Preserves Client IP:** Preserves source IP address natively to backend targets.

### C. Gateway Load Balancer (GWLB) — Layer 3/4
* **Use Case:** Deploy and scale 3rd-party virtual network appliances (Firewalls, Intrusion Detection Systems like Palo Alto, Fortinet) transparently using the **GENEVE protocol**.

---

## 3. Target Groups & Health Checks
* A Load Balancer routes traffic to **Target Groups**.
* **Target Types:** `instance` (EC2 ID), `ip` (Private IP or EKS Pod IP with AWS VPC CNI), `lambda` (ALB only), `alb` (NLB routing to ALB).
* **Health Check Parameters:**
  * `HealthCheckPath` (e.g., `/healthz` or `/api/status`)
  * `HealthyThresholdCount` (e.g., 2 consecutive 200 OKs)
  * `UnhealthyThresholdCount` (e.g., 3 consecutive failures)
  * `TimeoutSeconds` & `IntervalSeconds`
* If a target fails health checks, the LB immediately stops sending new requests to that instance/pod.

---

## 4. Advanced LB Concepts
* **Cross-Zone Load Balancing:** Distributes traffic evenly across all registered targets in *all* AZs, rather than just targets in the receiving AZ. (Default ON for ALB, optional for NLB).
* **Connection Draining (Deregistration Delay):** When an instance is removed, the LB gives in-flight requests a grace period (default 300s) to complete before terminating connections.
* **Sticky Sessions (Session Affinity):** Uses cookies (ALB-generated or application-generated) to bind a user's requests to the same target for stateful apps.

---

# ⚡ MODULE 3: AWS Lambda (Serverless Compute)

## 1. First-Principles Mental Model
AWS Lambda is **Function-as-a-Service (FaaS)**. You provide the code (or a container image up to 10GB), and AWS provisions microVMs (**Firecracker**) on demand, executes your function, and tears it down. You pay only for **invocations + compute duration (in milliseconds)**.

---

## 2. Hard Limits & Specifications (Memorize These!)
* **Max Execution Timeout:** **15 minutes** (900 seconds).
* **Memory:** Configurable from **128 MB to 10,240 MB (10 GB)**. (CPU, network bandwidth, and disk I/O scale linearly with memory allocation).
* **Ephemeral `/tmp` Storage:** 512 MB up to **10 GB**.
* **Deployment Package Size:** 50 MB (zipped direct upload), 250 MB (unzipped with layers), **10 GB (OCI Container Image)**.
* **Concurrent Executions:** Default account limit of **1,000 per region** (can be increased).

---

## 3. Invocation Models
1. **Synchronous Invocation (Request & Response):**
   * Caller waits for the function to execute and return output.
   * *Sources:* API Gateway, ALB, AWS CLI, SDK `RequestResponse`.
   * *Error Handling:* Caller is responsible for retries.
2. **Asynchronous Invocation (Fire & Forget):**
   * Event is placed in an internal AWS queue; Lambda returns `202 Accepted` immediately and processes in background.
   * *Sources:* Amazon S3, Amazon SNS, EventBridge, CloudWatch Logs.
   * *Error Handling:* Built-in **2 automatic retries** with exponential backoff, then sends failed events to **DLQ (Dead Letter Queue - SQS/SNS)** or **Lambda Destinations**.
3. **Event Source Mapping (Polling / Streaming):**
   * Lambda polls the stream or queue and reads records in batches.
   * *Sources:* Amazon SQS, Amazon Kinesis, Amazon DynamoDB Streams, Kafka/MSK.

---

## 4. Cold Starts vs Warm Starts & Mitigations
* **Cold Start:** When a new microVM container is initialized (downloading code, booting runtime, running global initialization code outside handler). Takes 100ms – 2s (longer if connecting to a VPC).
* **Warm Start:** Subsequent invocations reuse the already running container instance. Global variables and database connection pools are preserved.
* **Mitigation:** **Provisioned Concurrency** (keeps a pre-initialized pool of execution environments ready to respond instantly with zero cold start).

---

# 🔒 MODULE 4: AWS IAM (Identity and Access Management)

## 1. First-Principles Mental Model
IAM is the security backbone of AWS. It governs **Authentication (Who are you?)** and **Authorization (What are you allowed to do?)**.

---

## 2. The 4 Core IAM Entities
1. **IAM User:** A permanent person or service account with permanent long-term credentials (Console Password or Access Key / Secret Key).
2. **IAM Group:** A collection of IAM Users. Used to attach policies to multiple users at once (e.g., `DevOpsAdmins`, `Developers`). *Groups cannot be identified as principals in policies.*
3. **IAM Role:** An identity with **temporary security credentials** generated via AWS STS (Security Token Service). Used by:
   * AWS Services (EC2, Lambda, EKS via IRSA).
   * Cross-Account Access.
   * Identity Federation (SAML 2.0 / Okta / Azure AD / OIDC).
4. **IAM Policy:** A JSON document defining permissions (`Effect`, `Action`, `Resource`, `Condition`).

---

## 3. Policy Types & Evaluation Logic

### Anatomy of an IAM Policy JSON:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadWithCondition",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::company-data-prod",
        "arn:aws:s3:::company-data-prod/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" }
      }
    }
  ]
}
```

### The Golden Rule of AWS Policy Evaluation:
1. **Default:** Everything is **Implicit Deny**.
2. **Explicit Allow:** Overrides implicit deny.
3. **Explicit Deny:** **Trumps everything.** If any applicable policy (SCP, Permissions Boundary, Identity Policy, Resource Policy) has an explicit `Deny`, the request is blocked regardless of any `Allow`.

---

## 4. Roles: Trust Policy vs Permission Policy
An IAM Role has TWO distinct policies:
1. **Trust Policy (AssumeRolePolicyDocument):** Defines **WHO can assume this role** (The Principal).
   ```json
   {
     "Effect": "Allow",
     "Principal": { "Service": "ec2.amazonaws.com" },
     "Action": "sts:AssumeRole"
   }
   ```
2. **Permission Policy (Identity Policy):** Defines **WHAT the assumed role can do** on AWS resources (e.g., `s3:GetObject` on a specific bucket).

---

## 5. What is an Instance Profile?
* An **Instance Profile** is a dedicated container used exclusively to pass an **IAM Role to an EC2 instance**.
* EC2 does not accept IAM Roles directly; it only accepts an Instance Profile.
* When attached, the EC2 metadata service (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`) automatically dispenses short-lived STS credentials to applications running on the instance.

---

# 🧠 Fast Cross-Question Rapid Fire (Test Yourself!)

| Question | The 5-Second Pro Answer |
|---|---|
| *Can a Security Group block an IP?* | **No.** SGs are allow-only. You must block IPs at the **NACL level** or via **AWS WAF**. |
| *What happens to EBS data if EC2 is stopped?* | **Data persists.** Root volumes by default delete on *termination*, but survive instance *stop/start*. |
| *Can Lambda run for 30 minutes?* | **No.** Max timeout is 15 mins. For longer jobs, use **AWS Fargate (ECS/EKS)** or **AWS Batch / Step Functions**. |
| *How does an EC2 app get AWS credentials without hardcoded keys?* | Via an **IAM Role attached to an Instance Profile**, which dispenses STS tokens through the Instance Metadata Service (IMDSv2). |
| *How do you route `shop.com/api` to ECS and `shop.com/static` to S3?* | Use an **Application Load Balancer (ALB)** with path-based routing rules, or put **CloudFront** in front with multiple cache behaviors. |
| *Can NLB terminate SSL?* | **Yes.** NLB supports TLS termination using ACM certificates, but also supports raw TLS passthrough. |
