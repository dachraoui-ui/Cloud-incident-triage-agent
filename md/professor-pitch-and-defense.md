# Project Pitch & Academic Defense: Cloud Incident Triage Agent

Use this guide when presenting the project to your professor, supervisor, or committee. It is structured to answer the exact questions professors care about: **industry problem, academic research value, architectural choices, and scientific methodology**.

---

## 1. Executive Summary (The 30-Second Elevator Pitch)

> "In modern cloud-native systems (Kubernetes), microservices generate millions of metrics and log lines every minute. When an outage occurs, on-call engineers spend 15 to 45 minutes manually combing through dashboards and logs just to identify what broke—costing companies thousands of dollars per minute of downtime.
>
> Our project builds and scientifically benchmarks an **autonomous LLM-powered Incident Triage Agent**. When an alert fires, our read-only agent queries Prometheus metrics, searches Loki logs, and inspects Kubernetes events to generate a ranked root-cause diagnosis backed by verified evidence.
>
> Rather than just building a demo, our core contribution is an **empirical evaluation benchmark** (40 injected faults across 8 failure categories) comparing our agent against rule-based scripts, tool-less LLMs, and human engineers on accuracy, diagnosis speed, and cost."

---

## 2. The Real-World Need & Industrial Context

### The Industry Problem: "Alert Fatigue & Mean Time to Detect (MTTD)"
1. **Microservice Complexity:** A single user action (e.g. clicking "Buy Now") can trigger cascading calls across 15+ microservices (frontend, auth, cart, payment, shipping, database).
2. **Cascading Failures Mask the Real Issue:** If the Payment database runs out of connections, Payment slows down, Cart times out, and Frontend shows an HTTP 500 error. The on-call engineer gets alerted on the *Frontend* (the symptom), but the actual root cause is 4 layers deep in the *Database* configuration.
3. **High MTTR (Mean Time to Resolution):** Over 70% of incident resolution time is spent on **triage** (answering *"what broke and where?"*). Fixing the issue (e.g., restarting a pod or rolling back a config) takes only 2 minutes once the cause is known.

### Why Existing Solutions Fall Short:
- **Traditional Monitoring (Dashboards):** Passive. Requires a human expert to be awake at 3:00 AM, interpret complex graphs, and manually correlate logs with metrics.
- **Rule-Based Alerting:** Brittle. Only detects issues you specifically wrote rules for. Cannot handle novel or multi-service failure combinations.
- **Standard Generative AI (ChatGPT):** Hallucinates. Without real-time access to live telemetry, it invents fictional logs or gives generic advice like *"check your network"*.

---

## 3. The Core Research Question (Academic Value)

Every strong engineering thesis must have a clear research question. Here is yours:

> **"How accurately and how quickly can a tool-using LLM agent identify the root cause of injected faults in a microservices system, compared with a rule-based script, a tool-less LLM, and human operators using dashboards?"**

### What makes this an Engineering Thesis (not just a tutorial):
- **Reproducible Benchmark:** 40 controlled faults with ground-truth labels.
- **Strict Dev/Test Split:** 20 dev faults for tuning; 20 held-out test faults never seen during development.
- **Anti-Hallucination Guardrails:** Every evidence claim in the agent's report must map to a cryptographically verifiable tool-call ID. Reports with ungrounded claims are automatically rejected.
- **Cost & Latency Analysis:** Measuring dollars spent on API tokens and seconds taken per diagnosis.

---

## 4. End-to-End System Architecture

```text
+---------------------------------------------------------------------------------+
|                                1. THE ENVIRONMENT                               |
|                                                                                 |
|   +--------------------------+         +--------------------------+             |
|   |  kind (Kubernetes Node)  | <-----> | Microservices Demo App   |             |
|   |  (Single Docker Container|         | (Google Online Boutique) |             |
|   +--------------------------+         +--------------------------+             |
+---------------------------------------------+-----------------------------------+
                                              | (Scrapes & Aggregates)
                                              v
+---------------------------------------------------------------------------------+
|                            2. OBSERVABILITY LAYER                               |
|                                                                                 |
|   +--------------------------+         +--------------------------+             |
|   |   Prometheus (Metrics)   |         |       Loki (Logs)        |             |
|   |  - CPU, RAM, Error Rates |         |  - Error lines, Panics   |             |
|   +--------------------------+         +--------------------------+             |
|                                     |                                           |
|                                     v                                           |
|                   +----------------------------------+                          |
|                   |  Alertmanager (Alert Webhook)    |                          |
|                   +-----------------+----------------+                          |
+-------------------------------------|-------------------------------------------+
                                      | (Triggers Investigation)
                                      v
+---------------------------------------------------------------------------------+
|                             3. THE AI TRIAGE AGENT                              |
|                                                                                 |
|     +---------------------------------------------------------------------+     |
|     |                         Agent Reasoning Loop                        |     |
|     |                 (Multi-step Tool Calling with LLM)                  |     |
|     +-----------------------------------+---------------------------------+     |
|                                         |                                       |
|             +---------------------------+---------------------------+           |
|             |                           |                           |           |
|             v                           v                           v           |
|    [query_metrics]                [search_logs]          [get_recent_deploys]   |
|    (Prometheus API)                (Loki API)                 (K8s API)         |
|                                         |                                       |
|                                         v                                       |
|                  +----------------------------------------------+               |
|                  | Structured JSON Report with Evidence Citing  |               |
|                  +----------------------+-----------------------+               |
+-----------------------------------------|---------------------------------------+
                                          |
                                          v
+---------------------------------------------------------------------------------+
|                          4. AUTOMATED EVALUATION & SCORING                      |
|                                                                                 |
|   +------------------------+      Compares       +--------------------------+   |
|   | Ground Truth (Manifest)| <-----------------> | Agent Diagnosis Report   |   |
|   +------------------------+                     +--------------------------+   |
|                                                                                 |
|   Outputs: Top-1 Accuracy | Top-3 Accuracy | Time to Diagnosis | Cost per Run   |
+---------------------------------------------------------------------------------+
```

