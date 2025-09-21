
# RAG Chatbot Starter (PDFs + Databases)

A minimal, production-friendly starter to build a **LLM chatbot with Retrieval-Augmented Generation (RAG)** that can answer from **PDF documents** and **database rows**. It uses **FastAPI**, **LangChain**, and **Chroma (no FAISS)**. You can choose **OpenAI** or a **local embedding model**.

---

## What you get

- 📄 **PDF ingestion** with page-aware chunking (citations like `[file.pdf:12]`).
- 🗄️ **DB row ingestion** (e.g., Postgres/SQLite) into the same vector store with citations like `[table:pk]`.
- 🔎 **MMR retriever** with sane defaults (k=5, fetch_k=20).
- 🤖 **LLM answerer** that **only** answers from retrieved context and says **"I don't know"** if not found.
- 🚀 **FastAPI endpoints**: `/ingest/pdfs`, `/ingest/db`, `/chat`.
- ❌ **No FAISS** to avoid Windows install issues.
- ⚙️ Optional: **OpenAI** for LLM/embeddings or **Local** embeddings via `sentence-transformers`.

---

## Prereqs (Windows/macOS/Linux)

- **Python 3.11** (recommended). On Windows PowerShell, if venv activation is blocked:
  - Run PowerShell **as Administrator** once:  
    `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`
- (Optional) A Postgres DB you can read from. (Any SQLAlchemy URL works, e.g., SQLite.)

---

## 1) Setup

```bash
# From this folder
python -m venv .venv
# Windows PowerShell
.\.venv\Scripts\Activate
# macOS/Linux
source .venv/bin/activate

pip install -r requirements.txt
(pip install \
  fastapi==0.115.0 \
  starlette==0.38.6 \
  uvicorn[standard]==0.30.6 \
  python-dotenv==1.0.1 \
  pydantic==2.6.4 \
  pydantic-core==2.16.3 \
  pydantic-settings==2.2.1 \
  langchain==0.2.16 \
  langchain-core==0.2.39 \
  langchain-community==0.2.11 \
  langchain-openai==0.1.22 \
  langchain-text-splitters==0.2.4 \
  langsmith==0.1.122 \
  chromadb==0.5.5 \
  SQLAlchemy==2.0.34 \
  pypdf==4.3.1)
The above requirements if you want to use OpenAI other wise remove langchain

cp .env.example .env
```

Open `.env` and set:

- Use **OpenAI** (easiest): set `EMBEDDING_PROVIDER=openai` and add your `OPENAI_API_KEY`.
- Or **Local** embeddings: set `EMBEDDING_PROVIDER=local` (downloads a small model on first run).
- (Optional) `DB_URL` for your database and `DB_TABLES` as a comma-separated list to ingest rows.

> Tip: If you don't have OpenAI API access, start with `EMBEDDING_PROVIDER=local` and `LLM_PROVIDER=openai` (or `ollama` if you run local models).

---

## 2) Put PDFs in place

Drop your files into: `data/pdfs/`

Then ingest them:

```bash
python app/ingest_pdfs.py
```

You can re-run ingestion whenever you add or update PDFs.

---

## 3) (Optional) Ingest DB rows

In `.env`, set for example:

```
DB_URL=sqlite:///./sample.db
DB_TABLES=customers,orders
```

Then run:

```bash
python app/ingest_db.py
```

This will embed text representations of specified tables' rows with metadata (`table`, `pk`).

> ⚠️ Keep it **read-only** in production. Use a replica or a role with SELECT-only permissions.

---

## 4) Run the API

```bash
uvicorn app.main:app --reload
```

- `POST /chat` with JSON `{"question": "your question"}`
- Returns: `answer` and `sources` (PDF pages or DB table/PK).

Example (PowerShell):

```powershell
$body = @{ question = "What does the onboarding PDF say about password policy?" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://127.0.0.1:8000/chat" -Method Post -Body $body -ContentType "application/json"
```

Example (curl):

```bash
curl -s -X POST http://127.0.0.1:8000/chat   -H "Content-Type: application/json"   -d '{"question": "Show me the most common issue in the support handbook"}' | jq
```

---

## 5) How it works (quick architecture)

1. **Ingestion**
   - PDFs: page-wise text → chunked → embeddings → Chroma with `[file:page]` metadata.
   - DB: rows → text serialization (selected columns) → embeddings → Chroma with `[table:pk]` metadata.
2. **Retrieval**
   - MMR retriever (`k=5`, `fetch_k=20`) to reduce duplicates.
3. **Generation**
   - LLM answers **only** from retrieved context; includes citations.
   - If nothing relevant: `"I don't know."`
4. **Extensibility**
   - Swap vector DB (e.g., pgvector, Qdrant).
   - Add cross-encoder re-ranking, filters, auth, or a web UI later.

---

## 6) Common pitfalls & tips

- If `sentence-transformers` is slow to install, use `EMBEDDING_PROVIDER=openai`.
- If on Windows and venv activation fails, use the `Set-ExecutionPolicy` command above.
- Keep chunk sizes 500–1,000 with 100–200 overlap; experiment per corpus.
- For DB ingestion, avoid dumping huge blobs; select columns that are actually useful.
- For **SQL** questions, prefer a separate **read-only SQL tool** (text-to-SQL) rather than RAG over rows—this starter indexes **rows as text** for semantic lookup.

---

## 7) Minimal frontend (optional)

You can add a React/Vite or simple HTML UI later to call `/chat` (not included here to keep it compact).

---

## 8) License

MIT — do whatever, but please be careful with API keys and PII.
