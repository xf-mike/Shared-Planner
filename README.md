# ✈️ Shared Planner (AI Trip Planner | v2.0 Microservice Refactor)

> **Evolution Note:** This repository has been completely refactored from a file-based monolithic script (v1.0) into a scalable, containerized **Agent Platform**.
> The new architecture adopts a **Unified Container Strategy**—bundling the React frontend and Flask backend into a single deployable unit—while offloading long-term memory to a dedicated **Weaviate** vector database service.

## 📖 Project Overview

**AI Trip Planner** is a multi-user, social-aware AI agent designed to create highly personalized travel itineraries.

Unlike standard chatbots that forget context after a session ends, this system features a persistent **Social Memory Network**. It allows the Agent to "remember" user preferences across sessions and, uniquely, to secure permission to "learn" from the travel experiences of other users in the network.

By integrating **RAG (Retrieval-Augmented Generation)** with a custom **Hybrid Reranking Algorithm**, the Agent delivers precise, context-rich responses that combine the semantic understanding of LLMs with the factual accuracy of retrieved memories.

## ✨ Core Features (The "Why")

### 🏗️ Unified Microservice Deployment
* **Single-Container Efficiency:** We abandoned the complex multi-container setup for the application layer. The modern Vite/React frontend is built and directly injected into the Flask backend container.
* **Result:** You get a full-stack application (API + UI) running in a single lightweight Docker container, with zero CORS issues and simplified deployment.

### 🧠 Hybrid Long-Term Memory
* **Beyond Simple Vector Search:** We implemented a sophisticated retrieval pipeline that doesn't just look for similar vectors.
* **Dual-Strategy Recall:** Combines **Semantic Search** (Weaviate `text2vec`) to understand intent with **Keyword Match** (BM25) to capture specific proper nouns (e.g., "Kyoto," "Sushi").
* **Time-Aware Reranking:** A custom Python scoring layer prioritizes recent memories using a **Time Decay** function, ensuring the agent doesn't act on outdated information.

### 🤝 Social Memory Network
* **Collaborative Intelligence:** Users are no longer isolated islands. Through a graph-based permission system (`exposed_to` / `amplify_from`), users can selectively share their travel logs.
* **Contextual Amplification:** When planning a trip, the Agent can dynamically reference the shared "pro tips" or "hidden gems" from trusted friends in your network.

### 🛡️ Privacy-Preserving Sharing
* **Automatic Sanitization:** Sharing doesn't mean compromising privacy. Before any memory enters the public graph, it passes through an **LLM-based PII Firewall**.
* **Safe-by-Design:** Sensitive details (names, phone numbers, addresses) are automatically redacted, ensuring only useful travel knowledge is shared.

### 🔌 Resilient Context Management
* **Smart Trimming:** To prevent the "infinite loop" errors common in LLM agents, we developed a "Block-wise" context trimming algorithm.
* **Logic:** It intelligently identifies and preserves "Atomic Tool Chains" (the User query + Tool Call + Tool Output), ensuring the LLM never loses track of the current operation's state.

-----

## 🛠️ Deployment & Configuration

This project uses **Docker Compose** as the primary entry point. It orchestrates the application logic (Backend/Frontend) and the vector database (Weaviate) into a unified, isolated network.

### 1\. Prerequisites

  * **Docker & Docker Compose**: Ensure Docker Desktop or the Docker Engine is installed and running.
  * **OpenAI API Key**: Required for the LLM (GPT-4o-mini) and the Embedding Model (`text-embedding-3-small`).

### 2\. Environment Variables

Create a `.env` file in the root directory of the repository.

**Essential Keys**

| Variable | Description | Required? | Example |
| :--- | :--- | :--- | :--- |
| `OPENAI_API_KEY` | Key for LLM reasoning and Weaviate vectorization. | **Yes** | `sk-...` |
| `USE_LTM` | Enable/Disable Long-Term Memory (LTM). | Yes | `True` |
| `USE_VEC_DB` | Enable Weaviate Vector DB (requires `USE_LTM=True`). | Yes | `True` |

**Tool Configurations (Optional)**
*If not provided, the corresponding tools (Google Search, Google Maps) will be automatically disabled in `tools.py`.*

