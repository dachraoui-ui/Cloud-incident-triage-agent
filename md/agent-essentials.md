# Pillar 4: The LLM Triage Agent (The AI Brain)

Welcome to **Pillar 4**! This is where all the pieces come together. 

Up until now, you learned about the **app** (Docker + Kubernetes) and the **sensors** (Prometheus + Loki). 

Now, you are going to build the **AI Doctor** (the Agent) that looks at the sensors, asks questions, and figures out what broke.

---

## Part 1: What is an "Agent" vs a Normal Chatbot?

| Normal ChatGPT / Chatbot | An "Agent" (What You Are Building) |
|---|---|
| Only knows what it was trained on in the past. | Has access to **live tools** (functions) to check your real cluster right now. |
| You paste logs into the prompt manually. | It **decides by itself** what logs or metrics it needs to see. |
| Answers in 1 turn (you ask, it replies). | Works in a **loop**: *Think -> Call Tool -> Read Result -> Think -> Call Next Tool -> Final Report*. |

```text
                               THE AGENT LOOP
                             
                       +-----------------------------+
                       |   1. Alert Comes In         |
                       |  "Payment error rate > 50%" |
                       +--------------+--------------+
                                      |
                                      v
                               +--------------+
                               |   LLM THINKS | <-----------------+
                               +-------+------+                   |
                                       |                          |
                       (Decides: "I need metrics")                |
                                       |                          |
                                       v                          |
+--------------------------------------------------------------+  |
| 2. CALLS TOOL: query_metrics("payment", "error_rate")        |  |
|    - Talks to Prometheus                                     |  |
|    - Returns: "Errors jumped from 0% to 75% at 14:02"        |  |
+------------------------------+-------------------------------+  |
                               |                                  |
                               +----------------------------------+
                               |
                 (After 2-4 tool steps, LLM knows enough)
                               |
                               v
                       +---------------+
                       | 3. DONE!      |
                       | Writes JSON   |
                       | Diagnosis     |
                       +---------------+
```

---

## Part 2: What You Actually Need to Build

The agent is just a **Python program**! You do not need complex frameworks like LangChain or AutoGen. In fact, your guide strongly warns against them because you need to defend every line of code to your professor.

You only need to build **3 things**:

```text
agent/
  ├── tools.py      <-- 1. The 4 Tools (Functions that talk to Prometheus, Loki, K8s)
  ├── schemas.py    <-- 2. The Tool Menus (JSON descriptions the LLM reads)
  └── loop.py       <-- 3. The Agent Loop (The while loop that runs the investigation)
```

---

## Part 3: The 4 Tools You Will Build (`tools.py`)

Tools are just regular Python functions that your LLM is allowed to call. 

> ⚠️ **Golden Rule for Tools:** NEVER return 10,000 lines of data! If you dump raw logs into the LLM, it will crash, get confused, or cost too much money. Always return **short, capped summaries** (e.g. top 10 error lines).

Here are the 4 tools:

| Tool Name | What it does | Where it gets data |
|---|---|---|
| **`query_metrics`** | Gets numbers over time for a service (CPU, RAM, error rate). | HTTP request to Prometheus API (`/api/v1/query_range`) |
| **`search_logs`** | Searches the last 50 error lines for a service. | HTTP request to Loki API (`/loki/api/v1/query_range`) |
| **`get_recent_deploys`** | Checks if someone changed a config or rolled out new code recently. | Kubernetes API (`kubectl get events` or deployment revision) |
| **`search_runbooks`** | Searches team guidebooks (e.g. *"If payment fails, check DB connection"*). | Vector search over 10 short markdown files |

---

## Part 4: The Tool Menu / Schema (`schemas.py`)

How does the LLM know these tools exist? You give it a "menu" written in JSON format.

Here is an example of what the menu for `query_metrics` looks like:

```python
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "query_metrics",
            "description": "Get numbers (CPU, memory, error rate) for a specific microservice over the last X minutes.",
            "parameters": {
                "type": "object",
                "properties": {
                    "service": {
                        "type": "string",
                        "description": "Name of service, e.g. 'paymentservice', 'cartservice'"
                    },
                    "metric": {
                        "type": "string",
                        "enum": ["cpu", "memory", "error_rate", "latency_p95"],
                        "description": "Which metric to inspect"
                    },
                    "minutes_back": {
                        "type": "integer",
                        "description": "How many minutes back in time to look (max 30)"
                    }
                },
                "required": ["service", "metric"]
            }
        }
    }
]
```

When you send this to OpenAI or Gemini, the model reads the description and knows: *"Aha! When an incident happens in payment, I can call `query_metrics` with `service='paymentservice'`!"*

---

## Part 5: The Agent Loop (`loop.py`)

This is the entire core logic of your project (around 40 lines of Python):

```python
import json
from openai import OpenAI  # Works with OpenAI, or local models via Ollama

client = OpenAI()

def investigate(alert_message, max_steps=6):
    # 1. Start the conversation history
    messages = [
        {
            "role": "system", 
            "content": """You are an SRE incident responder. 
            Investigate the alert using the tools provided.
            When you find the root cause, output a JSON report with:
            - root_cause: what broke
            - affected_service: which service
            - confidence: high/medium/low
            - evidence: list of tool findings that prove your diagnosis"""
        },
        {"role": "user", "content": f"ALERT RECEIVED: {alert_message}"}
    ]

    evidence_collected = {}

    # 2. The Loop
    for step in range(max_steps):
        # Ask LLM: "What do you want to do next?"
        response = client.chat.completions.create(
            model="gpt-4o-mini",      # Cheap, fast, smart at tool calling
            messages=messages,
            tools=TOOL_SCHEMAS
        )

        choice = response.choices[0]

        # Case A: The LLM wants to call a tool
        if choice.finish_reason == "tool_calls":
            for tool_call in choice.message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)

                # Actually run the Python function!
                result = run_tool(tool_name, tool_args)
                
                # Save evidence to prove no hallucination
                evidence_collected[tool_call.id] = result

                # Feed the tool result back into the LLM conversation
                messages.append(choice.message) # Remember what tool LLM picked
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })

        # Case B: The LLM has finished its investigation
        else:
            final_report = choice.message.content
            return final_report

    return "Investigation stopped: max step limit reached"
```

---

## Part 6: Hallucination Prevention (Crucial for Professor & Defense!)

One of the biggest questions your professor will ask is:
> *"How do you know the AI didn't just invent fake logs or guess the answer?"*

Your architecture has an answer built-in: **Evidence Citation Validation**.

1. Every time a tool runs, it has a unique ID (e.g. `call_abc123`).
2. In the final report, the LLM **must** cite the tool ID for every piece of evidence:
   ```json
   {
     "root_cause": "Database connection pool exhausted",
     "evidence": [
       {"citation_id": "call_abc123", "fact": "Loki log shows pool size maxed at 2"}
     ]
   }
   ```
3. Your Python script checks: *"Did `call_abc123` actually return that text?"*
   - If **YES** ➡️ Valid diagnosis.
   - If **NO** ➡️ Reject report (Hallucination caught!).

---

## Part 7: Your Step-by-Step Learning Plan for Pillar 4

1. **Step 1 (Day 1 - Morning):** 
   - Write a simple 10-line Python script that sends a prompt to an LLM API (OpenAI or Ollama) and prints the response.
2. **Step 2 (Day 1 - Afternoon):**
   - Give the LLM 1 fake tool (`get_weather(city)`).
   - See how the LLM responds with a `tool_call` instead of normal text.
3. **Step 3 (Day 2):**
   - Connect the real tools: Write a Python function that does `requests.get("http://localhost:9090/api/v1/query")` to fetch real CPU numbers from Prometheus.
4. **Step 4 (Day 3):**
   - Put them together into the `while` loop. Run a fake alert and watch the AI investigate!
