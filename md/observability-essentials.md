# Observability Explained from Absolute Zero (Prometheus, Loki, Grafana)

Welcome to **Pillar 3**!

If Kubernetes is the **babysitter** that keeps your app running, **Observability** is the **doctor's monitoring equipment** (heart rate monitor, blood test, stethoscope).

Without observability, when your app breaks, you are completely blind. You only know *"it doesn't work"*, but you have zero idea *why*.

---

## Part 1: The Doctor Analogy 🩺

When you feel sick and visit a hospital, the doctor does 3 things:

1. **Checks your numbers (Vitals):** Takes your pulse, body temperature, and blood pressure.
   👉 In cloud computing, these numbers are called **Metrics** (handled by **Prometheus**).

2. **Listens to your story (Symptoms):** *"My stomach hurts since 2:00 PM after I ate seafood."*
   👉 In cloud computing, these written messages are called **Logs** (handled by **Loki**).

3. **Looks at the screen / chart:** The doctor glances at the computer monitor displaying the heart rate graph and test results in one nice dashboard.
   👉 In cloud computing, this visual screen is **Grafana**.

---

## Part 2: What Are the 3 Tools & What Do They Do?

```text
+---------------------------------------------------------------------------------+
|                               YOUR K8s CLUSTER                                  |
|                                                                                 |
|   +--------------------------+         +--------------------------+             |
|   |   Payment Service Pod    |         |     Cart Service Pod     |             |
|   |  - CPU: 95%              |         |  - CPU: 10%              |             |
|   |  - Memory: 210MB         |         |  - Memory: 50MB          |             |
|   |  - Prints: "DB timeout!" |         |  - Prints: "Item added"  |             |
|   +------------+-------------+         +------------+-------------+             |
+----------------|------------------------------------|---------------------------+
                 |                                    |
       +---------+------------------------------------+---------+
       |                                                        |
       v (Collects Numbers)                                     v (Collects Text)
+-------------------------------+                       +-------------------------------+
|          PROMETHEUS           |                       |             LOKI              |
|        (The Accountant)       |                       |        (The Librarian)        |
|                               |                       |                               |
| Stores NUMBERS over time:     |                       | Stores TEXT messages (logs):  |
| - 14:00 -> CPU was 12%        |                       | - 14:02: "Payment started"    |
| - 14:05 -> CPU was 95%        |                       | - 14:05: "ERROR: DB timeout!" |
+---------------+---------------+                       +---------------+---------------+
                |                                                       |
                +---------------------------+---------------------------+
                                            |
                                            v (Draws the picture)
                            +-------------------------------+
                            |            GRAFANA            |
                            |         (The TV Screen)       |
                            |                               |
                            |  Plots graphs, shows red      |
                            |  alert badges, displays logs  |
                            +-------------------------------+
```

---

## Part 3: Deep Dive into Each Tool (Simple Words)

### 1. Prometheus (The Accountant of Numbers) 📈

- **What does it care about?** Strictly **NUMBERS** that change over time.
  - Examples:
    - How much memory is Payment using right now? (e.g., `180MB`)
    - How many errors per second are happening? (e.g., `4.2 errors/sec`)
    - How long does a page take to load? (e.g., `850ms`)

- **How does it work?**
  - Prometheus doesn't wait for your app to send data.
  - Every 15 seconds, Prometheus knocks on your app's door: *"Hey, give me your current numbers!"*
  - This action of knocking on the door and collecting numbers is called **scraping**.

- **How do you ask Prometheus questions? (PromQL)**
  - PromQL is just the query language to ask math questions about your numbers:
  - *Example question:* *"Show me the error rate of the payment service in the last 5 minutes."*
  - *In PromQL:*
    ```promql
    rate(http_requests_total{service="payment", status=~"5.."}[5m])
    ```
    *(Don't be scared by the syntax! It just means: count requests where status is 500-error, over a 5-minute window).*

---

### 2. Loki (The Librarian of Text) 📝

- **What does it care about?** Strictly **TEXT LINES** (called **Logs**).
  - When your Python code runs `print("User clicked checkout")` or raises an exception:
    ```text
    ERROR 14:05:22 - DatabaseConnectionError: connection pool exhausted (max=2)
    ```
  - That text gets written to the console.

- **How does it work?**
  - A small background collector grabs all these text lines from every container in the cluster.
  - It ships them to **Loki**.
  - Loki organizes them by service name (e.g. `service="payment"`, `service="frontend"`).

- **How do you search logs in Loki? (LogQL)**
  - *Example question:* *"Show me all log lines from the payment service containing the word 'error'."*
  - *In LogQL:*
    ```logql
    {app="payment"} |= "error"
    ```
  - That is literally it! It's like pressing `Ctrl + F` inside all your servers at the same time.

---

### 3. Grafana (The TV Screen / Dashboard) 🖥️

- **What does it care about?** Making things **easy to see for humans**.
- Grafana has **no data of its own**. It is just a display screen.
- You open Grafana in your web browser (`http://localhost:3000`).
- Grafana connects to **Prometheus** (to draw lines and bar graphs) and to **Loki** (to show text boxes with logs).

---

## Part 4: Alertmanager (The Smoke Alarm) 🚨

Inside the Prometheus family, there is a small component called **Alertmanager**.

1. You write a rule in Prometheus:
   > *"If error rate of payment service is greater than 10% for 2 minutes, sound the alarm!"*
2. When that happens, Alertmanager fires an **Alert**.
3. **In this project:** 
   Alertmanager sends this alert as a message (a webhook) to your **AI Agent**!
   > *"Hey AI Agent, wake up! Payment service is failing!"*

---

## Part 5: How Does Your AI Agent Use This? 🤖

Remember, your AI Agent cannot look at a screen like a human. It uses **code functions (Tools)** to ask Prometheus and Loki questions:

```text
[Step 1: Alert arrives at Agent]
"Payment service error rate > 50%"

[Step 2: Agent asks Prometheus]
Tool Call: query_metrics(service="payment", metric="error_rate")
Prometheus Answer: "Error rate jumped from 0% to 75% at 14:02"

[Step 3: Agent asks Loki]
Tool Call: search_logs(service="payment", filter="error", time="14:02")
Loki Answer: "Connection refused: cannot reach database on port 5432"

[Step 4: Agent writes the diagnosis]
"Root Cause: Payment service cannot connect to its database. 
 Evidence: Connection refused error in logs starting at 14:02."
```

---

## Part 6: Quick Vocabulary Summary (Your Cheat Sheet)

| Term | In Plain English |
|---|---|
| **Metrics** | Numbers measured over time (CPU %, RAM MB, Requests/sec). Handled by **Prometheus**. |
| **Logs** | Text messages printed by the app (errors, warnings). Handled by **Loki**. |
| **Scraping** | Prometheus asking a pod every 15s: *"Give me your numbers"*. |
| **PromQL** | The formula you write to calculate metric numbers (e.g. calculate error rate). |
| **LogQL** | The search phrase you write to find specific text in logs (e.g. `{app="payment"} \|= "timeout"`). |
| **Grafana** | The web page where you view charts and read logs visually. |
| **Alertmanager** | The smoke alarm that notifies your AI Agent when numbers go bad. |

---

That is all of Pillar 3! You don't need to be a data engineer. You just need to know:
- **Prometheus** = gives the Agent the **numbers**.
- **Loki** = gives the Agent the **text clues**.
- **Grafana** = lets you verify with your own eyes.