| Variable | Description | Source |
| :--- | :--- | :--- |
| `GOOGLE_API_KEY` | Google Cloud API Key (Search & Maps). | [Google Cloud Console](https://console.cloud.google.com/) |
| `GOOGLE_CSE_ID` | Custom Search Engine ID (for `search_tool`). | [Google Programmable Search](https://programmablesearchengine.google.com/) |

**System Tuning**

| Variable | Default | Description |
| :--- | :--- | :--- |
| `RUN_AS_DEV` | `False` | `True` enables CORS and disables static file serving (for local dev). `False` serves the React app. |
| `VERBOSE` | `True` | Prints detailed RAG retrieval logs and tool outputs to the console. |
| `MAX_TURNS_IN_CONTEXT` | `16` | Maximum number of chat turns to keep in the LLM context window. |

### 3\. Running the Application

We provide two ways to start the system using Docker Compose.

#### Option A: Quick Start (Using Pre-built Image)

If you just want to run the application without modifying the code:

```bash
docker compose up -d
```

#### Option B: Build from Source

If you have modified the code (Backend or Frontend) and need to rebuild:

```bash
# This triggers the multi-stage build (Node build -> Python runtime)
docker compose up --build -d
```

### 4\. Verifying the Deployment

Once the containers are running (`docker compose ps`), you can access the services:

  * **Web Interface:** [http://localhost:8080](https://www.google.com/search?q=http://localhost:8080)
      * *Serves the compiled React Frontend.*
  * **Backend API Health:** [http://localhost:8080/api/healthz](https://www.google.com/search?q=http://localhost:8080/api/healthz)
      * *Returns JSON status: `{"ok": true, "use_ltm": true, ...}`*

### 5\. Troubleshooting

  * **Weaviate Connection Failed:**
      * Ensure the `weaviate` service is healthy.
      * Check logs: `docker compose logs -f weaviate`
  * **OpenAI Errors:**
      * Verify `OPENAI_API_KEY` is set correctly in `.env`.
      * The backend logs (`docker compose logs -f backend`) will show authentication errors.

-----

## 🏗️ System Architecture

The V2 architecture moves away from a monolithic script to a modular, containerized environment orchestrated by **Docker Compose**. The system runs within a private network, ensuring secure communication between the Agent logic and its Memory store.

### 1\. Infrastructure Layout

The system consists of two primary services defined in `docker-compose.yaml`:

| Service | Role | Internal DNS | Exposed Port | Description |
| :--- | :--- | :--- | :--- | :--- |
| **`backend`** | **The Brain & Face** | `backend` | `8080` | Hosts the Flask API, the LangGraph Agent, and serves the compiled React frontend. |
| **`weaviate`** | **The Hippocampus** | `weaviate` | *(Isolated)* | The persistent vector store. It is accessible **only** by the `backend` service via the internal `app-network`. |

**Data Persistence:**

  * **User Data:** User profiles, session logs, and relationship graphs are stored in `./data`, mapped to the backend container.
  * **Vector Data:** Weaviate embeddings and indices are persisted in a named Docker volume (or local path `./weaviate_data`).

### 2\. The Unified Container Strategy

A key architectural decision in V2 is the **Frontend Injection** pattern. Instead of running a separate Nginx container for the frontend, we bundle the UI directly into the application container.

This is achieved via a **Multi-Stage Docker Build**:

```dockerfile
# Stage 1: Frontend Builder (Node.js)
FROM node:20-alpine AS frontend-builder
WORKDIR /frontend
# ... install dependencies ...
RUN npm run build  # Compiles React to static HTML/CSS/JS

# Stage 2: Backend Runtime (Python)
FROM python:3.11-slim AS backend
WORKDIR /backend
# ... install python dependencies ...

# THE INJECTION: Copy artifacts from Stage 1 to Backend
COPY --from=frontend-builder /frontend/dist ./dist

CMD ["python", "app.py"]
```

**How it works at runtime:**

  * **Production Mode (`RUN_AS_DEV=False`):** The Flask app (`app.py`) serves as both the API server (at `/api/*`) and the Static File Server (at `/`).
  * **Development Mode (`RUN_AS_DEV=True`):** The Flask app only serves the API and enables CORS. You run the frontend separately (`npm run dev`) for hot-reloading.

-----

## 🧠 Technical Deep Dive

This section explores the engineering decisions behind the system's three core pillars: **Memory Reranking**, **Social Graph Consistency**, and **Resilient Orchestration**.

### 1. The Memory Engine: Recall & Rerank

The system implements a sophisticated "Recall-Rerank" pipeline (in `vectorDB.py` and `memory.py`) to ensure retrieved memories are not just semantically similar, but also factually precise and temporally relevant.

**The Pipeline:**

1.  **Hybrid Recall (Weaviate):**
    We use Weaviate's **Hybrid Search**, which combines two scores:
    * **Sparse Score (BM25):** Matches exact keywords (e.g., "Sushi in Tokyo").
    * **Dense Score (Vector):** Matches semantic intent via `text2vec-openai`.
2.  **Permission Filtering:**
    The search scope is strictly limited by the Social Graph. A memory is recalled only if:
    * `Owner == Current User` (Private Memory)
    * **OR** `Owner ∈ amplify_from List` **AND** `Shared == True` (Social Memory)
3.  **Custom Reranking (Python):**
    We apply a secondary scoring layer in Python to prioritize "fresh" memories. The final relevance score $$S$$ is calculated as:

    $$S = (\alpha \cdot S_{cos} + (1-\alpha) \cdot S_{keyword}) \times (0.85 + 0.15 \cdot \text{TimeDecay})$$

    * *TimeDecay*: An exponential decay function with a 14-day half-life.
    * *Result*: A memory from yesterday is ranked higher than a similar memory from last year.

### 2. Social Graph & Privacy Firewall

The social aspect is managed by `relation.py` and protected by a "Privacy Firewall" in `vectorDB.py`.

**The Trust Graph:**
* **`exposed_to` (Outgoing):** I allow specific users to index my shared memories.
* **`amplify_from` (Incoming):** I subscribe to specific users' shared memories to augment my planning.
* **Consistency:** The system enforces bidirectional consistency. If User A stops exposing data to User B, User A is automatically removed from User B's `amplify_from` list.

**The Privacy Firewall (PII Sanitization):**
To ensure safety, no raw memory is ever shared directly.
* **Private Write:** Raw text is saved with `shared=False`.
* **Public Write:** The text passes through an LLM check.
    * *If PII detected:* The LLM generates a **Sanitized Version** (e.g., replacing names with `[NAME]`), which is saved as a separate object with `shared=True`.
    * *If clean:* The original text is marked `shared=True`.

### 3. Agent Orchestration & Context Safety

The agent core (`orchestrate.py`) uses **LangGraph** to manage the loop between reasoning and tool execution. A critical innovation here is the **Context Trimming Strategy** (`context.py`).

**The "Broken Tool Chain" Problem:**
Standard context trimmers often blindly cut the history to fit token limits. If a trimmer removes a `Tool Call` but keeps the `Tool Output` (or vice versa), the OpenAI API returns a 400 Validation Error.

**Our "Block-Wise" Solution:**
Our `trim_context` algorithm preserves logical consistency:
1.  **System Prefix:** Always keeps the System Prompt and Memory Injections.
2.  **Atomic Tool Blocks:** It identifies sequences of `AI Message (Tool Call)` + `Tool Message (Result)` and treats them as an indivisible block.
3.  **Last-Mile Guarantee:** It forces the retention of the most recent `Human Message` and all subsequent processing, ensuring the Agent never loses track of the immediate request, even if older history must be aggressively pruned.

---

## ⚡ Performance Engineering

To ensure the Agent remains responsive even under heavy load or high-latency disk operations, we implemented specific optimization patterns in `cache.py` and `memory.py`.

### 1. Asynchronous Write-Behind
Database and file system writes are the bottleneck of most chat applications.
* **Implementation:** We utilize Python's `ThreadPoolExecutor` to handle persistence tasks in the background.
* **Benefit:** When the Agent generates a reply, it returns the response to the user immediately. The operations to append the chat log to disk and vectorize the memory into Weaviate happen asynchronously in separate threads, resulting in zero perceived latency for the user.

### 2. Thread-Safe LRU Caching
Chat applications frequently re-read the conversation history to build the LLM context.
* **Implementation:** A custom `JSONLCache` wraps the session file reader. It uses an **LRU (Least Recently Used)** eviction policy to keep active session data in RAM.
* **Benefit:** Reduces disk read operations by over 90% during active conversations, significantly speeding up the "Context Building" phase of the pipeline.

---

## 📄 License & Disclaimer

**License:**
This project is open-source and available under the **MIT License**.

**Disclaimer:**
This is an experimental AI Agent system.
1.  **Hallucinations:** Like all LLMs, the Agent may generate incorrect information. Always verify travel details (flight times, visa rules) with official sources.
2.  **Privacy:** While the system includes a PII sanitization layer, no automatic filter is 100% perfect. Users are advised not to share highly sensitive personal data (e.g., credit card numbers, passwords) with the Agent.
