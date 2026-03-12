# kai: I Built a Local AI Tool That Debugs Kubernetes Pods Automatically

> Version 4 — final version before building

---

## Before We Start — What You're Reading

This document has three parts:

- **The Blog** — full story + tutorial, ready to publish on Medium or Dev.to
- **The Code** — every file, complete and ready to copy
- **The LinkedIn Post** — two options with a screenshot checklist

Read it top to bottom once. Then build it.

---

---

# PART A — THE BLOG

---

## kai: I Built a Local AI Tool That Debugs Kubernetes Pods Automatically

*No API keys. No cloud. Just a terminal, a local LLM, and a Saturday afternoon.*

---

### The Problem

You've been here before.

A pod is crashing in your cluster. You run `kubectl describe pod nginx`. Wall of text. You scroll. You find something that looks like the error. You copy it. You open a new tab. You paste it into ChatGPT. You wait. You get an answer. You go back to the terminal. You try the fix. It doesn't work. You repeat.

That loop is exhausting. And the worst part? All the data kubectl needs to diagnose the problem is *already there*. It's sitting in your logs, your events, your describe output. You just need something to *read it and think*.

So I built that thing. I called it `kai` — kubectl AI. It's a CLI tool that:

1. Collects the relevant Kubernetes data automatically
2. Sends it to a local LLM running on your machine
3. Returns a plain English diagnosis with the exact fix command

The entire thing runs locally. No internet. No API keys. No cost per token.

Here's what it looks like:

```bash
kai diagnose pod nginx-broken-image-abc123
```

```
→ Checking Ollama connection...
→ Collecting describe output...
→ Collecting logs...
→ Collecting events...
⠋ AI analyzing pod...

╭─────────────────────────────────────────────────────╮
│  KAI — Pod Diagnosis: nginx-broken-image-abc123     │
│                                                     │
│  Problem:                                           │
│    ImagePullBackOff — the image                     │
│    nginx:this-tag-does-not-exist does not exist     │
│    in the registry.                                 │
│                                                     │
│  Why it happens:                                    │
│    Kubernetes tried to pull the image, received a   │
│    404 from Docker Hub, and is now backing off      │
│    with increasing wait times before retrying.      │
│                                                     │
│  Fix command:                                       │
│    kubectl set image deployment/nginx-broken-image  │
│      nginx=nginx:latest                             │
╰─────────────────────────────────────────────────────╯
```

And if you just want kubectl output explained in plain English without a full diagnosis:

```bash
kai explain describe pod nginx-broken-image-abc123
```

```
╭─────────────────────────────────────────────╮
│  KAI — Describe Explanation                 │
│                                             │
│  What this output shows:                    │
│    This pod is stuck trying to pull its     │
│    container image. The image tag you       │
│    specified doesn't exist...               │
╰─────────────────────────────────────────────╯
```

And because it's also a proper kubectl plugin, this works too:

```bash
kubectl ai diagnose pod nginx-broken-image-abc123
```

Same tool. Two ways to call it. Let me show you exactly how to build it from scratch.

---

### The Stack

Everything in this project is free and open source:

| Tool | What it does | Why we use it |
|---|---|---|
| **Python** | The glue | Runs everything |
| **Typer** | CLI framework | Turns functions into commands |
| **Rich** | Terminal formatting | Colors, panels, spinners |
| **Ollama** | Local LLM runtime | Runs AI on your machine |
| **Mistral 7B** | The AI model | Fast, smart, fits in 5GB RAM |
| **Kind** | Local Kubernetes | Cluster in Docker for testing |
| **kubectl** | Kubernetes CLI | Collects the data |

Your machine needs: **8GB+ RAM** (15GB is comfortable), Python 3.9+, Docker.

---

### How AI Actually Helps Here

Before writing a single line of code, it's worth understanding *why* AI is useful for Kubernetes debugging — not just that it is.

Kubernetes failures are repetitive. The same errors appear over and over:

```
ImagePullBackOff     — wrong image tag, missing registry credentials
CrashLoopBackOff     — container exits immediately, bad command, missing config
OOMKilled            — container exceeded its memory limit
Pending              — no node has enough resources to schedule it
Probe failures       — app started but health check fails
```

Each of these has a signature. `ImagePullBackOff` always shows up in events as `Failed to pull image`. `OOMKilled` always appears in `kubectl describe` under `Last State`. `CrashLoopBackOff` always has exit code 1 in the logs.

LLMs are extremely good at **pattern matching on text**. When you feed Mistral the output of `kubectl describe` + `kubectl logs` + `kubectl get events`, it's reading the same information an experienced SRE reads — and it has seen thousands of examples of these patterns in its training data.

The structured prompt we write tells it: *here is the data, here is the format I want back*. That combination — real data plus a forced output format — is what makes the diagnosis reliable.

**AI isn't magic here. It's pattern matching on data you already had.**

That sentence is the whole insight. Keep it in mind as you build.

---

### Why subprocess Instead of the Kubernetes Python Client

This is a deliberate design decision worth explaining, because it comes up in every project that automates kubectl.

Python has an official Kubernetes client library (`kubernetes`). You can install it and query the API directly. So why didn't we use it?

Four reasons:

**1. kubectl is already installed everywhere.** Any machine running Kubernetes has kubectl. Adding a Python library adds a dependency that could have version conflicts, auth issues, and kubeconfig quirks to debug.

**2. The output matches what engineers actually see.** When an SRE debugs a pod, they run `kubectl describe`. Mistral was trained on text — including documentation, Stack Overflow answers, and blog posts that quote `kubectl describe` output. Feeding it the same format humans use means it recognizes patterns more reliably.

**3. No extra dependencies.** `subprocess` is part of Python's standard library. Zero install required.

**4. Simpler code for a CLI tool.** The Kubernetes client returns structured Python objects. We'd have to convert them back to readable strings to put in a prompt anyway. We'd be doing extra work to end up with the same result.

The rule of thumb: use the Kubernetes Python client when you're building something that needs to *act* on the cluster programmatically (create resources, patch deployments). Use subprocess + kubectl when you're building something that needs to *read and explain* the cluster to a human or an AI.

---

### How The Pieces Connect

```
You type:  kai diagnose pod nginx
                    ↓
         cli.py checks Ollama is running
         (fails fast with clear message if not)
                    ↓
         k8s.py runs kubectl commands
         (describe, logs, events) with 30s timeout
                    ↓
         ai.py truncates data if too large,
         builds a structured prompt,
         sends to Ollama on localhost:11434
                    ↓
         Mistral 7B reads the prompt
         returns diagnosis in forced format
                    ↓
         cli.py shows spinner, prints Rich panel
```

Three files. Three jobs. No file does more than one thing.

---

### Step 1 — Install and Verify Your Tools

Install everything and verify it before writing a single line of project code. This is how you avoid debugging the environment instead of the project.

#### Python Environment

```bash
python --version
```

You need 3.9 or higher. If you're on Ubuntu and only have 3.8:

```bash
sudo apt update
sudo apt install python3.10 python3.10-venv python3-pip -y
```

Create a clean virtual environment:

```bash
mkdir kai && cd kai
python3 -m venv venv
source venv/bin/activate
```

**What just happened:** `venv` creates an isolated Python environment. Any package you install goes only into this folder, not your whole system. You'll see `(venv)` in your terminal prompt. Every time you come back to this project, run `source venv/bin/activate` first.

Install the dependencies:

```bash
pip install typer rich requests
```

- `typer` — turns Python functions into CLI commands automatically
- `rich` — colors, panels, spinners in the terminal
- `requests` — HTTP calls to Ollama's API

#### Ollama

Ollama is the runtime that lets you run AI models locally. Think of it like Docker, but for LLMs.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

When installed on Ubuntu with this script, Ollama registers as a **systemd service** and runs in the background automatically.

Check if it's running:

```bash
sudo systemctl status ollama
```

If it's not:

```bash
sudo systemctl start ollama
```

Now pull the Mistral model:

```bash
ollama pull mistral
```

This downloads ~4.1GB. Get a coffee.

**Why Mistral 7B?** 7 billion parameters — big enough to reason well about technical problems, small enough to run on 8GB RAM. On a 15GB machine it runs comfortably without touching swap.

Verify it works:

```bash
ollama run mistral "What is a Kubernetes pod? Answer in 2 sentences."
```

> **Note on first run:** The very first Mistral response may take **30–40 seconds** while the model loads from disk into RAM. This is normal. Subsequent requests are 10–20 seconds.

#### Docker

```bash
docker --version && docker ps
```

If Docker isn't running:

```bash
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

#### Kind — Local Kubernetes

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind --version
```

#### kubectl

```bash
kubectl version --client
```

If not installed:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Final Verification

```bash
echo "=== Environment Check ===" && \
python --version && \
docker --version && \
kind --version && \
kubectl version --client --short 2>/dev/null && \
ollama --version && \
echo "=== All good ==="
```

Expected:

```
=== Environment Check ===
Python 3.10.x
Docker version 24.x.x
kind v0.23.0
Client Version: v1.29.x
ollama version 0.x.x
=== All good ===
```

**Screenshot this.** It's your proof-of-setup for the blog.

---

### Step 2 — Create the Kubernetes Lab

#### Create the Cluster

`kind-cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-cluster.yaml --name kai-lab
kubectl get nodes
```

```
NAME                    STATUS   ROLES           AGE   VERSION
kai-lab-control-plane   Ready    control-plane   2m    v1.29.x
kai-lab-worker          Ready    <none>          2m    v1.29.x
kai-lab-worker2         Ready    <none>          2m    v1.29.x
```

#### Deploy Broken Apps

