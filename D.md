## 🎙️ SECTION 1: Self-Introduction & Project Monologues

### 1. Master Self-Introduction (90–120 Seconds)
> "Hi, I’m **Mr X **. I’m a **Cloud DevOps & Backend Engineer** with over a year of production experience managing distributed microservices, AI inference workloads, and cloud infrastructure on **AWS and Linux**.
> 
> Currently at **iProtechs**, my daily responsibilities center on four key pillars:
> 1. **Infrastructure as Code & Cloud:** Provisioning and maintaining secure AWS infrastructure (EKS, VPC, IAM, ALB, S3) using modular **Terraform** with remote S3 backends and DynamoDB state locking.
> 2. **CI/CD & GitOps Automation:** Building zero-trust pipelines in **GitHub Actions** with **OIDC authentication** (eliminating static access keys) and implementing automated, self-healing deployments using **ArgoCD**.
> 3. **Production Reliability & Observability:** Setting up full-stack observability with **Prometheus, Grafana, Loki, and OpenTelemetry**, handling live incident triage (502/503 HTTP errors, PM2/pod crashes), and authoring post-incident Root Cause Analysis (RCA) reports.
> 4. **AI/ML Workload Deployments:** Successfully deployed conversational AI backends (LangGraph with Qdrant vector DB) and managed high-performance ONNX inference servers.
> 
> I hold a B.Tech in Electronics & Communication and was a **Smart India Hackathon National Finalist**. I’m excited about this opportunity at Deloitte because I want to bring my hands-on AWS, Kubernetes, Terraform, and reliability engineering skills to solve enterprise-scale challenges."

---

### 2. Current Project Deep Dive: Secure CI/CD & GitOps Platform on AWS EKS (2 Minutes)
> "In my current role, one of the major platforms I architected and maintain is our **Secure CI/CD & GitOps Platform on AWS EKS**.
> 
> * **The Problem:** Previously, infrastructure provisioning was manual, environments suffered from configuration drift between staging and prod, and CI/CD pipelines used long-lived static AWS access keys, creating security and audit risks.
> * **What I Engineered:**
>   * **Modular Terraform:** Built reusable Terraform modules for VPC (multi-AZ public/private subnets), EKS node groups, and IAM with DynamoDB distributed state locking. This cut cluster provisioning time from hours to **under 15 minutes (70%+ time reduction)**.
>   * **Zero-Trust CI/CD Security:** Integrated **GitHub Actions with AWS via OpenID Connect (OIDC)** and configured **IAM Roles for Service Accounts (IRSA)** inside Kubernetes, completely removing static credentials.
>   * **Automated GitOps with ArgoCD:** Implemented an **ArgoCD App-of-Apps pattern** with automated self-healing. Any change merged into the Git repository is automatically reconciled in the cluster, with **Semgrep (SAST)** and **Trivy container scanning** integrated into the pipeline to block vulnerable container images before deployment.
>   * **Observability & Reliability:** Hardened all Kubernetes workloads with resource limits, liveness/readiness probes, and Prometheus/Grafana alerting, achieving **60% p99 latency reduction** and zero pod starvation under peak loads.
> * **Business Outcome:** It gave our engineering team fully automated, zero-downtime releases with complete auditability, strict zero-trust security, and fast incident recovery."

---

## 🏛️ SECTION 2: Daily Operations & Org Infrastructure

### 3. Tell me your day-to-day activities as a DevOps engineer — like how your day begins?
* **Your Answer:**
  > "My day starts with a **15-minute operational health check**:
  > 1. **Monitoring & Alerts:** I check our Grafana dashboards and CloudWatch alerts for any overnight p99 latency spikes, pod crashloops, or 5xx API errors.
  > 2. **Daily Standup:** Sync with development squads on active sprint deliverables, upcoming deployment windows, and blocked dependencies.
  > 3. **Pipeline & GitOps Management:** Review PRs on our Terraform infrastructure repos and verify ArgoCD application sync states across dev/staging clusters.
  > 4. **Core Engineering Tasks:** Developing reusable Terraform modules, writing Python/Bash automation scripts, hardening container images with Trivy, and optimizing CI/CD build caches.
  > 5. **On-Call & Production Support:** If an incident or deployment failure occurs, I perform live triage, inspect container/journalctl logs, coordinate rollbacks if necessary, and document actionable Root Cause Analysis (RCA) reports."

---

