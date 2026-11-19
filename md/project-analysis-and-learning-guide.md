# 🔍 Cloud Incident Triage Agent — Full Analysis & Learning Guide

## 1. Project Summary: What Is This?

You're building an **AI-powered incident diagnosis system** for cloud applications. Here's the plain-English version:

> **Imagine:** A microservices app (like an online shop) is running on Kubernetes. Something breaks. Instead of a human manually checking dashboards, logs, and metrics to figure out *what went wrong*, an **LLM agent** does it automatically — reading logs, querying metrics, checking runbooks — and produces a ranked list of "here's what I think went wrong, and here's my evidence."

Then you **scientifically evaluate** how good the agent is by comparing it to:
- A dumb rule-based script
- An LLM without any tools
- A human using dashboards

```text
+-----------------------+
| 🏗️ Demo App (K8s)     |
+-----------+-----------+
            |
            v
+-----------------------+       [Ground Truth]
| 💥 Fault Injection   | ----------------------------+
+-----------+-----------+                             |
            |                                         |
            v                                         v
+-----------------------+                     +---------------+
| 📊 Observability      |                     |   ✅ Scorer   |
| (Prometheus + Loki)   |                     +-------+-------+
+-----------+-----------+                             ^
            |                                         |
            v                                         |
+-----------------------+                             |
| 🚨 Alert Fires        |                             |
+-----------+-----------+                             |
            |                                         |
            v                                         |
+-----------------------+     +-------------------+   |
| 🤖 LLM Agent          | --> | 📋 Ranked Report  | --+
| (Investigates tools)  |     +-------------------+
+-----------------------+
```

---

## 2. The 5 Big Pillars You Need to Learn

Your project sits at the intersection of **5 technical domains**. You don't need to master any of them — you need to know *just enough* to build. Here's the breakdown:

### Pillar 1: Containers & Docker 🐳

> **What it is:** A way to package an app + its dependencies into a portable unit (a "container") that runs the same everywhere.

**Key concepts:**
| Concept | What it means | Analogy |
|---|---|---|
| Image | A snapshot/template of your app | A recipe |
| Container | A running instance of an image | A dish cooked from the recipe |
| Dockerfile | Instructions to build an image | The recipe card |
| Volume | Persistent storage that survives container restarts | A USB drive you plug in |
| Docker Compose | Run multiple containers together | A dinner menu (multiple recipes at once) |

**Learning path:**
1. Install Docker Desktop on Windows
2. Run `docker run hello-world` — understand what just happened
3. Write a simple Dockerfile for a Python app (Flask "Hello World")
4. Use `docker-compose.yml` to run 2 containers that talk to each other
5. **✅ Done when:** You can build and run a containerized app

