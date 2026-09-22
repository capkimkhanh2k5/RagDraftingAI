# RagDraftingAI

> An AI-powered assistant for **Vietnamese administrative document drafting and legal question answering**, built with a custom Retrieval-Augmented Generation (RAG) pipeline.

RagDraftingAI combines semantic search, lexical retrieval, reranking, large language models, OCR, document templates, and a full-stack web application to help users search Vietnamese legal knowledge and generate formatted administrative documents.

The current implementation is a multi-service application consisting of a **React frontend**, a **FastAPI application backend**, a dedicated **RAG service**, and **PostgreSQL**, orchestrated with Docker Compose.

---

## ✨ Key Features

### ⚖️ Legal Question Answering

- Retrieval-grounded legal Q&A over a Vietnamese legal knowledge base.
- Multi-turn conversation context.
- Token streaming to the client using NDJSON.
- Legal source metadata returned with generated answers.
- Configurable LLM model selection.

### 📝 Administrative Document Drafting

The drafting workflow can:

1. Understand the user's drafting request.
2. Detect the appropriate administrative document type.
3. Retrieve relevant:
   - legal provisions,
   - document forms,
   - example documents.
4. Generate structured document fields with an LLM.
5. Fill those fields into an existing `.docx` template.
6. Preserve the formatting of the original Word template.
7. Upload the generated document to cloud storage and persist its metadata.

Supported template categories currently include:

- Individual resolutions
- Individual decisions
- Regulatory decisions
- Named administrative documents
- Official dispatches
- Urgent dispatches
- Invitations
- Introduction letters
- Meeting minutes
- Leave forms

### 🔎 Custom Hybrid RAG Pipeline

The project implements its own retrieval pipeline rather than relying on a high-level RAG orchestration framework.

The pipeline combines:

- **BM25 lexical retrieval**
- **Dense vector retrieval with ChromaDB**
- **Reciprocal Rank Fusion (RRF)**
- **Cross-Encoder reranking**
- Metadata-aware filtering
- Deduplication and document-level result limiting

Embedding model:

```text
intfloat/multilingual-e5-large-instruct
```

Reranking model:

```text
BAAI/bge-reranker-v2-m3
```

The knowledge base is logically separated into:

```text
legal
forms
examples
```

This allows the drafting pipeline to independently retrieve legal grounding, document structure, and representative examples.

### 🇻🇳 Vietnamese Administrative Document Routing

The RAG layer contains domain-specific routing logic for Vietnamese administrative documents.

For example, it distinguishes between different types of `Quyết định` using query semantics before retrieval, reducing ambiguity between individual decisions and regulatory/indirect decisions.

If the query cannot be classified confidently, retrieval and reranking are allowed to make the final selection.

### 📚 Knowledge Base Ingestion

The RAG service supports runtime knowledge-base maintenance:

- ingest a legal document,
- extract document metadata,
- split content by articles and clauses,
- chunk long legal text,
- generate embeddings,
- upsert chunks into ChromaDB,
- delete documents or individual articles,
- batch delete documents,
- rebuild the BM25 index,
- inspect database state.

The ingestion pipeline also includes heuristics for Vietnamese legal document numbers, document types, titles, chapters, and article boundaries.

### 📄 File Upload & OCR

Users can upload documents through Cloudinary-backed storage.

Text extraction currently supports:

- TXT / Markdown
- CSV / TSV
- JSON
- XML
- PDF
- DOCX
- Images

For scanned PDFs and images, the backend uses **Tesseract OCR** with Vietnamese and English language support.

PDF files are first processed with PyMuPDF. If the extracted text density is too low, the backend automatically falls back to OCR.

### ☁️ Cloud Document Storage

Cloudinary is used for:

- user file uploads,
- generated document storage,
- signed browser uploads,
- document preview/download URLs.

Uploaded assets are namespaced by user and chat session, and backend validation prevents users from accessing another user's uploaded asset namespace.

### 🔐 Authentication & Authorization

Authentication includes:

- email/password registration and login,
- JWT access tokens,
- Google OAuth,
- role-protected routes,
- separate user and administrator interfaces,
- login rate limiting.

### 💬 Chat Workspace

The frontend provides a conversational workspace with:

- persistent chat sessions,
- multi-turn legal Q&A,
- streaming AI responses,
- separate **Q&A** and **Document Generation** modes,
- selectable LLM models,
- additional drafting instructions,
- file attachments,
- document preview,
- AI response feedback,
- chat history.

