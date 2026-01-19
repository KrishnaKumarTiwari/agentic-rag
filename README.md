# Enterprise Agentic RAG with MCP

> **A production-grade Agentic Retrieval-Augmented Generation system using the Model Context Protocol (MCP).**

![Architecture Overview](rag_flow_fixed_layout_1768806498365.webp)

## 🚀 Overview
This project implements a next-generation **Agentic RAG** architecture designed for enterprise scale. Unlike traditional linear RAG chains, this system uses an **Agent Orchestrator** ("The Brain") to dynamically plan, select tools, and synthesize answers from disparate corporate data sources (SharePoint, Databases, APIs) via the **Model Context Protocol (MCP)**.

## 📚 Documentation
The engineering and design documentation is split into the following detailed guides:

| Document | Description |
| :--- | :--- |
| 📘 **[High-Level Design (HLD)](./HLD.md)** | Strategic architecture, component breakdown, cost optimization strategies, and system context. |
| ⚙️ **[Low-Level Design (LLD)](./LLD.md)** | Implementation details, API specifications, database schemas, and configuration. |
| ⚖️ **[Architecture Comparison](./ARCHITECTURE_COMPARISON.md)** | A deep dive into why we chose an *Agentic* approach over standard *Pipeline* RAG. |

## ✨ Key Features
*   **Agentic Core**: Dynamic planning and reasoning loop (Loop-based).
*   **MCP Ecosystem**: Standardized connectivity to internal data silos.
*   **Hybrid Search**: Dense Vector + Sparse Keyword (BM25) + Cohere Re-ranking.
*   **Cost Optimized**: Smart routing between Small/Large models and semantic caching.
*   **Enterprise Guardrails**: PII redaction and Hallucination checks.

## 🏗️ Architecture

```mermaid
graph TD
    User[Corporate User] -->|HTTPS/WSS| Frontend[Web/Mobile UI]
    Frontend -->|REST/GraphQL| Gateway[API Gateway / Load Balancer]
    
    subgraph AgenticPlatform [Agentic RAG Platform]
        Gateway --> Orchestrator[Agent Orchestrator Service]
        Orchestrator <-->|Memory| StateStore[Redis/Postgres]
        Orchestrator <-->|Inference| LLM_Gateway[LLM Gateway / Model Service]
        
        subgraph Guardrails [Guardrails Layer]
            InputGuard[Input Guardrail]
            OutputGuard[Output Guardrail]
        end
        
        Orchestrator -.-> InputGuard
        Orchestrator -.-> OutputGuard
    end

    subgraph MCPEcosystem [MCP Ecosystem - Tools and Data]
        Orchestrator -- "MCP Protocol (JSON-RPC)" --> Host[MCP Host/Client]
        
        Host -->|Connects| KMCP[Knowledge MCP Server]
        Host -->|Connects| SQLMCP[SQL/Data MCP Server]
        Host -->|Connects| ActionMCP[API Action MCP Server]
        
        KMCP <-->|Search| VectorDB[(Vector Database)]
        KMCP <-->|Search| Elastic[(Search Engine)]
        
        SQLMCP <-->|Query| DataWarehouse[(Snowflake/BigQuery)]
        ActionMCP <-->|API| InternalApp[Internal ERP/CRM]
    end

    LLM_Gateway --> OpenAI["External LLM (GPT-4o)"]
    LLM_Gateway --> SelfHosted["Internal LLM (vLLM/Llama)"]
```