**Scenario 1 — ImagePullBackOff** (`broken-image.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-broken-image
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-broken-image
  template:
    metadata:
      labels:
        app: nginx-broken-image
    spec:
      containers:
      - name: nginx
        image: nginx:this-tag-does-not-exist
```

**Scenario 2 — CrashLoop** (`crashloop.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-crashloop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-crashloop
  template:
    metadata:
      labels:
        app: nginx-crashloop
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        command: ["sh", "-c", "echo starting && exit 1"]
```

**Scenario 3 — OOMKilled** (`oom.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-oom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-oom
  template:
    metadata:
      labels:
        app: nginx-oom
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            memory: "1Mi"
```

```bash
kubectl apply -f broken-image.yaml
kubectl apply -f crashloop.yaml
kubectl apply -f oom.yaml
```

Wait 30 seconds:

```bash
kubectl get pods
```

```
NAME                                  READY   STATUS             RESTARTS
nginx-broken-image-xxx                0/1     ImagePullBackOff   0
nginx-crashloop-xxx                   0/1     CrashLoopBackOff   4
nginx-oom-xxx                         0/1     OOMKilled          3
```

Three broken pods. **Screenshot this.**

---

### Step 3 — Build the Project

Final file structure:

```
kai/
 ├── venv/
 ├── k8s.py             ← collects kubectl data
 ├── ai.py              ← talks to Ollama
 ├── cli.py             ← the commands you type
 ├── kai                ← entry point (kai command)
 ├── kubectl-ai         ← entry point (kubectl ai command)
 ├── requirements.txt
 ├── .gitignore
 ├── LICENSE
 ├── kind-cluster.yaml
 ├── broken-image.yaml
 ├── crashloop.yaml
 └── oom.yaml
```

---

#### File 1 — k8s.py

```python
# k8s.py
# Runs kubectl commands and returns their output as plain text.
# One function per data type. No classes. No parsing.

import subprocess


def run_command(command):
    """
    Runs a shell command and returns output as a string.
    timeout=30 prevents hanging forever if the cluster connection drops.
    """
    try:
        result = subprocess.run(
            command,
            capture_output=True,
            text=True,
            timeout=30
        )
        if result.returncode == 0:
            return result.stdout
        else:
            return f"Error: {result.stderr}"

    except subprocess.TimeoutExpired:
        return "Error: command timed out after 30s — is your cluster reachable? Try: kubectl get nodes"


def get_pod_describe(pod_name, namespace="default"):
    """
    kubectl describe pod <pod_name>
    Most information-rich command for pod diagnosis.
    Shows image, restart count, resource limits, conditions, and events.
    """
    return run_command(["kubectl", "describe", "pod", pod_name, "-n", namespace])


def get_pod_logs(pod_name, namespace="default"):
    """
    Gets logs from the pod, trying --previous first.
    --previous = logs from the last crashed container run.
    That's where the actual error message usually lives.
    """
    logs = run_command(["kubectl", "logs", pod_name, "--previous", "-n", namespace])

    if "Error" in logs or logs.strip() == "":
        logs = run_command(["kubectl", "logs", pod_name, "-n", namespace])

    return logs if logs.strip() else "No logs available for this pod."


def get_pod_events(pod_name, namespace="default"):
    """
    Gets events filtered to only those mentioning this specific pod.
    Filtering by pod_name only (not "Warning") prevents unrelated
    cluster warnings from polluting the AI's context.
    """
    all_events = run_command([
        "kubectl", "get", "events",
        "-n", namespace,
        "--sort-by=.lastTimestamp"
    ])

    relevant_lines = [
        line for line in all_events.splitlines()
        if pod_name in line
    ]

    return "\n".join(relevant_lines) if relevant_lines else "No events found for this pod."


def get_deployment_describe(deployment_name, namespace="default"):
    """kubectl describe deployment <name>"""
    return run_command(["kubectl", "describe", "deployment", deployment_name, "-n", namespace])


def get_deployment_pods(deployment_name, namespace="default"):
    """
    Gets all pods in the namespace and lets the AI find the relevant ones.
    We don't filter by label selector because deployment labels vary wildly
    in real clusters (app=X vs app.kubernetes.io/name=X etc).
    Safer to give the AI the full list — it matches by deployment name
    from the describe output it already has.
    """
    return run_command([
        "kubectl", "get", "pods",
        "-n", namespace,
        "-o", "wide"
    ])


def get_rollout_status(deployment_name, namespace="default"):
    """kubectl rollout status deployment/<name>"""
    return run_command([
        "kubectl", "rollout", "status",
        f"deployment/{deployment_name}",
        "-n", namespace
    ])


def get_cluster_nodes():
    """All nodes and their status."""
    return run_command(["kubectl", "get", "nodes", "-o", "wide"])


def get_all_pods():
    """All pods across all namespaces."""
    return run_command(["kubectl", "get", "pods", "-A", "-o", "wide"])


def get_cluster_events():
    """
    Warning events across the cluster, sorted by time.
    Normal events (Scheduled, Pulled, Started) are noise.
    Warnings are where problems live.
    """
    return run_command([
        "kubectl", "get", "events",
        "-A",
        "--field-selector", "type=Warning",
        "--sort-by=.lastTimestamp"
    ])


def is_cluster_reachable():
    """
    Quick check used by 'kai doctor'.
    Returns True if kubectl can reach the cluster, False if not.
    """
    result = run_command(["kubectl", "get", "nodes", "--request-timeout=5s"])
    return "Error" not in result and result.strip() != ""
```

---

#### File 2 — ai.py

```python
# ai.py
# Talks to Ollama's HTTP API.
# Build prompt → truncate if needed → send → return response.

import requests

OLLAMA_URL = "http://localhost:11434/api/generate"
OLLAMA_HEALTH_URL = "http://localhost:11434"
MODEL = "mistral"

# Safety limit: Mistral has a context window limit.
# Huge logs (thousands of lines) slow responses and can overflow the context.
# 12000 characters is roughly 3000 tokens — plenty for diagnosis.
MAX_CHARS = 12000


def truncate(text):
    """
    Truncates text to MAX_CHARS if it's too long.
    Adds a [truncated] marker so the AI knows it's seeing partial data.

    Why does this matter?
    If someone runs 'kai logs huge-pod' and the logs are 50,000 lines,
    sending the full thing to Mistral would be very slow and might
    exceed the model's context window entirely. We cut it safely.
    """
    if len(text) > MAX_CHARS:
        return text[:MAX_CHARS] + "\n\n[output truncated — showing first 12000 characters]"
    return text


def check_ollama():
    """
    Checks if Ollama is reachable.
    Called before every command so we fail fast with a clear message
    instead of waiting 120s for a timeout deep in the code.
    """
    try:
        requests.get(OLLAMA_HEALTH_URL, timeout=2)
        return True
    except Exception:
        return False


def check_model():
    """
    Checks if the mistral model is available in Ollama.
    Returns True if available, False if it needs to be pulled.
    """
    try:
        response = requests.get("http://localhost:11434/api/tags", timeout=5)
        models = response.json().get("models", [])
        return any("mistral" in m.get("name", "") for m in models)
    except Exception:
        return False


def ask_ollama(prompt):
    """
    Sends a prompt to Ollama and returns the full response as a string.
    stream=False = wait for complete response (simpler for CLI tools).
    """
    payload = {
        "model": MODEL,
        "prompt": prompt,
        "stream": False
    }

    try:
        response = requests.post(OLLAMA_URL, json=payload, timeout=120)
        response.raise_for_status()
        return response.json()["response"]

    except requests.exceptions.ConnectionError:
        return (
            "Error: Cannot connect to Ollama.\n"
            "Check status: sudo systemctl status ollama\n"
            "Start it:     sudo systemctl start ollama"
        )
    except requests.exceptions.Timeout:
        return "Error: Ollama timed out. Model may still be loading — try again in 30 seconds."
    except Exception as e:
        # Check if it's a missing model error
        if "model" in str(e).lower() or "not found" in str(e).lower():
            return (
                f"Error: Model '{MODEL}' not found in Ollama.\n\n"
                f"Run this to install it:\n"
                f"  ollama pull {MODEL}"
            )
        return f"Unexpected error: {str(e)}"


def build_pod_prompt(pod_name, describe_output, logs_output, events_output):
    """
    Builds the pod diagnosis prompt with forced output format.

    "senior Kubernetes SRE debugging production clusters" — this framing
    makes the model respond with more confidence and specificity.
    Models mirror the authority level you give them in the system role.

    Forced format (Problem / Why / Fix) ensures every response is
    scannable and consistent — no essays, no questions back.
    """
    return f"""You are a senior Kubernetes SRE debugging production clusters. Be concise and specific.

Diagnose this pod and respond ONLY in this exact format:

Problem:
[one sentence — name the exact error: ImagePullBackOff, CrashLoopBackOff, OOMKilled, etc.]

Why it happens:
[2-3 sentences of root cause in plain English]

Fix command:
[exact kubectl command or minimal YAML change to resolve it]

---

Pod name: {pod_name}

kubectl describe pod:
{truncate(describe_output)}

Pod logs:
{truncate(logs_output)}

Pod events:
{truncate(events_output)}
"""


def build_deployment_prompt(deployment_name, describe_output, pods_output, rollout_output):
    return f"""You are a senior Kubernetes SRE debugging production clusters. Be concise and specific.

Diagnose this deployment and respond ONLY in this exact format:

Problem:
[one sentence — healthy or what is wrong]

Why it happens:
[2-3 sentences referencing specific fields from the data below]

Fix command:
[exact kubectl command or YAML change]

---

Deployment: {deployment_name}

kubectl describe deployment:
{truncate(describe_output)}

Pods in namespace:
{truncate(pods_output)}

Rollout status:
{truncate(rollout_output)}
"""


def build_logs_prompt(pod_name, logs_output):
    return f"""You are a senior Kubernetes SRE debugging production clusters. Be concise and specific.

Explain these logs and respond ONLY in this exact format:

What is happening:
[plain English summary of what the logs show]

Is anything wrong:
[yes or no — if yes, what specifically and why]

Next step:
[one concrete action to take]

---

Pod name: {pod_name}

Logs:
{truncate(logs_output)}
"""


def build_cluster_prompt(nodes_output, pods_output, events_output):
    return f"""You are a senior Kubernetes SRE debugging production clusters. Be concise and specific.

Analyze this cluster and respond ONLY in this exact format:

Node health:
[are all nodes Ready? list any that aren't]

Problem pods:
[each non-Running/non-Completed pod with its issue — or "None"]

Event patterns:
[repeated warnings or patterns — or "None"]

Top 3 fixes (priority order):
1. [most urgent]
2. [second]
3. [third]

---

Nodes:
{truncate(nodes_output)}

All pods:
{truncate(pods_output)}

Warning events:
{truncate(events_output)}
"""


def build_explain_describe_prompt(resource_type, name, describe_output):
    """
    Explains raw kubectl describe output in plain English.
    No diagnosis, no fix — just "what am I looking at?"

    This is useful for anyone learning Kubernetes who wants to understand
    what kubectl describe is actually telling them, field by field.
    """
    return f"""You are a senior Kubernetes SRE and a patient teacher.

A developer has run 'kubectl describe {resource_type} {name}' and wants to understand what the output means.

Explain this output in plain English. Respond ONLY in this exact format:

What this shows:
[2-3 sentence overview of what this resource is and its current state]

Key fields to notice:
[explain 3-5 of the most important fields in the output — what they mean and why they matter]

Anything concerning:
[flag anything that looks wrong or unusual — or "Nothing concerning" if it all looks healthy]

---

kubectl describe {resource_type} {name}:
{truncate(describe_output)}
"""


# Functions cli.py calls — one per command

def diagnose_pod(pod_name, describe, logs, events):
    return ask_ollama(build_pod_prompt(pod_name, describe, logs, events))

def diagnose_deployment(deployment_name, describe, pods, rollout):
    return ask_ollama(build_deployment_prompt(deployment_name, describe, pods, rollout))

def explain_logs(pod_name, logs):
    return ask_ollama(build_logs_prompt(pod_name, logs))

def analyze_cluster(nodes, pods, events):
    return ask_ollama(build_cluster_prompt(nodes, pods, events))

def explain_describe(resource_type, name, describe_output):
    return ask_ollama(build_explain_describe_prompt(resource_type, name, describe_output))
```

---

#### File 3 — cli.py

```python
# cli.py
# Defines CLI commands using Typer.
# Each @app.command() function = one command you can type.

import typer
from rich.console import Console
from rich.panel import Panel
from rich.table import Table

import k8s
import ai

app = typer.Typer(
    name="kai",
    help="kubectl AI — local AI assistant for Kubernetes debugging",
    add_completion=False
)

console = Console()


def check_connection():
    """
    Runs before every command that calls the AI.
    Fails fast with a clear message if Ollama isn't running.
    """
    if not ai.check_ollama():
        console.print("[red]✗ Ollama is not running.[/red]")
        console.print("  Start it: [yellow]sudo systemctl start ollama[/yellow]")
        raise typer.Exit(1)

    if not ai.check_model():
        console.print(f"[red]✗ Mistral model not found in Ollama.[/red]")
        console.print("  Install it: [yellow]ollama pull mistral[/yellow]")
        raise typer.Exit(1)


def print_diagnosis(title, content):
    """Prints AI response in a cyan bordered panel."""
    console.print()
    console.print(Panel(
        content,
        title=f"[bold cyan] KAI — {title}[/bold cyan]",
        border_style="cyan",
        padding=(1, 2)
    ))
    console.print()


def print_status(message):
    """Yellow arrow status line during data collection."""
    console.print(f"[yellow]→[/yellow] {message}")


@app.command()
def diagnose(
    resource_type: str = typer.Argument(..., help="Resource type (pod or deployment)"),
    name: str = typer.Argument(..., help="Name of the pod or deployment"),
    namespace: str = typer.Option("default", "--namespace", "-n", help="Kubernetes namespace")
):
    """
    Diagnose a failing pod or deployment.

    Examples:
        kai diagnose pod nginx-abc123
        kai diagnose deployment nginx-deploy
        kai diagnose pod nginx --namespace staging
    """
    check_connection()

    if resource_type.lower() == "pod":
        console.print(f"\n[bold]Diagnosing pod:[/bold] [cyan]{name}[/cyan]")
        print_status("Collecting describe output...")
        describe = k8s.get_pod_describe(name, namespace)
        print_status("Collecting logs...")
        logs = k8s.get_pod_logs(name, namespace)
        print_status("Collecting events...")
        events = k8s.get_pod_events(name, namespace)

        with console.status("[bold green]AI analyzing pod...[/bold green]"):
            result = ai.diagnose_pod(name, describe, logs, events)
        print_diagnosis(f"Pod Diagnosis: {name}", result)

    elif resource_type.lower() == "deployment":
        console.print(f"\n[bold]Diagnosing deployment:[/bold] [cyan]{name}[/cyan]")
        print_status("Collecting describe output...")
        describe = k8s.get_deployment_describe(name, namespace)
        print_status("Collecting pod status...")
        pods = k8s.get_deployment_pods(name, namespace)
        print_status("Checking rollout status...")
        rollout = k8s.get_rollout_status(name, namespace)

        with console.status("[bold green]AI analyzing deployment...[/bold green]"):
            result = ai.diagnose_deployment(name, describe, pods, rollout)
        print_diagnosis(f"Deployment Diagnosis: {name}", result)

    else:
        console.print(f"[red]Unknown resource type:[/red] {resource_type}")
        console.print("Use: pod or deployment")
        raise typer.Exit(1)


@app.command()
def logs(
    name: str = typer.Argument(..., help="Pod name"),
    namespace: str = typer.Option("default", "--namespace", "-n", help="Kubernetes namespace")
):
    """
    Explain what a pod's logs mean in plain English.

    Examples:
        kai logs nginx-abc123
        kai logs nginx-abc123 --namespace staging
    """
    check_connection()
    console.print(f"\n[bold]Explaining logs for pod:[/bold] [cyan]{name}[/cyan]")
    print_status("Collecting logs...")
    log_output = k8s.get_pod_logs(name, namespace)

    with console.status("[bold green]AI reading logs...[/bold green]"):
        result = ai.explain_logs(name, log_output)
    print_diagnosis(f"Log Explanation: {name}", result)


@app.command()
def explain(
    resource_type: str = typer.Argument(..., help="Resource type (describe)"),
    target_type: str = typer.Argument(..., help="Kubernetes resource type (pod, deployment, node)"),
    name: str = typer.Argument(..., help="Resource name"),
    namespace: str = typer.Option("default", "--namespace", "-n", help="Kubernetes namespace")
):
    """
    Explain kubectl describe output in plain English.

    Examples:
        kai explain describe pod nginx-abc123
        kai explain describe deployment nginx-deploy
        kai explain describe node worker-1
    """
    check_connection()

    if resource_type.lower() != "describe":
        console.print(f"[red]Unknown subcommand:[/red] {resource_type}")
        console.print("Currently supported: describe")
        console.print("Example: kai explain describe pod nginx")
        raise typer.Exit(1)

    console.print(f"\n[bold]Explaining:[/bold] kubectl describe {target_type} [cyan]{name}[/cyan]")
    print_status(f"Running kubectl describe {target_type} {name}...")

    describe_output = k8s.run_command(["kubectl", "describe", target_type, name, "-n", namespace])

    with console.status("[bold green]AI explaining output...[/bold green]"):
        result = ai.explain_describe(target_type, name, describe_output)

    print_diagnosis(f"Describe Explanation: {target_type}/{name}", result)


@app.command()
def analyze(
    target: str = typer.Argument(..., help="What to analyze (cluster)")
):
    """
    Analyze the entire cluster and report health status.

    Example:
        kai analyze cluster
    """
    check_connection()

    if target.lower() == "cluster":
        console.print(f"\n[bold]Analyzing cluster health...[/bold]")
        print_status("Collecting node status...")
        nodes = k8s.get_cluster_nodes()
        print_status("Collecting all pod status...")
        pods = k8s.get_all_pods()
        print_status("Collecting warning events...")
        events = k8s.get_cluster_events()

        with console.status("[bold green]AI analyzing cluster...[/bold green]"):
            result = ai.analyze_cluster(nodes, pods, events)
        print_diagnosis("Cluster Health Analysis", result)

    else:
        console.print(f"[red]Unknown target:[/red] {target}")
        console.print("Currently supported: cluster")
        raise typer.Exit(1)


@app.command()
def doctor():
    """
    Check that your environment is set up correctly.

    Verifies:
        - Ollama is running
        - Mistral model is installed
        - kubectl is reachable
        - Cluster is accessible

    Example:
        kai doctor
    """
    console.print("\n[bold]Running environment checks...[/bold]\n")

    # Use Rich Table for aligned output
    table = Table(show_header=False, box=None, padding=(0, 2))
    table.add_column(style="bold")
    table.add_column()

    all_good = True

    # Check 1 — Ollama
    if ai.check_ollama():
        table.add_row("[green]✔[/green]", "Ollama is running")
    else:
        table.add_row("[red]✗[/red]", "Ollama is NOT running — run: sudo systemctl start ollama")
        all_good = False

    # Check 2 — Mistral model
    if ai.check_model():
        table.add_row("[green]✔[/green]", "Mistral model is installed")
    else:
        table.add_row("[red]✗[/red]", "Mistral not found — run: ollama pull mistral")
        all_good = False

    # Check 3 — kubectl reachable
    import subprocess
    kubectl_check = subprocess.run(
        ["kubectl", "version", "--client", "--short"],
        capture_output=True, text=True, timeout=5
    )
    if kubectl_check.returncode == 0:
        table.add_row("[green]✔[/green]", "kubectl is installed")
    else:
        table.add_row("[red]✗[/red]", "kubectl not found — install from k8s.io/docs")
        all_good = False

    # Check 4 — Cluster accessible
    if k8s.is_cluster_reachable():
        table.add_row("[green]✔[/green]", "Cluster is accessible")
    else:
        table.add_row("[yellow]⚠[/yellow]", "Cluster not reachable — check kubeconfig or start kind cluster")

    console.print(table)
    console.print()

    if all_good:
        console.print("[bold green]All checks passed. kai is ready.[/bold green]")
    else:
        console.print("[bold red]Some checks failed. Fix the issues above before running kai.[/bold red]")


@app.command()
def version():
    """Show kai version."""
    console.print("kai version 1.0.0")
    console.print("kubectl AI — local Kubernetes debugging assistant")
    console.print("Model: Mistral 7B via Ollama")


if __name__ == "__main__":
    app()
```

---

#### File 4 — kai (Entry Point)

```python
#!/usr/bin/env python3
# kai — entry point for the 'kai' command
#
# The shebang (#!/usr/bin/env python3) tells the OS to run this with
# Python 3 when executed directly. Without it, the OS doesn't know
# what interpreter to use.

from cli import app

if __name__ == "__main__":
    app()
```

---

#### File 5 — kubectl-ai (kubectl Plugin Entry Point)

```python
#!/usr/bin/env python3
# kubectl-ai — entry point for the 'kubectl ai' command
#
# kubectl's plugin system works by scanning PATH for executables
# named kubectl-<something>. When you run:
#   kubectl ai diagnose pod nginx
# kubectl finds 'kubectl-ai' in PATH and runs it, passing
# 'diagnose pod nginx' as arguments. That's the entire system —
# just a file in PATH with the right name.
#
# This file is identical to 'kai'. Same codebase, two entry points.

from cli import app

if __name__ == "__main__":
    app()
```

Make both executable:

```bash
chmod +x kai kubectl-ai

# Temporary (current session):
export PATH="$PATH:$(pwd)"

# Permanent:
echo 'export PATH="$PATH:/path/to/your/kai/folder"' >> ~/.bashrc
source ~/.bashrc

# Or symlink both:
sudo ln -s $(pwd)/kai /usr/local/bin/kai
sudo ln -s $(pwd)/kubectl-ai /usr/local/bin/kubectl-ai
```

---

#### File 6 — requirements.txt

```
typer==0.9.0
rich==13.7.0
requests==2.31.0
```

---

#### File 7 — .gitignore

```gitignore
# Python
venv/
__pycache__/
*.pyc
*.pyo
.env

# Logs
*.log

# OS
.DS_Store
```

Without this, you'd accidentally push your entire virtual environment to GitHub — hundreds of megabytes of packages anyone can reinstall with one command.

---

#### File 8 — LICENSE

```
MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Without a LICENSE file, people technically cannot legally reuse your code, even if it's public on GitHub. MIT is the most permissive — it lets anyone use, modify, and distribute freely.

---

### Step 4 — Verify Setup First

Before testing commands, run:

```bash
kai doctor
```

Expected output:

```
Running environment checks...

  ✔  Ollama is running
  ✔  Mistral model is installed
  ✔  kubectl is installed
  ✔  Cluster is accessible

All checks passed. kai is ready.
```

If anything shows red, fix it before continuing. This is the point of `kai doctor` — you know exactly what's wrong before you waste time debugging it in a real command.

---

### Step 5 — Test It

**Test 1 — ImagePullBackOff**

```bash
kubectl get pods
kai diagnose pod nginx-broken-image-<hash>
```

**Test 2 — CrashLoop**

```bash
kai diagnose pod nginx-crashloop-<hash>
```

**Test 3 — Logs**

```bash
kai logs nginx-crashloop-<hash>
```

**Test 4 — Explain describe (the new one)**

```bash
kai explain describe pod nginx-broken-image-<hash>
```

This is different from diagnose. It doesn't say "here's what's wrong and how to fix it." It says "here's what this output is showing you, field by field." Perfect for learning what kubectl describe actually contains.

**Test 5 — The money shot**

Run these two back to back and screenshot both:

```bash
kubectl get pods
kai analyze cluster
```

Three broken pods, then the AI seeing all of them and giving you a prioritized fix list. That pair sells the whole idea.

**Test 6 — kubectl plugin syntax**

```bash
kubectl ai diagnose pod nginx-broken-image-<hash>
kubectl ai version
```

Same output. Different entry point. This is what makes it feel like a real DevOps tool.

---

### What You Actually Built

Five commands. Three files. Two entry points.

```bash
kai doctor                              # environment health check
kai diagnose pod <n>                    # AI pod diagnosis
kai diagnose deployment <n>             # AI deployment diagnosis
kai logs <n>                            # AI log explanation
kai explain describe pod <n>            # AI explain kubectl output
kai analyze cluster                     # AI cluster health report
kai version                             # tool version

kubectl ai diagnose pod <n>             # all of the above via kubectl plugin
```

The Python is intentionally simple. Functions, no classes. `subprocess` with a timeout for shell commands. `requests` for HTTP. `typer` for CLI. These four things together can build most DevOps automation tools you'll ever need.

---

### Future Work

**Watch mode**
```bash
kai watch pod nginx
```
Poll every 30 seconds. Alert when pod status changes. One `while True` loop with `time.sleep(30)`.

**Multi-model support**
```bash
kai diagnose pod nginx --model llama3
```
One extra `typer.Option`, passed through to `ask_ollama()`.

**JSON output**
```bash
kai diagnose pod nginx --output json
```
Wrap the result in `json.dumps()`. Useful for piping into other tools.

**Packaging to TestPyPI**
Add `pyproject.toml`, define entry points, and `pip install kai` works anywhere. This is the path from a learning project to something you'd put in a production runbook.

**The bigger picture**
What you built is the first step toward a broader local DevOps AI assistant. Same architecture, different collectors:

```bash
kai diagnose docker nginx
kai diagnose linux
kai explain logs server.log
```

Same three-file pattern. Different k8s.py equivalent per platform. That's the extensibility the architecture already supports.

---

### Final Thoughts

I built this to learn, not to impress anyone. But by the end of it I genuinely understood:

- Where Kubernetes failure information lives and how to extract it
- How local LLMs work and why prompt structure matters more than model size for specific tasks
- How CLI tools are architected — one file per concern, functions as the unit of logic
- How kubectl's plugin system works and why it's so simple
- Why design decisions like "subprocess not kubernetes-client" matter and how to explain them

The code is evidence of the understanding. That's what's actually yours after building this.

---

---

# PART B — FILE CHECKLIST

Create these files in order. Eight files total.

```
k8s.py           ← File 1
ai.py            ← File 2
cli.py           ← File 3
kai              ← File 4  →  chmod +x kai
kubectl-ai       ← File 5  →  chmod +x kubectl-ai
requirements.txt ← File 6
.gitignore       ← File 7
LICENSE          ← File 8
```

Then:

```bash
pip install -r requirements.txt
export PATH="$PATH:$(pwd)"

kai doctor          # verify everything is set up
kai version         # confirm it runs
kubectl ai version  # confirm kubectl plugin works
```

---

### README.md

````markdown
# kai — kubectl AI

A local AI assistant that debugs Kubernetes for you.
No API keys. No cloud. Runs entirely on your machine.

## Quick Demo

```bash
$ kubectl get pods
NAME                       READY   STATUS             RESTARTS
nginx-broken-image-abc     0/1     ImagePullBackOff   0
nginx-crashloop-def        0/1     CrashLoopBackOff   4

$ kai diagnose pod nginx-broken-image-abc
→ Collecting describe output...
→ Collecting logs...
→ Collecting events...
⠋ AI analyzing pod...

Problem:
  ImagePullBackOff — image nginx:this-tag-does-not-exist not found

Why it happens:
  The image tag doesn't exist in Docker Hub. Kubernetes received
  a 404 and is backing off before retrying.

Fix command:
  kubectl set image deployment/nginx-broken-image nginx=nginx:latest
```

## Commands

```bash
kai doctor                         # check environment setup
kai diagnose pod <n>               # diagnose a failing pod
kai diagnose deployment <n>        # diagnose a failing deployment
kai logs <n>                       # explain pod logs
kai explain describe pod <n>       # explain kubectl describe output
kai analyze cluster                # full cluster health report
kai version

# All commands also work as kubectl plugin:
kubectl ai diagnose pod <n>
kubectl ai analyze cluster
```

## Requirements

- Python 3.9+
- Ollama: `sudo systemctl start ollama`
- Mistral model: `ollama pull mistral`
- kubectl connected to a cluster
- Docker + Kind for local testing

## Setup

```bash
git clone https://github.com/yourusername/kubectl-ai
cd kubectl-ai
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
chmod +x kai kubectl-ai
sudo ln -s $(pwd)/kai /usr/local/bin/kai
sudo ln -s $(pwd)/kubectl-ai /usr/local/bin/kubectl-ai
kai doctor
```

## First run note

First AI response takes 30–40 seconds (model loading into RAM).
Subsequent responses: 10–20 seconds.

## Stack

Python · Typer · Rich · Ollama · Mistral 7B · Kind · kubectl

## License

MIT
````

---

---

# PART C — LINKEDIN POST

---

**Option 1 — Story version**

---

A pod was crashing. I ran `kubectl describe`. Wall of text. Then `kubectl logs`. More text. Then I opened ChatGPT and pasted everything in.

I've done that loop more times than I want to count.

So I built a tool that does it for me.

```bash
kai diagnose pod nginx
```

Collects the logs, events, and describe output automatically. Sends them to a local LLM. Returns a plain English diagnosis with the exact fix command.

My favourite command is the new one:

```bash
kai explain describe pod nginx
```

It takes raw `kubectl describe` output and explains every field in plain English. For anyone learning Kubernetes, that alone is worth the whole project.

Runs 100% locally. No API keys. No cloud. $0.

What I learned building it:

→ AI isn't magic here. It's pattern matching on data you already had. Structure the prompt right and a 7B model diagnoses Kubernetes errors reliably.

→ The hard part wasn't the AI — it was deciding what data to collect, how to truncate it safely, and how to force a consistent output format.

→ The kubectl plugin system is just a file in PATH named `kubectl-*`. That's it.

Stack: Python · Typer · Rich · Ollama · Mistral 7B · Kind · kubectl

Full build guide → [link in comments]

---

**Option 2 — Short punchy version**

---

Built a local AI Kubernetes debugger.

```bash
kai diagnose pod nginx          → diagnoses failing pods
kai analyze cluster             → full cluster health report
kai logs nginx                  → explains logs in plain English
kai explain describe pod nginx  → explains kubectl output field by field
kai doctor                      → checks your whole environment
kubectl ai diagnose pod nginx   → works as a kubectl plugin too
```

100% local. Mistral 7B via Ollama. No API key needed.

Stack: Python · Typer · Ollama · Kind

Blog walkthrough → [link]

---

**Screenshot checklist:**

1. `=== All good ===` from the environment check
2. `kubectl get pods` — three broken pods
3. `kai doctor` — all four green checkmarks
4. `kai diagnose pod nginx-broken-image-xxx` — cyan panel with Problem/Why/Fix
5. `kai explain describe pod nginx-broken-image-xxx` — the explain output
6. `kai analyze cluster` — AI seeing all three broken pods with prioritized fixes
7. `kubectl ai diagnose pod nginx-broken-image-xxx` — same output, kubectl plugin syntax

---

*End of document — v4 (final)*