### 🛠️ Administration Console

The administrator interface includes modules for:

- dashboard analytics,
- user management,
- knowledge-base management,
- AI performance monitoring,
- prompt template management,
- audit logs,
- system health,
- CPU / RAM / NVIDIA GPU monitoring,
- application settings.

---

# 🏗️ Architecture

```mermaid
flowchart LR
    U[User Browser]

    subgraph FE["Frontend"]
        React["React 19 + TypeScript\nVite + Tailwind CSS"]
    end

    subgraph BE["Application Backend"]
        API["FastAPI\nREST API"]
        Auth["JWT / Google OAuth"]
        OCR["Document Processing\nOCR / DOCX / PDF"]
        Admin["Admin & Monitoring"]
    end

    subgraph RAG["RAG Service"]
        Prompt["Prompt / Domain Logic"]
        BM25["BM25"]
        Dense["Dense Retrieval"]
        RRF["Reciprocal Rank Fusion"]
        Rerank["CrossEncoder Reranker"]
        LLM["Groq / OpenAI"]
        Docx["DOCX Template Filler"]
    end

    PG[(PostgreSQL)]
    Chroma[(ChromaDB)]
    Cloud[(Cloudinary)]

    U --> React
    React --> API

    API --> Auth
    API --> OCR
    API --> Admin
    API --> PG
    API --> Cloud
    API --> RAG

    Prompt --> BM25
    Prompt --> Dense
    BM25 --> RRF
    Dense --> RRF
    RRF --> Rerank
    Dense --> Chroma
    Rerank --> LLM
    LLM --> Docx
```

The RAG service is intentionally isolated from the application backend because loading embedding and reranking models is significantly more resource-intensive than serving ordinary application API requests.

---

# 🧠 RAG Retrieval Flow

```text
User Query
   │
   ├── Query normalization
   │
   ├── Administrative-form detection
   │
   ├── BM25 lexical retrieval
   │
   ├── Dense embedding retrieval
   │
   └──────────────┐
                  ▼
        Reciprocal Rank Fusion
                  │
                  ▼
        CrossEncoder reranking
                  │
                  ▼
       Top relevant contexts
          ├── Legal chunks
          ├── Form template
          └── Similar examples
                  │
                  ▼
             Prompt Builder
                  │
                  ▼
          Groq / OpenAI LLM
                  │
         ┌────────┴────────┐
         ▼                 ▼
   Legal Answer      Structured Fields
                           │
                           ▼
                    DOCX Template Filler
                           │
                           ▼
                   Generated Document
```

---

# 🧩 Shared RAG Resources

The RAG process maintains shared singleton resources for:

- the embedding model,
- the ChromaDB client,
- the document updater.

This prevents the query pipeline and knowledge-base ingestion pipeline from independently loading duplicate embedding models.

Besides reducing memory usage, it also avoids multiple competing ChromaDB clients writing to the same persistent database.

---

# 🤖 LLM Configuration

The current implementation supports **Groq** and **OpenAI**.

Default Groq models:

| Use case | Model |
|---|---|
| Document drafting | `meta-llama/llama-4-scout-17b-16e-instruct` |
| Legal Q&A | `llama-3.3-70b-versatile` |

If Groq credentials are not configured, the RAG layer can use an OpenAI model configured through `LLM_MODEL`.

Default OpenAI model:

```text
gpt-4o-mini
```

Runtime LLM parameters such as `max_tokens` and `temperature` can also be updated through the RAG API.

---

# 🛠️ Tech Stack

## Frontend

| Technology | Purpose |
|---|---|
| React 19 | User interface |
| TypeScript | Type-safe frontend development |
| Vite | Development/build tooling |
| Tailwind CSS 4 | Styling |
| React Router | Client-side routing |
| Motion | UI animations |
| React Markdown | Rendering AI Markdown responses |
| Recharts | Admin analytics visualization |
| Google OAuth | Google authentication |
| docx-preview | Word document preview |

## Application Backend

| Technology | Purpose |
|---|---|
| Python 3.11 | Backend runtime |
| FastAPI | REST API |
| SQLAlchemy | ORM |
| PostgreSQL | Application database |
| Alembic | Database migrations |
| JWT / python-jose | Authentication |
| Passlib / bcrypt | Password hashing |
| SlowAPI | Rate limiting |
| HTTPX | Async service communication |
| Cloudinary | Cloud document storage |
| PyMuPDF | PDF parsing |
| Tesseract OCR | OCR for scanned documents/images |
| python-docx | DOCX parsing and generation |
| psutil / NVML | System monitoring |

