# Multi-Tenant Agentic RAG System

A lightweight, enterprise-ready Multi-Tenant Retrieval-Augmented Generation (RAG) system built with **FastAPI**, **CrewAI**, **ChromaDB**, and **Google Gemini**.

Designed for financial invoice ingestion and querying, this architecture enforces **strict multi-tenant metadata isolation** at the vector database layer to prevent cross-vendor data leakage while leveraging AI agents for grounded response synthesis.

---

## 🌟 Key Features

- **Strict Multi-Tenant Isolation**: Hard vector store metadata filtering (`tenant_id = vendor_id`) ensures each vendor can only ingest, view, and query their own documents.
- **Zero-LLM Cost Ingestion**: Uses local HuggingFace embeddings (`BAAI/bge-small-en-v1.5`, 384-dim) running on CPU—no external API keys or quotas are consumed during document indexing.
- **Agentic Synthesis with CrewAI**: Single-agent grounded synthesis pipeline powered by Google Gemini (`gemini-2.5-flash`) that answers questions strictly using retrieved tenant context.
- **Async Execution & Resilience**:
  - Non-blocking asynchronous FastAPI execution (`await crew.run_async()`).
  - Exponential backoff retry logic with jitter via `tenacity`.
  - Graceful fallback for API quota limits.
- **Interactive Web Portal**: Clean FastAPI + Jinja2 user interface supporting PDF/TXT file uploads, raw text pasting, per-vendor statistics, and natural language query exploration.
- **Distilled Architecture**: Clean 3-tier codebase (`app.py` $\rightarrow$ `store.py` $\rightarrow$ `query_crew.py`) that reads like executable pseudocode.

---

## 🏗️ Architecture & Data Flow

### 1. Ingestion Workflow
```
[User / Web Portal]
        │
        ▼ (Upload PDF / TXT / Paste Text)
    [app.py]
        │
        ▼ (Extract text from file)
    [store.py: ingest_invoice]
        │
        ├─► Chunk text (character-level with overlap)
        ├─► Attach metadata: {"tenant_id": vendor_id, "invoice_number": inv_num}
        ├─► Embed locally (BAAI/bge-small-en-v1.5, 0 API cost)
        └─► Upsert into ChromaDB collection
```

### 2. Retrieval & Synthesis Workflow
```
[User Query from Vendor Portal]
        │
        ▼
    [app.py: /query]
        │
        ▼
    [query_crew.py: QueryCrew.run_async]
        │
        ├─► [store.py: retrieve_chunks]
        │        │
        │        └─► ChromaDB query strictly filtered by:
        │            {"tenant_id": {"$eq": vendor_id}}
        │
        ├─► [store.py: format_chunks_for_context]
        │
        └─► [CrewAI SynthesisAgent (Gemini 2.5 Flash)]
                 │
                 ▼ (AsyncRetrying with exponential backoff)
           Synthesized Grounded Answer
```

---

## 📁 Repository Layout

```
multi-tenant_AgenticRAG/
├── app.py                      # FastAPI web server, routes, & PDF/TXT handlers
├── query_crew.py               # CrewAI Synthesis Agent, Task, & async retry runner
├── store.py                    # ChromaDB vector store, local embeddings, chunking, & tenant isolation
├── templates/
│   ├── index.html              # Vendor login & isolated workspace dashboard
│   ├── ingest_results.html     # Ingestion status & chunk index feedback
│   └── query_results.html      # Synthesized answer & ChromaDB chunk inspection
├── requirements.txt            # Python dependencies
├── .env.example                # Environment configuration template
└── .env                        # Local environment secrets (API keys)
```

---

## 🚀 Quick Start