### 4. How many EKS clusters are you handling?
* **Your Answer:**
  > "We manage **3 active EKS clusters**:
  > 1. **Non-Prod / Dev Cluster:** Used for active developer branch deployments, automated testing, and preview environments.
  > 2. **Staging / UAT Cluster:** An exact mirror of production used for performance load testing, database migration validation, and release candidate sign-offs.
  > 3. **Production Cluster:** Multi-AZ, highly available cluster hosting our core customer-facing APIs and background workers, managed via ArgoCD hub-and-spoke GitOps."

---

### 5. How many AWS accounts does your organization have?
* **Your Answer:**
  > "We follow the AWS Multi-Account best practice using **AWS Control Tower / AWS Organizations** with **4 core dedicated accounts**:
  > 1. **Management / Master Account:** Centralized billing, organization-level SCPs (Service Control Policies), and AWS SSO / IAM Identity Center.
  > 2. **Shared Services & Security Account:** Central ECR container registry, CI/CD runners (Jenkins/GitHub runner pools), GuardDuty, Security Hub, and centralized CloudTrail logging.
  > 3. **Non-Prod Account (Dev/Staging):** Isolated VPCs and compute resources for development and QA.
  > 4. **Production Account:** Strict access controls, multi-AZ VPC, production EKS clusters, and RDS/DocumentDB databases."

---

## ⚡ SECTION 3: AWS Lambda & Automation

### 6. Tell me what you know about Lambda functions
* **Your Answer:**
  > "AWS Lambda is a serverless, event-driven compute service that executes code in stateless containers in response to events (e.g., S3 object uploads, EventBridge cron triggers, API Gateway requests, SQS messages). 
  > * Key specs: Maximum execution timeout of **15 minutes**, ephemeral storage `/tmp` up to 10GB, memory configurable from 128 MB to 10 GB (CPU scales proportionally).
  > * Use cases: Operational automation (cleaning snapshots, auto-tagging), event processing, webhook handlers, and security compliance."

---

### 7. Tell me the workflow on how to deactivate IAM secret keys using Lambda functions
* **Your Answer:**
  > "We implement automated **IAM Access Key Rotation/Deactivation** via this 4-step workflow:
  > 1. **Trigger:** An **Amazon EventBridge Rule** runs on a scheduled cron (e.g., daily at midnight `cron(0 0 * * ? *)`).
  > 2. **Execution:** It invokes a Python Lambda function with an IAM Execution Role containing `iam:ListUsers`, `iam:ListAccessKeys`, and `iam:UpdateAccessKey` permissions.
  > 3. **Logic:** 
  >    * Lambda iterates over all IAM users via `iam.list_users()`.
  >    * For each user, calls `iam.list_access_keys()`, compares the `CreateDate` against `datetime.now()`.
  >    * If key age $> 90$ days and `Status == 'Active'`:
  >      * Sets status to Inactive via `iam.update_access_key(UserName=user, AccessKeyId=key_id, Status='Inactive')`.
  > 4. **Notification:** Lambda publishes an alert to an **Amazon SNS Topic** which sends a Slack/Email notification to the DevOps team with user details and key IDs."

---

### 8. What other Lambda functions do you know / have implemented?
* **Your Answer:**
  > 1. **EBS Snapshot & AMI Pruning:** Deletes unattached EBS snapshots and unmapped AMIs older than 30 days to optimize AWS cost.
  > 2. **EC2 Auto Start/Stop for Non-Prod:** Schedules non-production EC2/RDS instances to shut down at 8 PM and start at 8 AM on weekdays, saving ~60% compute cost.
  > 3. **GuardDuty / CloudTrail Security Alert Parser:** Parses high-severity GuardDuty JSON events and formats rich Slack webhook incident alerts.
  > 4. **S3 Object Processor:** Automatically triggers on S3 upload to scan files with an antivirus engine or generate thumbnails."

---

## ⚖️ SECTION 4: Compute, ASG & Load Balancing (ALB vs NLB)

### 9. If my application is in an EC2 instance and there is sudden traffic, how would you scale?
* **Your Answer:**
  > "1. **Immediate Vertical Scaling (Short-term):** If it's a standalone instance without an ASG, stop the instance, change instance type (e.g., from `t3.medium` to `c5.xlarge`), and restart. (Causes brief downtime).
  > 2. **Architectural / Production Solution (Horizontal Auto-Scaling):**
  >    * Put the EC2 instances behind an **Application Load Balancer (ALB)** inside an **Auto Scaling Group (ASG)** spanning multiple Availability Zones.
  >    * Attach a **Target Tracking Scaling Policy** (e.g., target 60% average CPU utilization or ALB `RequestCountPerTarget`).
  >    * When traffic spikes, the ASG launches additional instances automatically from a golden AMI / launch template and the ALB distributes traffic immediately."

