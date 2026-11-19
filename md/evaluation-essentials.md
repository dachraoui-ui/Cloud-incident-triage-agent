# Pillar 5: Evaluation & Scientific Benchmarking (How You Grade Your AI)

Welcome to **Pillar 5**! 

This is the **most important part of your graduation project (PFA / thesis)**. 

Anyone can write a Python script that calls an LLM. What makes this an **engineering thesis** rather than a casual weekend tutorial is **scientific proof**:
> *"Did we actually build something useful? How good is it compared to simple alternatives? Where does it fail?"*

---

## Part 1: The School Exam Analogy 🎓

Think of your AI Agent like a student preparing for a final exam:

1. **The Question Bank (40 Faults):** You create 40 different ways to break the microservices app.
2. **Practice Homework (Dev Set - 20 Faults):** The student practices on these 20 problems. You are allowed to adjust prompts and fix tools based on these.
3. **The Final Exam (Test Set - 20 Held-Out Faults):** The student is tested on 20 new problems they **have never seen before**. You are forbidden from modifying the agent during this exam.
4. **Grading (The Scorer):** You compare the agent's diagnosis against the official answer key.

---

## Part 2: What Are the 40 Faults? (The Fault Manifest)

You define a file called `faults/manifest.yaml`. It lists all 40 ways you break the cluster across 8 categories:

```text
                               8 FAULT CATEGORIES
                               
 1. Bad Config        ---> Database pool size changed from 20 to 2
 2. Memory Leak (OOM) ---> Payment pod memory limit set to 32MB (OOMKilled)
 3. CPU Saturation    ---> Heavy math loop injected; CPU throttled
 4. Dependency Down   ---> Payment service scaled down to 0 replicas
 5. Dependency Slow   ---> Network latency added (e.g. 2000ms delay to database)
 6. Crash Loop        ---> Invalid startup flag passed to container
 7. Bad Deploy        ---> Pushed an image tag that doesn't exist
 8. Network Partition ---> Firewall rule blocking frontend from talking to cart
```

### The Dev/Test Split:
- **20 Dev Faults:** Used while building the agent in Weeks 2 to 4.
- **20 Test Faults:** Kept in a locked box. Only run in Week 5 to collect final numbers for the report.

---

## Part 3: What Do You Compare Your Agent Against? (The 4 Baselines)

In science, you cannot say *"Our agent scored 70%, so it is good!"* 
Why? Because maybe a simple 10-line Python script could have scored 80%!

To prove your agent's value, you compare **5 different systems** on the exact same faults:

```text
+---------------------------------------------------------------------------------+
|                               SYSTEMS COMPARED                                  |
|                                                                                 |
| 1. Rule-Based Script  ---> Dumb script: "If payment error spiked, blame payment"|
| 2. LLM, No Tools      ---> Just gives the alert text to ChatGPT without tools   |
| 3. Local Model Agent  ---> Your agent loop running a small local model (Ollama) |
| 4. Hosted Model Agent ---> Your agent loop running a smart API (GPT-4o-mini)    |
| 5. Human Operators    ---> 3 classmates trying to diagnose via Grafana in 10 min|
+---------------------------------------------------------------------------------+
```

---

## Part 4: The 4 Metrics You Measure (Your Final Report Table)

When your evaluation runs finish, this exact table goes on Page 1 of your results chapter:

| Metric | What It Means | Why It Matters |
|---|---|---|
| **Top-1 Accuracy** | Was the true cause ranked **#1** in the report? | Did it find the exact problem immediately? |
| **Top-3 Accuracy** | Was the true cause in the **top 3** suggestions? | Did it give the engineer a useful shortlist? |
| **Time to Diagnosis** | How many seconds did the investigation take? | Is the agent faster than a human? (e.g. 25s vs 7 minutes) |
| **Cost per Run** | How many tokens / cents did it cost? | Is it affordable to run in production? (e.g. $0.01 per alert) |
| **Evidence Validity** | Did the cited tool call actually prove the claim? | Proof that the agent didn't hallucinate! |

---

## Part 5: How Does the Automated Scorer Work? (`eval/scorer.py`)

You don't grade 40 tests by hand. You write a small Python script (`scorer.py`):

```text
[Fault Injected]       ---> "F-014: Payment DB pool lowered to 2"
                                      │
[Agent Output]         ---> "Root cause: Payment connection pool too small"
                                      │
                                      v
                             [eval/scorer.py]
Does agent output match ground truth?
- Correct service? YES (Payment)
- Correct root cause? YES (Connection pool size)
- Evidence cited valid? YES (Checked Loki log)
                                      │
                                      v
                                [SCORE: PASS]
```

---

## Part 6: Academic Honesty Rules (How to Impress Your Professors)

Professors and defense juries love students who understand real scientific methodology:

1. **Never hide bad results:** 
   If your agent struggles on network faults, **say so proudly!** A section titled *"Where our agent fails and why"* gets higher marks than pretending everything is 100% perfect.
2. **Never report percentages alone:** 
   Don't say *"75% accuracy"*. Say: **"15 out of 20 held-out test faults diagnosed correctly."** (75% on 20 samples can be pure luck; 15/20 is honest science).
3. **Run 3 repeats:**
   LLMs have randomness. Run each test fault 3 times and average the results.

---

## Summary of the Entire 5-Pillar Journey

```text
[Pillar 1: Docker]        --> Packages your microservices into boxes.
[Pillar 2: Kubernetes]    --> Runs, manages, and restarts your boxes.
[Pillar 3: Observability] --> Watches the boxes (Prometheus = numbers, Loki = logs).
[Pillar 4: AI Agent]      --> Investigates alerts by calling tools on Prometheus & Loki.
[Pillar 5: Evaluation]    --> Tests the agent on 40 faults and measures how smart it is.
```

That is the entire project from start to finish! Everything is now documented in your workspace.