### 1. Prerequisites
- **Python 3.10+**
- A **Google Gemini API Key** ([Get one from Google AI Studio](https://aistudio.google.com/))

### 2. Environment Setup

Clone or open the repository directory:
```bash
cd multi-tenant_AgenticRAG
```

Create and activate a virtual environment:

**Windows (PowerShell):**
```powershell
python -m venv venv
.\venv\Scripts\activate
```

**macOS / Linux:**
```bash
python3 -m venv venv
source venv/bin/activate
```

Install dependencies:
```bash
pip install -r requirements.txt
```

### 3. Configure `.env`

Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```

Edit `.env` and provide your Gemini API key:
```env
GEMINI_API_KEY=your_actual_gemini_api_key_here
GEMINI_MODEL=gemini-2.5-flash
CHROMA_PERSIST_DIR=./chroma_db
CHROMA_COLLECTION_NAME=tenant_invoices
EMBED_MODEL_NAME=BAAI/bge-small-en-v1.5
CHUNK_SIZE=512
CHUNK_OVERLAP=100
TOP_K=3
```

---

## 🖥️ Running the Application

Start the FastAPI development server:
```bash
uvicorn app:app --reload --port 8080
```

Open your browser and navigate to:
```
http://localhost:8080
```

---

## 📖 User Walkthrough & Multi-Tenancy Demo

### Step 1: Sign in as Vendor A
1. Navigate to `http://localhost:8080`.
2. Enter `vendor_alpha` in the **Vendor ID** field and click **Sign In as Vendor**.

### Step 2: Ingest Invoices for Vendor A
1. In the **Ingest Invoices** section, either:
   - Upload PDF or TXT invoice files, OR
   - Paste raw invoice text into the textarea:
     ```text
     INVOICE: INV-001
     Vendor: vendor_alpha
     Date: 2026-08-01
     Amount: $12,500.00
     Description: Cloud infrastructure hosting & database services.
     ```
2. Click **Upload & Ingest**. The document is chunked and stored in ChromaDB tagged with `tenant_id: "vendor_alpha"`.

### Step 3: Sign in as Vendor B
1. Click **🚪 Switch / Logout Vendor** in the top navigation bar.
2. Sign in as `vendor_beta`.
3. Ingest an invoice for `vendor_beta`:
   ```text
   INVOICE: INV-201
   Vendor: vendor_beta
   Date: 2026-08-10
   Amount: $4,200.00
   Description: Office stationery and hardware accessories.
   ```

### Step 4: Verify Strict Tenant Isolation
1. While logged in as `vendor_beta`, enter the query:
   > *"What is the total amount for cloud infrastructure hosting?"*
2. **Result**:
   - The query returns **0 chunks** and the agent confirms no matching invoices exist for `vendor_beta`.
   - `vendor_beta` cannot see or retrieve `vendor_alpha`'s cloud hosting invoice.
3. Switch back to `vendor_alpha` and run the same query:
   - The system retrieves `INV-001` with high similarity and answers: **"$12,500.00 for Cloud infrastructure hosting"**.

---

## ⚙️ Configuration Reference

| Environment Variable | Default Value | Description |
| :--- | :--- | :--- |
| `GEMINI_API_KEY` | *(Required)* | Google Gemini API key for CrewAI synthesis agent. |
| `GEMINI_MODEL` | `gemini-2.5-flash` | Gemini model name used by CrewAI agent. |
| `CHROMA_PERSIST_DIR` | `./chroma_db` | Local filesystem path where ChromaDB persists vector collections. |
| `CHROMA_COLLECTION_NAME`| `tenant_invoices` | Name of the ChromaDB vector collection. |
| `EMBED_MODEL_NAME` | `BAAI/bge-small-en-v1.5` | SentenceTransformers embedding model (local CPU). |
| `CHUNK_SIZE` | `512` | Character length of each chunk during document ingestion. |
| `CHUNK_OVERLAP` | `100` | Overlap character length between sequential chunks. |
| `TOP_K` | `3` | Number of most similar chunks retrieved per query. |

---

## 🔒 Security & Multi-Tenant Design

1. **Deterministic Tenant Partitioning**:
   - Every chunk indexed in ChromaDB contains metadata: `{"tenant_id": "<vendor_id>", "invoice_number": "<inv_id>", "chunk_index": <i>}`.
2. **Server-Enforced Where Filtering**:
   - All retrieval operations in `store.py` execute with strict `$eq` metadata filters:
     ```python
     where = {"tenant_id": {"$eq": tenant_id}}
     ```
   - Filtering is enforced on the database query level before any documents reach the LLM or UI.
3. **Grounded Synthesis**:
   - The LLM prompt is injected *only* with chunks retrieved under the authenticated vendor ID. Even if the user attempts prompt injection, the model has zero access to other tenants' documents.

---

## 🛠️ Tech Stack

- **Backend Web Server**: [FastAPI](https://fastapi.tiangolo.com/) & [Uvicorn](https://www.uvicorn.org/)
- **Templates**: [Jinja2](https://jinja.palletsprojects.com/)
- **Agentic Framework**: [CrewAI](https://crewai.com/)
- **Vector Database**: [ChromaDB](https://www.trychroma.com/)
- **Local Embeddings**: [Sentence-Transformers](https://www.sbert.net/) (`BAAI/bge-small-en-v1.5`)
- **LLM**: [Google Gemini 2.5 Flash](https://ai.google.dev/)
- **Retry & Reliability**: [Tenacity](https://tenacity.readthedocs.io/)
- **Document Extraction**: [PyPDF](https://pypdf.readthedocs.io/)