---

### 10. Explain types of scaling policies in AWS ASG
* **Your Answer:**
  > 1. **Target Tracking Scaling:** You choose a metric (e.g., Average CPU = 60% or ALB Request Count = 1000/target), and AWS adjusts ASG capacity automatically to maintain that target (like a thermostat).
  > 2. **Step Scaling:** Scales in steps based on CloudWatch alarm thresholds (e.g., CPU 60–75% $\rightarrow$ add 1 instance; CPU 75–90% $\rightarrow$ add 3 instances; CPU $>90\%$ $\rightarrow$ add 5 instances).
  > 3. **Simple Scaling:** Scales by a fixed count when a single alarm triggers, but pauses for a **cooldown period** before evaluating again.
  > 4. **Scheduled Scaling:** Scales based on predictable time events (e.g., scale up to 10 instances on Friday 6 PM for an e-commerce flash sale, scale down Monday morning).
  > 5. **Predictive Scaling:** Uses machine learning models to forecast traffic patterns based on historical CloudWatch metrics and proactively provisions instances."

---

### 11. Let’s say there are 100 instances and my application is only on 2 particular instances. How would you scale them?
* **Your Answer:**
  > "You separate that application into its own **dedicated Auto Scaling Group and Target Group**:
  > 1. Create a distinct **Launch Template** specific to that application with its exact AMI and UserData.
  > 2. Create a dedicated **Target Group** on the Load Balancer and point the routing rule (path-based or host-based) to this target group.
  > 3. Create a dedicated **Auto Scaling Group** with Min=2, Desired=2, Max=10 attached to that target group.
  > 4. Apply a Target Tracking policy specifically to that ASG so only those 2 application instances scale up/down independently without affecting the other 98 instances in the account."

---

### 12. Difference between ALB and NLB
| Feature | Application Load Balancer (ALB) | Network Load Balancer (NLB) |
|---|---|---|
| **OSI Layer** | **Layer 7** (Application: HTTP, HTTPS, gRPC, WebSocket) | **Layer 4** (Transport: TCP, UDP, TLS) |
| **Routing Features** | Path-based (`/api`, `/users`), Host-based (`app.example.com`), Query params, HTTP headers | Port / IP based routing only |
| **Performance / Latency** | High throughput with sub-millisecond overhead (Evaluates HTTP headers) | Ultra-low latency (microsecond level), handles millions of requests/sec |
| **IP Addresses** | Dynamic IP addresses (Resolves via CNAME/DNS) | **Static IP** per AZ and supports Elastic IPs (EIP) |
| **SSL/TLS Handling** | SSL Termination, SNI, Redirects (HTTP to HTTPS) | SSL Termination OR direct **SSL/TLS Passthrough** (preserves client certs) |

---

### 13. If I have a shopping app, which LB would you choose?
* **Your Answer:**
  > "**Application Load Balancer (ALB)**. 
  > * **Reason:** A shopping application requires advanced Layer-7 routing features:
  >   * **Path-based routing:** `/cart` $\rightarrow$ Cart Service, `/products` $\rightarrow$ Catalog Service, `/checkout` $\rightarrow$ Payment Service.
  >   * **Host-based routing:** `m.shop.com` vs `shop.com`.
  >   * **Sticky Sessions (Cookie affinity):** Keeps user shopping sessions on the same instance.
  >   * **Native AWS WAF Integration:** Protects checkout endpoints against SQLi, XSS, and bot attacks."

---

### 14. If you have a banking app, which LB would you choose and why?
* **Your Answer:**
  > "It depends on the component:
  > * **For Core High-Throughput Financial & Payment Gateways (FIX protocol, Swift, TCP stream):** **Network Load Balancer (NLB)**. 
  >   * **Why:** Banking core transaction systems require **ultra-low latency**, **Static/Elastic IP whitelisting** on partner firewalls, and **End-to-End TLS Passthrough** so data is never decrypted at the load balancer level (PCI-DSS compliance).
  > * **For the Customer Banking Web/Mobile Portal UI:** **ALB** behind **AWS WAF** with strict TLS 1.3 encryption and IP rate-limiting."

---

### 15. Which has path-based routing?
* **Your Answer:**
  > "**Application Load Balancer (ALB)**. Layer 7 can inspect the HTTP request URI path (e.g., `rule: path-pattern == '/api/v1/*' -> forward to TargetGroup-API`)."

---