## RAG / AI Service

| Technology | Purpose |
|---|---|
| Python 3.10 | RAG runtime |
| FastAPI | RAG service API |
| ChromaDB | Persistent vector database |
| Sentence Transformers | Dense embeddings |
| `multilingual-e5-large-instruct` | Embedding model |
| `bge-reranker-v2-m3` | Cross-Encoder reranking |
| rank-bm25 | Lexical retrieval |
| Reciprocal Rank Fusion | Hybrid ranking |
| PyTorch | Model execution |
| Groq API | Primary LLM provider |
| OpenAI API | Alternative LLM provider |
| pandas / PyArrow | Knowledge-base processing |

## Infrastructure

| Technology | Purpose |
|---|---|
| Docker | Service containerization |
| Docker Compose | Local orchestration |
| PostgreSQL 15 | Local relational database |
| pgAdmin | Database administration |
| Cloudinary | External object/document storage |
| Aiven-compatible PostgreSQL | Optional cloud database |

---

# 📁 Project Structure

```text
RagDraftingAI/
│
├── BACKEND/
│   ├── app/
│   │   ├── api/              # REST API routes
│   │   ├── core/             # Configuration, auth, rate limiting
│   │   ├── db/               # SQLAlchemy database setup
│   │   ├── models/           # Database models
│   │   ├── repositories/     # Persistence layer
│   │   ├── schemas/          # Pydantic request/response schemas
│   │   └── services/         # Application/business services
│   ├── requirements.txt
│   └── Dockerfile
│
├── FRONTEND/
│   ├── src/
│   │   ├── components/       # Shared UI + admin views
│   │   ├── lib/              # API/auth/state helpers
│   │   ├── Chat.tsx          # Main AI workspace
│   │   ├── AdminConsoleApp.tsx
│   │   └── App.tsx
│   ├── package.json
│   └── Dockerfile
│
├── RAG/
│   ├── main.py               # RAG FastAPI service
│   ├── hybrid_retrieval.py   # Hybrid retrieval pipeline
│   ├── promptApi.py          # Drafting / legal-QA orchestration
│   ├── promptTemplates.py    # Prompt construction
│   ├── updateDB.py           # Knowledge-base ingestion/update
│   ├── app_state.py          # Shared AI resources
│   ├── docx_filler.py        # Word-template generation
│   ├── dataset/              # Runtime RAG indexes*
│   ├── models/               # Local embedding/reranker models*
│   ├── requirements.txt
│   └── Dockerfile
│
├── docs/                     # Project documentation
├── docker-compose.yml
├── .env.example
└── README.md
```

`RAG/dataset`, `RAG/models`, and generated drafts are intentionally excluded from Git.

---

# 🚀 Running the Project

## Prerequisites

Recommended:

- Docker
- Docker Compose
- at least **16 GB RAM** for the full local stack
- sufficient disk space for embedding and reranking models

A CUDA-compatible GPU is optional. The RAG service automatically selects CUDA when available and otherwise runs on CPU.

---

## 1. Clone the Repository

```bash
git clone https://github.com/anh130305/RagDraftingAI.git
cd RagDraftingAI
```

---

## 2. Configure Environment Variables

Create the root Compose environment:

```bash
cp .env.example .env
```

Configure at least:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_password
POSTGRES_DB=rag_db
DATABASE_URL=postgresql://postgres:your_password@DataBase:5432/rag_db

VITE_API_URL=http://localhost:8000
VITE_GOOGLE_CLIENT_ID=your_google_client_id
```

The backend also has its own example configuration:

```bash
cp BACKEND/.env.example BACKEND/.env
```

Important backend configuration includes:

```env
SECRET_KEY=replace_with_a_secure_secret
DATABASE_URL=postgresql://postgres:your_password@DataBase:5432/rag_db

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

RAG_SERVICE_URL=http://rag_service:8001
```

Create `RAG/.env` for LLM credentials:

```env
GROQ_API_KEY=
OPENAI_API_KEY=
LLM_MODEL=gpt-4o-mini