---

## 5. Technology Stack & Design Decisions

If the professor asks *"Why did you choose this technology over that one?"*, use these exact answers:

| Component | Technology Used | Why This Choice (Academic Justification) |
|---|---|---|
| **Cluster Engine** | **`kind` (Kubernetes in Docker)** | Runs the entire cluster inside a single Docker container. Reproducible on any laptop with 1 command (`make up`). No cloud bills. |
| **Demo Application** | **Google Online Boutique** | Industry standard microservices benchmark (10+ polyglot services in Go, Python, Java, Node.js). Realistic inter-service calls. |
| **Metrics Collector** | **Prometheus + Alertmanager** | Cloud Native Computing Foundation (CNCF) graduated standard for cloud time-series metrics. |
| **Log Collector** | **Grafana Loki** | Designed specifically to pair with Prometheus labels; indexing metadata instead of full-text saves massive RAM. |
| **Visualization** | **Grafana** | Unified dashboard used for human operator baseline benchmarking. |
| **Agent Framework** | **Custom Python Loop (No LangChain)** | Zero unnecessary abstraction. We write and control the loop directly so we can inspect tokens, control timeouts, and defend every line of code. |
| **LLM Inference** | **Hosted API (GPT-4o-mini / Gemini)** + **Local (Ollama 3B)** | Enables comparative analysis between lightweight open-source edge models vs. centralized enterprise frontier models. |
| **Fault Injection** | **Python / Bash Scripts** | Deterministic and automated. Clean cluster state reset before every single fault run. |

---

## 6. Expected Professor Questions & How to Answer Them

### Q1: "Commercial tools like Dynatrace or Datadog already do AI RCA (Root Cause Analysis). What is your novelty?"
> **Answer:** 
> "Commercial AIOps tools use proprietary, black-box anomaly detection models that require enterprise licensing and do not share their methodology. 
> Our project is an **open, reproducible scientific benchmark**. We are not claiming commercial novelty; our goal is to empirically evaluate how well modern general-purpose LLMs perform tool-driven causal reasoning against microservice failures, where they fail, and whether a small 3B open-source model can rival larger hosted models."

### Q2: "Why is your agent read-only? Why not make it auto-heal the cluster?"
> **Answer:** 
> "In production Site Reliability Engineering (SRE), giving an autonomous AI write permissions to modify production infrastructure poses a severe safety risk (e.g. accidentally deleting healthy databases). 
> Triage (diagnosis) is where 70% of human time is lost. By keeping the agent strictly read-only, we solve the diagnosis bottleneck without risking destructive autonomous actions. Remediation with human approval is logical future work."

### Q3: "How do you ensure the LLM isn't just hallucinating or guessing the answer?"
> **Answer:** 
> "We implemented two explicit scientific controls:
> 1. **Baseline Comparison:** We compare the agent against a 'No-Tools LLM' (given only alert text). If the tool-using agent significantly outperforms it, the tools provided real information.
> 2. **Evidence Citation Validation:** Every claim in the generated JSON diagnosis must cite a specific tool-call identifier. Our automated scorer inspects the tool response logs; if an agent cites a fact that never appeared in the tool outputs, the report is rejected as a hallucination failure."

### Q4: "How do you prevent data leakage / cheating in your evaluation?"
> **Answer:** 
> "We maintain a strict 50/50 Dev/Test split across 40 faults. During development, prompts and tools are refined only against the 20 Dev faults. The 20 Test faults remain completely untouched until final evaluation. Furthermore, runbooks contain general operational procedures and never contain specific fault IDs or injection answers."

---

## 7. Deliverables & Expected Outcomes

By the end of this 6-week project, we deliver:
1. **Fully Automated Infrastructure:** A single `make up` command that bootstraps the cluster, installs microservices, and deploys monitoring.
2. **The Autonomous Triage Agent:** A lightweight Python agent capable of autonomous multi-step troubleshooting.
3. **The 40-Fault Benchmark Suite:** An automated fault injector and scorer evaluating across 8 failure categories.
4. **Empirical Results Chapter:** Comparative performance tables (Top-1/Top-3 accuracy, latency, cost) across 5 systems (Rule-based, No-Tools LLM, Local Model, Hosted Model, Human).
5. **Failure Analysis Study:** A deep-dive chapter analyzing the specific edge cases where the LLM's causal reasoning broke down.
