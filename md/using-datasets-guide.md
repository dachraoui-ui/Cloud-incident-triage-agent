# Using Datasets to Boost Agent Accuracy: The Right vs. Wrong Way

Yes, using datasets to improve your agent's diagnosis capability is an **excellent research idea**! In fact, AI for IT Operations (AIOps) relies heavily on public incident datasets.

However, in an academic project, **how** you use datasets determines whether your professor gives you top marks or calls your evaluation invalid ("data leakage").

Here is the complete guide on:
1. **The Crucial Distinction:** Distinguishing internal monitoring bugs from real app incidents.
2. **Where to find real-world cloud incident datasets.**
3. **How to use them correctly (RAG / Few-Shot) to double your agent's accuracy.**
4. **Academic guardrails to avoid cheating.**

---

## 1. The Crucial Distinction: What Are You Actually Diagnosing?

Before using a dataset, be clear on what you are diagnosing:

```text
                                TWO DIFFERENT TYPES OF ERRORS
                                
   Type A: "The App Broke" (YOUR PROJECT GOAL)    Type B: "The Sensor Broke" (Infrastructure Flaw)
   --------------------------------------------    -------------------------------------------------
   - Prometheus is healthy                        - Loki disk is full -> cannot write logs
   - Loki is healthy                              - Prometheus ran out of memory -> scraping stopped
   - BUT the Payment app ran out of DB pool        - Grafana cannot connect to Prometheus datasource
   -> AGENT DIAGNOSES THE MICROSERVICE            -> AGENT DIAGNOSES THE MONITORING STACK
```

- **Your main project goal is Type A:** Using Prometheus and Loki as *sensors* to diagnose application incidents (e.g. `paymentservice`, `cartservice`).
- **However, handling Type B is a great advanced feature:** Teaching the agent: *"If Loki returns HTTP 500 or timeout, check if Loki itself is crashing before concluding the app has no logs!"*

---

## 2. Best Public Datasets for Cloud & Kubernetes Incidents

You don't need to invent logs from thin air. You can use these famous, open-source research datasets:

| Dataset Name | Source | What It Contains | How You Can Use It |
|---|---|---|---|
| **Loghub** | [logpai/loghub](https://github.com/logpai/loghub) | Over 77 million log lines from real systems (Hadoop, Linux, OpenStack, HDFS, Spark, Zookeeper). | Extract real distributed system error patterns and regexes. |
| **TrainTicket Incident Benchmark** | Fudan University / Microservices | Real microservice traces, metrics, and logs with 22 injected fault types. | Perfect reference for microservice failure cascades. |
| **CloudOps / AIOps Datasets** | Microsoft / Alibaba Research | Real production cluster trace & metric anomalies. | Study how CPU/memory spikes correlate with downstream errors. |

---

## 3. The 3 Ways to Use a Dataset to Boost Performance

```text
                                  HOW TO INJECT DATASET KNOWLEDGE
                                  
  Approach 1: Few-Shot Examples              Approach 2: Vector RAG (Best)               Approach 3: Fine-Tuning
  (Simple & Fast)                            (Scalable & Dynamic)                        (NOT Recommended)
  -----------------------------              -----------------------------               -----------------------
  Include 3-4 real case examples             Index hundreds of real incident             Train the model weights.
  directly in the Agent system prompt.       post-mortems into ChromaDB.                 
                                             Agent queries it like a library.            
  + Zero cost                                + Zero cost, 100% offline                   - Needs GPU, slow,
  + Immediate accuracy jump                  + Doesn't bloat prompt                      - Prone to hallucinating
  - Uses up prompt tokens                    + Defensible in oral defense                  unverified facts
```

### Approach 1: In-Context "Few-Shot" Examples (Simplest)
Give the agent 3 concrete examples in its system prompt showing what a real root cause investigation looks like:

```text
Example 1:
- Symptom: Frontend HTTP 504 Gateway Timeout.
- Metric: cartservice p95 latency jumped from 20ms to 4000ms.
- Log clue: "redis: connection pool exhausted (max=10)".
- Root Cause: Redis connection starvation in cartservice causing cascading frontend timeouts.
```
Small models (like 3B-8B parameters) perform dramatically better when they see 2 or 3 examples of the reasoning pattern you expect.

### Approach 2: Retrieval-Augmented Generation (RAG) over Post-Mortems (Most Impressive)
Turn incident post-mortems and known error catalogues into a searchable vector database:

1. **Collect 30–50 incident post-mortems:**
   - Real examples: *"When Kubernetes reports Exit Code 137, inspect container memory limits vs working set bytes."*
   - *"When gRPC returns 'DeadlineExceeded', check downstream service CPU saturation."*
2. **Embed them with a free CPU embedding model:** (`sentence-transformers/all-MiniLM-L6-v2`).
3. **Store in ChromaDB** (local, runs in Python memory).
4. **Give the agent a 5th tool: `search_knowledge_base(query)`**:
   - When the agent sees a weird gRPC error in Loki, it calls:
     `search_knowledge_base("gRPC DeadlineExceeded upstream")`
   - It gets back the exact troubleshooting strategy from past incidents!

---

## 4. Academic Honesty Rules: Avoiding "Data Leakage"

If you use a dataset, your professor might ask:
> *"Did your agent succeed only because you fed it the exact answers to your own test faults?"*

Here is how you protect yourself and guarantee an honest, publishable result:

```text
                                  PREVENTING DATA LEAKAGE
                                  
   EXTERNAL DATASET (Loghub / Runbooks)           YOUR 40 INJECTED FAULTS
   ------------------------------------           -----------------------
   - Generic patterns                            - Your specific boutique app
   - General post-mortems                        - Split into 20 Dev / 20 Test
   - "How to troubleshoot OOM"                   - "F-014: Payment DB pool = 2"
                 │                                          │
                 ▼                                          ▼
   Allowed in Knowledge Base / RAG               NEVER put F-014 into Knowledge Base!
```

1. **No Specific Ground-Truth in the Knowledge Base:** 
   The dataset should explain *concepts* (e.g., *"How connection pools behave under load"*), but **never** mention your specific test fault IDs (e.g., *"In test F-014, the payment pool was set to 2"*).
2. **Explicit Evaluation Baseline:**
   Measure your agent **WITH RAG** vs. **WITHOUT RAG** (an *Ablation Study*).
   - *Result example:* "Adding our incident dataset via RAG improved Top-1 accuracy from 11/20 to 16/20."
   - Professors love this because it scientifically proves that the dataset was the reason for the improvement!

---

## 5. Summary: Should You Do It?

- **Yes!** Incorporating a curated dataset of error patterns and post-mortems via **Few-Shot prompting** or **RAG (Runbooks)** will significantly improve accuracy on small/free models.
- It provides a dedicated section in your thesis: *"Impact of Domain Knowledge Retrieval on LLM Diagnostic Accuracy"*.
- It requires zero cloud budget—everything runs in Python with local embeddings!