# Optional when accessing private/gated Hugging Face assets
HF_TOKEN=
```

Do **not** commit API keys or production secrets.

---

## 3. Prepare RAG Runtime Assets

The repository intentionally excludes the large runtime directories:

```text
RAG/models/
RAG/dataset/
```

The current RAG implementation expects local copies of:

```text
RAG/models/embedding/
RAG/models/reranker/
```

corresponding to:

```text
intfloat/multilingual-e5-large-instruct
BAAI/bge-reranker-v2-m3
```

The service currently loads these models with local-file mode enabled, so a fresh clone must provision the model files before the full RAG service can start successfully.

The RAG database/index directories are expected under:

```text
RAG/dataset/chromadb/
RAG/dataset/bm25/
RAG/dataset/chunks/
```

These assets are not stored in Git because of their size.

---

## 4. Start the Stack

```bash
docker compose up --build
```

Services:

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:8000 |
| Swagger API Docs | http://localhost:8000/docs |
| RAG Service | http://localhost:8001 |
| RAG Health Check | http://localhost:8001/api/v1/rag/health |
| PostgreSQL | localhost:5432 |
| pgAdmin | http://localhost:5050 |

---

# 🔌 Main API Areas

The application backend exposes versioned APIs under:

```text
/api/v1
```

Major route groups include:

```text
/api/v1/auth
/api/v1/users
/api/v1/chat
/api/v1/drafting
/api/v1/documents
/api/v1/admin
/api/v1/internal
```

Important RAG endpoints include:

```text
POST /api/v1/rag/draft
POST /api/v1/rag/legal_qa
POST /api/v1/rag/legal_qa_stream

GET  /api/v1/rag/health
GET  /api/v1/rag/llm_config
PUT  /api/v1/rag/llm_config
```

Knowledge-base maintenance endpoints include:

```text
GET  /api/v1/db/status
GET  /api/v1/db/check/{so_hieu}
POST /api/v1/db/ingest
POST /api/v1/db/delete_doc
POST /api/v1/db/delete_article
POST /api/v1/db/batch_delete
POST /api/v1/db/rebuild_bm25
GET  /api/v1/db/rebuild_bm25/status
```

---

# 📄 DOCX Generation

Document templates use placeholders such as:

```text
{{TEN_CQ_BAN_HANH}}
{{SO_KY_HIEU}}
{{NOI_DUNG_DIEU_1}}
```

The generated structured fields from the LLM are mapped onto those placeholders.

Unlike simple string replacement, the DOCX filler works with Word XML runs so placeholders that are split across multiple runs can still be replaced while retaining the template's original formatting.

It processes:

- body paragraphs,
- nested tables,
- headers,
- footers,
- multiline generated content.

---

# 📊 Monitoring & Observability

The backend records operational data including:

- query latency,
- selected LLM model,
- errors,
- whether retrieval found legal chunks,
- user feedback,
- audit events.

The administrator dashboard exposes aggregated statistics and AI monitoring data.

System monitoring supports:

- CPU usage,
- RAM usage,
- NVIDIA GPU utilization,
- VRAM usage.

---

# 🔒 Security Considerations

The current implementation includes:

- JWT authentication,
- password hashing,
- Google identity verification,
- role-based route guards,
- login rate limiting,
- Cloudinary asset ownership validation,
- CORS configuration,
- audit logging.

Before a production deployment, replace all development/default secrets and review the deployment-specific CORS, OAuth, database, and storage configuration.

---

# 💡 Engineering Highlights

A few technically notable parts of the implementation are:

- **Custom RAG stack** instead of hiding retrieval behind a high-level framework.
- **Hybrid BM25 + semantic retrieval** with RRF and CrossEncoder reranking.
- **Vietnamese-domain-specific routing** for administrative document forms.
- **Runtime legal knowledge ingestion** with structured article/clause chunking.
- **Shared embedding-model state** to avoid duplicate multi-gigabyte model instances.
- **Streaming multi-turn legal Q&A** through a separate AI service.
- **OCR fallback for scanned PDFs**.
- **Template-aware DOCX generation** that preserves Word formatting.
- **Separated application and AI services**, allowing each component to scale independently.
- **Role-based administration and AI observability** beyond the core chatbot interface.

---

# 📌 Current Project Scope

RagDraftingAI currently focuses on two primary workflows:

### Legal Assistant

```text
Question → Retrieve Vietnamese legal context → Rerank → LLM → Grounded answer
```

### Administrative Drafting Assistant

```text
Draft request
→ Detect document form
→ Retrieve law + template + examples
→ Generate structured fields
→ Fill DOCX template
→ Upload/store generated document
```

---

## License

No explicit open-source license is currently included in the repository.

Unless a license is added, normal copyright restrictions apply to the source code.