**Free resources:**
- [Docker Official "Get Started" Tutorial](https://docs.docker.com/get-started/)
- YouTube: "Docker in 100 seconds" by Fireship (for intuition)

---

### Pillar 2: Kubernetes (K8s) ☸️

> **What it is:** An orchestrator that manages your containers at scale — decides where they run, restarts them if they crash, scales them up/down.

**Key concepts (only what you need):**
| Concept | What it means | In your project |
|---|---|---|
| Pod | Smallest unit; 1+ containers running together | Each microservice runs as a pod |
| Deployment | Manages pod replicas, handles rollouts | You deploy the demo shop app via deployments |
| Service | A stable network address for a set of pods | How services find each other |
| Namespace | Logical isolation within a cluster | Separate your app from monitoring |
| ConfigMap | External config injected into pods | You'll **break things** by changing these |
| `kubectl` | The CLI tool to manage K8s | Your daily driver |

**Key `kubectl` commands to learn:**
```bash
kubectl get pods -n <namespace>          # List pods
kubectl describe pod <name>              # Detailed info + events
kubectl logs <pod-name>                  # Read container logs
kubectl get events --sort-by=.metadata.creationTimestamp  # Recent events
kubectl apply -f <file.yaml>             # Deploy/update resources
kubectl delete pod <name>                # Delete a pod (it restarts if in a deployment)
```

**Learning path:**
1. Install **kind** (K8s in Docker) locally
2. Deploy a simple nginx pod, expose it, access it
3. Deploy a multi-service app (the demo shop)
4. Break something on purpose (scale a deployment to 0) and see what happens
5. **✅ Done when:** You can deploy an app, break it, and read the error from `kubectl`

---

### Pillar 3: Observability (Prometheus + Loki + Grafana) 📊

> **What it is:** The system that **watches** your app — collecting numbers (metrics) and text (logs) so you can detect and diagnose problems.

**Three pillars of observability:**
```text
+-----------------------+     +-----------------------+
|  📈 Prometheus        |     |  📝 Loki              |
|  (Numbers / Metrics)  |     |  (Text / Logs)        |
+-----------+-----------+     +-----------+-----------+
            |                             |
            +--------------+--------------+
                           |
                           v
              +-------------------------+
              |  📊 Grafana             |
              |  (Visual Dashboard)     |
              +-------------------------+
```

**Key concepts:**
| Tool | What it does | Query language |
|---|---|---|
| Prometheus | Scrapes metrics from your services every N seconds | PromQL |
| Loki | Aggregates logs from your services | LogQL |
| Grafana | Web UI to visualize metrics and logs | — |
| Alertmanager | Fires alerts when metrics cross thresholds | — |

**Essential PromQL (metrics queries):**
```promql
# Error rate of a service over 5 minutes
rate(http_requests_total{service="payment", status=~"5.."}[5m])

# CPU usage per pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))

# Memory usage
container_memory_usage_bytes{namespace="default"}
```

**Essential LogQL (log queries):**
```logql
# All error logs from the payment service
{app="payment"} |= "error"

# Parse and filter structured logs
{app="payment"} | json | level="error" | line_format "{{.msg}}"
```

---

### Pillar 4: LLM Tool Calling (The AI Agent Part) 🤖

> **What it is:** An LLM that doesn't just generate text — it can **call functions** (tools) to get real data, then reason about it.

**How an agent loop works:**
```text
Alert: "Payment error rate spiked"
   │
   ▼
Agent thinks: "I should check metrics"
   │ (Calls query_metrics)
   ▼
Prometheus: "error_rate jumped from 0.01 to 0.45 at 14:32"
   │
   ▼
Agent thinks: "Let me check logs around that time"
   │ (Calls search_logs)
   ▼
Loki: "ConnectionPool exhausted, max=2"
   │
   ▼
Agent thinks: "Check if config changed"
   │ (Calls get_recent_deploys)
   ▼
K8s: "ConfigMap changed at 14:30, pool_size: 20 -> 2"
   │
   ▼
Agent Report: "Root cause = config change reduced connection pool"
```

**Python skeleton (the core of your agent):**
```python
from openai import OpenAI  # or any provider

# 1. Define tool schemas
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_metrics",
            "description": "Get a metric time series for a service",
            "parameters": {
                "type": "object",
                "properties": {
                    "service": {"type": "string"},
                    "metric": {"type": "string", "enum": ["cpu", "memory", "error_rate", "latency_p95"]},
                    "minutes_back": {"type": "integer", "maximum": 60}
                },
                "required": ["service", "metric"]
            }
        }
    }
]

# 2. The agent loop
def investigate(alert_text, max_steps=8):
    messages = [
        {"role": "system", "content": "You are an SRE agent. Investigate the alert using tools..."},
        {"role": "user", "content": alert_text}
    ]
    
    for step in range(max_steps):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools
        )
        
        choice = response.choices[0]
        
        if choice.finish_reason == "tool_calls":
            for tool_call in choice.message.tool_calls:
                result = execute_tool(tool_call.function.name, tool_call.function.arguments)
                messages.append({"role": "tool", "content": result, "tool_call_id": tool_call.id})
        else:
            return parse_report(choice.message.content)
    
    return "Step budget exhausted"
```

---

### Pillar 5: Evaluation & Scientific Method 🔬

> **What it is:** How you measure whether your agent actually works, and compare it fairly against baselines.

**Key concepts:**
| Concept | What it means |
|---|---|
| Fault injection | Breaking the system deliberately in known ways |
| Ground truth | The actual cause you injected (the "answer key") |
| Dev/test split | 20 faults for tuning, 20 held out for final evaluation |
| Baselines | Simple alternatives to compare against (rule-based, no-tools LLM, human) |
| Top-1 / Top-3 accuracy | Was the correct cause ranked #1? In the top 3? |
| Ablation | Remove one component and measure the drop (proves each part matters) |

---

## 3. Step-by-Step Build Order

### Phase 1: Foundation (Week 0-1) 🏗️
1. Install Docker Desktop + kind
2. Deploy the demo shop app (Google Online Boutique)
3. Deploy Prometheus + Loki + Grafana
4. Verify you can see metrics & logs in Grafana
5. Write `make up` and `make down` scripts

### Phase 2: Contracts & Fixtures (Week 1-2) 📝
1. Define tool schemas (JSON)
2. Define fault manifest (YAML)
3. Record real responses as JSON fixtures (unblocks agent dev)

### Phase 3: Build Tools + Agent Loop (Week 2-3) 🔧
1. Build 4 read-only tools in Python
2. Test on recorded fixtures
3. Build the core agent loop
4. Add runbook retrieval

### Phase 4: Integration & Alert Glue (Week 3) 🔗
1. Connect Alertmanager webhook to the agent
2. Run 1 full end-to-end test (fault -> alert -> agent -> report)

### Phase 5: Evaluation (Week 4-5) 📊
1. Run 20 dev faults, tune prompts
2. Freeze the agent
3. Run 20 held-out test faults (3 times each)
4. Run baselines (rule-based, no-tools, human)

### Phase 6: Report & Demo (Week 6) 📄
1. Results tables and graphs
2. Failure analysis (5 real examples)
3. README + 2-minute demo video
4. Final academic report

---

## 4. Technologies Cheat Sheet

| Component | Technology | Why this one |
|---|---|---|
| Local K8s | kind | Lightweight, runs as 1 Docker container |
| Demo app | Google Online Boutique | Popular, well-documented |
| Metrics | Prometheus | Industry standard |
| Logs | Loki | Pairs seamlessly with Prometheus |
| Dashboards | Grafana | Universal UI for Prometheus & Loki |
| Alerts | Alertmanager | Built into Prometheus |
| Agent code | Python | Best LLM library ecosystem |
| LLM (main) | Hosted API (GPT-4o-mini, Gemini, etc.) | Reliable function calling |
| Embeddings | sentence-transformers (CPU) | Free, offline |
| Vector store | ChromaDB | Simple, in-process |

---

## 5. What to Learn First (Priority Order)

```text
[1. Docker basics]        (1 day)
        │
        ▼
[2. Kubernetes basics]    (1.5 days)
        │
        ▼
[3. Deploy demo shop]     (0.5 day)
        │
        ▼
[4. Prometheus + Grafana] (1 day)
        │
        ▼
[5. LLM tool calling]     (1 day)
        │
        ▼
[6. Build first tool]     (0.5 day)
        │
        ▼
[7. Agent loop]           (1 day)
        │
        ▼
[8. Connect everything]   (1-2 days)
```

---

## 6. Questions to Answer After Each Week

| Week | Questions |
|---|---|
| 0 | What is a container? What is a pod? What is PromQL? What is tool calling? |
| 1 | Can I run the demo app with one command? Can I see its metrics in Grafana? |
| 2 | Can I call `query_metrics("payment", "error_rate")` and get back JSON? |
| 3 | Can I inject a fault and get a report from the agent end-to-end? |
| 4 | What is the agent's accuracy on dev faults? Where does it fail? |
| 5 | What are the final numbers on held-out faults? How does it compare to baselines? |
| 6 | Can I demo the whole thing in 2 minutes? Is the report written? |