## 🛠️ SECTION 5: Terraform & Infrastructure as Code

### 16. Write Terraform code for creating an IAM user in AWS
```hcl
# 1. Create the IAM User
resource "aws_iam_user" "devops_user" {
  name = "srinivas-sarkar"
  path = "/system/"

  tags = {
    Environment = "Production"
    Role        = "DevOpsEngineer"
  }
}

# 2. Attach a Managed Policy (Read-Only EC2 example)
resource "aws_iam_user_policy_attachment" "user_attach" {
  user       = aws_iam_user.devops_user.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess"
}

# 3. Generate Access Keys (if programmatic access needed)
resource "aws_iam_access_key" "user_key" {
  user = aws_iam_user.devops_user.name
}

output "access_key_id" {
  value     = aws_iam_access_key.user_key.id
  sensitive = false
}
```

---

### 17. Write how you will call the IAM module?
```hcl
# Calling a reusable custom IAM module
module "iam_devops_team" {
  source = "./modules/iam-user" # or git::https://github.com/org/terraform-modules.git//iam

  username     = "devops-engineer"
  environment  = "Production"
  policy_arns  = [
    "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
  ]
  create_access_key = true
}
```

---

### 18. Why do you prefer storing statefiles in an S3 bucket, why not in a Git repository?
* **Your Answer:**
  > "Storing statefiles in Git is an **anti-pattern and major security vulnerability**:
  > 1. **Plaintext Secrets Exposure:** Terraform state (`terraform.tfstate`) stores sensitive outputs, DB passwords, and private keys in unencrypted plaintext JSON. Committing to Git exposes these to everyone with repo access.
  > 2. **State Locking (Concurrency):** Git has no native locking mechanism. If two engineers run `terraform apply` concurrently, state files become corrupted. S3 + **DynamoDB (`LockID`)** ensures strict distributed state locking.
  > 3. **Race Conditions & Merge Conflicts:** Merging JSON diffs in Git will break Terraform's resource dependency tree.
  > 4. **Data Security & Versioning:** S3 provides native **Server-Side Encryption (AES-256/KMS)**, **Bucket Versioning** (to roll back if state is corrupted), and IAM least-privilege bucket policies."

---

### 19. Some resources were created using CFT (CloudFormation), but the team wants to manage them using Terraform. How can you do it?
* **Your Answer:**
  > "We use **Terraform Import (`terraform import` or Terraform 1.5+ `import` blocks)**:
  > 1. **Step 1:** Write the equivalent resource block in Terraform code matching the CloudFormation resource configuration.
  > 2. **Step 2:** Run `terraform import <resource_type>.<resource_name> <aws_resource_id>`. (e.g., `terraform import aws_s3_bucket.my_bucket production-app-bucket-2026`).
  > 3. **Step 3:** Run `terraform plan` to verify that the generated state matches your code with **0 changes, 0 to add, 0 to destroy**.
  > 4. **Step 4 (In CloudFormation):** Set the CloudFormation stack **DeletionPolicy to `Retain`**, then safely delete or decouple the CFT stack so it doesn't accidentally delete the live AWS resources."

---

## ☸️ SECTION 6: Kubernetes & Secrets Management

