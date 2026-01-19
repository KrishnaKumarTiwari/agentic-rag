# Architecture Comparison: Pipeline RAG vs. Agentic RAG

> **Context**: Comparing the "Golden Standard" Agentic Architecture (proposed) vs. the Standard Enterprise Pipeline RAG (reference post).

## 1. Core Difference: Static vs. Dynamic
The fundamental difference lies in the **Control Flow**.

| Feature | Standard Pipeline RAG (The Post) | Agentic RAG (Proposed) |
| :--- | :--- | :--- |
| **Control Flow** | **Linear / DAG**: `Input -> Retrieval -> Rerank -> Generation` | **Cyclic / Loop**: `Input -> Plan -> Tool -> Reflect -> Tool -> Generate` |
| **Reasoning** | **Implicit**: The LLM just "sees" the context and answers. | **Explicit**: The Agent *decides* what it needs to know, plans steps, and iterates. |
| **Data Scope** | **Fixed**: Only retrieves from the configured Vector DB. | **Flexible**: Can decide to query Vector DB, SQL DB, or an API based on the question. |
| **Nature** | **Reader**: It reads found documents and summarizes. | **Actuator**: It can read, analyze, calculating, and even *do* things (send emails, query live APIs). |

## 2. Capabilities: What Agentic Solves that Pipeline Cannot
The Standard Pipeline RAG is excellent for "Knowledge Retrieval" but fails at "Task Execution" and "Complex Reasoning".

### A. Multi-Hop Reasoning (The "Missing Link" Problem)
*   **Query**: *"How does the Q3 sales report compare to the Q4 forecast in terms of growth %?"*
*   **Pipeline RAG**: Fails/Poor. It might retrieve "Q3 Sales" doc and "Q4 Forecast" doc, but if the *math* isn't in the text, it might hallucinate or just list the numbers without calculating the percentage difference correctly. It simply "stuff context".
*   **Agentic RAG**:
    1.  Tool Call 1: `get_sales(period="Q3")` -> Returns $1M.
    2.  Tool Call 2: `get_forecast(period="Q4")` -> Returns $1.2M.
    3.  Reasoning: Calculate `(1.2 - 1.0) / 1.0 = 20%`.
    4.  Answer: "Growth is 20%."

### B. Heterogeneous Data Sources (The "Silo" Problem)
*   **Query**: *"Find the latest policy on 'Remote Work' and tell me if 'John Doe' is eligible based on his location."*
*   **Pipeline RAG**: Fails. It can find the "Remote Work Policy" (unstructured text) but has no access to "John Doe's location" (structured SQL/HRIS data). It will say: *"I found the policy, but I don't know who John Doe is."*
*   **Agentic RAG**:
    1.  Tool Call 1: `knowledge_base.search("Remote Work Policy")` -> "Eligible if > 50 miles from office."
    2.  Tool Call 2: `hr_db.get_employee("John Doe")` -> "{ location: 'New York', office: 'Boston', distance: 200 miles }"
    3.  Reasoning: 200 > 50.
    4.  Answer: "Yes, John is eligible because he lives 200 miles away..."

### C. Action Taking
*   **Query**: *"Summarize this incident report and create a Jira ticket."*
*   **Pipeline RAG**: Cannot do this. It can only summarize.
*   **Agentic RAG**: Summarizes the text, then calls `jira_mcp.create_ticket(...)`.

## 3. Latency Comparison
This is the "Price" of agency.

| Metric | Pipeline RAG | Agentic RAG | Analysis |
| :--- | :--- | :--- | :--- |
| **Steps** | 1 Retrieval + 1 Generation | N Tool Calls + N Generations | Agentic is inherently chatter. |
| **Typical Latency** | **1.5s - 3s** | **5s - 15s+** | Agents need "thinking time". |
| **Determinism** | High (Same input -> Same output) | Lower (Can take different paths) | Agents can be variable. |
| **Cost** | 1x Inference | 3x - 10x Inference | Agents use more tokens for planning/tools. |

## 4. Summary Recommendation
*   **Use Pipeline RAG** when: Low latency is critical, the task is purely "search and summarize", and data exists in a single knowledge base.
*   **Use Agentic RAG** when: You need to answer *complex* questions that require combining data (SQL + Text), performing calculations, or taking actions, and users accept slightly higher latency for "smarter" answers.