### 20. Describe any Kubernetes issue you have faced recently
* **Your Answer:**
  > "Recently, after rolling out a new release for our AI backend service, pods entered a **`CrashLoopBackOff` with exit code 137 (OOMKilled)**.
  > 
  > **Investigation:** 
  > 1. Ran `kubectl describe pod <pod_name>` and inspected `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.
  > 2. Checked container resource specs: memory request was 512Mi, memory limit was 1Gi.
  > 3. Checked Prometheus memory metrics and noticed that when multiple client connections initiated ONNX embedding calculations, memory consumption spiked to 1.4Gi, triggering the Linux Kernel cgroup OOMKiller.
  > 
  > **Resolution:** 
  > * Tuned the batch processing size in the Node.js runtime to stream batches instead of buffering large payloads in memory.
  > * Adjusted the pod resource requests to 1Gi and limits to 2Gi, with HPA scaling out replicas based on memory threshold at 75%."

---

### 21. How do you debug logs in Kubernetes?
* **Your Answer:**
  > "1. **Current Container Logs:** `kubectl logs <pod_name> -n <namespace> -c <container_name>` (use `-f` to follow).
  > 2. **Previous Crashed Container Logs:** If a container crashed and restarted: `kubectl logs <pod_name> --previous`.
  > 3. **Events & State:** `kubectl describe pod <pod_name>` to inspect Kubelet events (e.g., ImagePullBackOff, failed probes, mounting issues).
  > 4. **Centralized Log Aggregation:** In production, we query **Grafana Loki via LogQL** using `{namespace='prod', app='api_gateway'} |= 'error'` to aggregate logs across all replicas and nodes simultaneously.
  > 5. **Ephemeral Debug Containers:** For containers with scratch/distroless images: `kubectl debug -it <pod_name> --image=busybox --target=<container_name>`."

---

### 22. Where will you store secrets?
* **Your Answer:**
  > "Secrets should never be hardcoded in Git or plain ConfigMaps. We store secrets in **AWS Secrets Manager** or **HashiCorp Vault**. 
  > Within Kubernetes, we project these secrets into pods using the **External Secrets Operator (ESO)** or the **AWS Secrets Store CSI Driver**, which syncs secrets securely into native Kubernetes Secret objects or mounts them directly as read-only in-memory volumes (`tmpfs`)."

---

### 23. If you store your secrets in AWS Secrets Manager, how will your app know that your secrets are stored in AWS Secrets Manager vs HashiCorp Vault?
* **Your Answer:**
  > "The application **does not and should not know where secrets are stored** — this follows the **12-Factor App design pattern**:
  > 1. **Decoupled Architecture:** The infrastructure layer (External Secrets Operator / CSI Driver) authenticates with AWS Secrets Manager (via IRSA) or Vault (via Vault Kubernetes Auth).
  > 2. **Injection as Environment Variables or Files:** ESO pulls the secret and injects it into the Pod as standard environment variables (`process.env.DB_PASSWORD`) or as a mounted file in `/etc/secrets/db-creds.json`.
  > 3. **Application Independence:** The application code simply reads standard environment variables, completely agnostic of whether the backend secret store is AWS Secrets Manager, Vault, or GCP Secret Manager."

---

## 🔒 SECTION 7: IAM, Cross-Region S3, ASG & CI/CD Validation

### 24. S3 bucket is in Region A (us-east-1) and EC2 instance is in Region B (us-west-2). How do you provide access to the EC2 instance for this S3 bucket?
* **Your Answer:**
  > "1. **IAM Role & Instance Profile:** Attach an IAM Role to the EC2 instance with an IAM Policy granting `s3:GetObject`, `s3:PutObject`, `s3:ListBucket` on `arn:aws:s3:::region-a-bucket-name/*`. S3 is a global namespace, so IAM permissions work globally across regions without VPC peering.
  > 2. **S3 Bucket Policy:** Ensure the S3 bucket policy allows the IAM Role's ARN and does not have an explicit `Deny` based on source region.
  > 3. **Networking & Security Consideration:** Traffic will flow over the AWS global network backbone. To optimize security and prevent traffic traversing the public internet, configure cross-region networking or access via AWS S3 endpoint."

---

### 25. What is an Instance Profile?
* **Your Answer:**
  > "An **Instance Profile** is an AWS IAM container that passes an IAM Role to an Amazon EC2 instance at launch time. 
  > * In the AWS Management Console, when you attach a role to an EC2 instance, the instance profile is created automatically with the same name.
  > * In Terraform or CloudFormation, you must explicitly declare an `aws_iam_instance_profile` resource that references the `aws_iam_role`, and then pass the instance profile name to the `aws_instance` or `aws_launch_template`."

---

### 26. When working on code, we can't directly push to main branch. How can we get this code tested and validated using Jenkins before merging it to main branch?
* **Your Answer:**
  > "We enforce a **Pull Request (PR) Verification CI Pipeline** using **Branch Protection Rules & Webhooks**:
  > 1. **GitHub/GitLab Branch Protection:** Enable branch protection on `main` to disallow direct pushes and require *'Status checks to pass before merging'*.
  > 2. **Webhook Trigger:** When a developer raises a PR against `main`, GitHub fires a webhook event to Jenkins (using the *GitHub Branch Source / Multibranch Pipeline plugin*).
  > 3. **Jenkins PR Pipeline Execution:**
  >    * Checks out the PR branch.
  >    * Runs Linters & Unit Tests.
  >    * Runs SAST & Security Scans (SonarQube, Trivy, Semgrep).
  >    * Builds the container image and runs integration tests in an ephemeral container.
  > 4. **Status Reporting:** Jenkins reports the build status back to GitHub via the GitHub Commit Status API (`SUCCESS` or `FAILURE`).
  > 5. **Merge Condition:** The PR merge button is only unlocked once Jenkins reports a green tick AND at least 1 peer approval is recorded."
