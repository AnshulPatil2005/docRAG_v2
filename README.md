# docRAG v2 — Async Pipeline

A document question-answering system that extracts text from PDFs using OCR and enables semantic search and natural-language Q&A through a REST API and an Angular frontend.

Upload a PDF, wait for it to process, then ask questions about it in plain English and get an answer with citations back to the source chunks.

This is the same retrieval model as [`docRAG v1`](https://github.com/AnshulPatil2005/docRAG_v1_IDP) — chunk, embed, cosine-search, generate — with the engineering rebuilt around it: ingestion moved off the request path and into Celery workers (upload returns a `task_id` immediately instead of blocking 30+ seconds), PyMuPDF-only extraction was replaced with page-by-page doctr OCR, a real Angular frontend replaced the bare tester page, and OpenRouter was added alongside Ollama. **Retrieval is still flat vector search over undifferentiated chunks** — none of this touches v1's core limitation (no section/citation awareness). That's what [`docRAG v3`](https://github.com/AnshulPatil2005/docRAG_v3) exists to fix with a knowledge-graph layer.

## How It Works

```
                              ┌────────────────────────────────────────┐
                              │              INGESTION                  │
                              │                                          │
 PDF ─upload─▶ FastAPI ─▶ Celery task ─▶ OCR (doctr) ─▶ Chunk text       │
                              │                              │           │
                              │                              ▼           │
                              │              Sentence-Transformers embed │
                              │                              │           │
                              │                              ▼           │
                              │                   Store vectors in Qdrant│
                              └────────────────────────────────────────┘

                              ┌────────────────────────────────────────┐
                              │                QUERY                    │
                              │                                          │
 Question ─▶ FastAPI /chat ─▶ Embed query (same model) ─▶ Vector search │
                              │  in Qdrant (top-K chunks) ──▶ Stuff into │
                              │  LLM prompt ──▶ Answer + citations       │
                              └────────────────────────────────────────┘
```

Two different models are used, for two different jobs:

1. **Embedding model** (local, free, always `sentence-transformers`) — turns text into vectors for semantic search. Runs the same way for both indexing and querying, so vectors stay comparable.
2. **LLM** (local via Ollama, or cloud via OpenRouter) — turns retrieved chunks + the question into a natural-language answer.

Everything after upload runs asynchronously: the API hands the PDF off to a Celery worker immediately and returns a `task_id`/`doc_id` you poll for status, so a slow OCR pass on a large PDF never blocks the HTTP request.

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| API | FastAPI | REST endpoints, request validation, rate limiting |
| Task queue | Celery + Redis | Async PDF processing so uploads don't block on OCR |
| OCR | doctr (`python-doctr`) | Extracts text from scanned/image-based PDF pages |
| Vector store | Qdrant | Stores chunk embeddings, serves top-K similarity search |
| Embeddings | Sentence-Transformers | Local, free text-to-vector model (`all-MiniLM-L6-v2` by default) |
| LLM | Ollama (local) or OpenRouter (cloud) | Generates the final answer from retrieved context |
| Frontend | Angular 17 | Standalone components + signals, talks to the API over HTTP |
| Rate limiting | slowapi | Per-IP limits on `/upload` and `/chat` |
| Logging | structlog | Structured JSON logs from the API and worker |

## Project Structure

```
docRAG/
├── app/                        # Backend Python application
│   ├── api/
│   │   ├── main.py            # FastAPI app, CORS, rate limiter, router mounting
│   │   └── routes.py          # /chat, /upload, /status, /health endpoints
│   ├── core/
│   │   └── config.py          # Pydantic Settings, reads .env
│   ├── services/
│   │   ├── ocr.py             # doctr text extraction
│   │   ├── text_processing.py # chunking
│   │   ├── embeddings.py      # Sentence-Transformers model loading + encode
│   │   ├── vector_store.py    # Qdrant upsert/search helpers
│   │   └── llm.py             # Ollama / OpenRouter client
│   └── worker/
│       ├── celery_app.py      # Celery app + Redis broker config
│       └── tasks.py           # process_pdf_task: OCR → chunk → embed → store
├── frontend-angular/            # Angular 17 frontend
│   ├── src/app/
│   │   ├── components/        # upload, status, chat, recent-tasks, header
│   │   ├── services/          # api.service.ts — all HTTP calls + shared state
│   │   └── models/             # TypeScript interfaces for API payloads
│   ├── Dockerfile
│   └── nginx.conf
├── scripts/                     # CLI helpers (load_small_pdf.sh, query_rag.sh, ...)
├── tests/
│   ├── test_api.py             # API endpoint tests
│   └── data/                   # Sample PDFs used by tests
├── docker-compose.yml           # redis, qdrant, ollama, api, worker
├── Dockerfile                    # Backend image (api + worker share this)
├── Makefile                      # up / down / build / logs / test shortcuts
└── requirements.txt
```

## Requirements

- Docker & Docker Compose (for the backend and its dependencies)
- Node.js 18+ (for frontend development)
- (Optional) `make` for the `Makefile` shortcuts

## Setup

### Backend Configuration

Create a `.env` file in the project root to configure your models.

#### LLM Configuration

Choose one of the following LLM providers:

**Option 1: Ollama (Local — Free)**

```env
LLM_PROVIDER=ollama
LLM_MODEL=llama3
```

After starting Docker, pull the model:
```bash
docker-compose exec ollama ollama pull llama3
```

Available Ollama models: `llama3`, `llama3.2`, `mistral`, `codellama`, `phi3`, etc.

**Option 2: OpenRouter (Cloud — Paid)**

```env
LLM_PROVIDER=openrouter
LLM_MODEL=google/gemini-2.0-flash-001
OPENROUTER_API_KEY=sk-or-v1-your-key-here
```

Get your API key at https://openrouter.ai/keys.

Popular OpenRouter models:
- `google/gemini-2.0-flash-001` — fast and cheap
- `anthropic/claude-3.5-sonnet` — high quality
- `meta-llama/llama-3-70b-instruct` — open source
- `openai/gpt-4o-mini` — good balance

See all models at https://openrouter.ai/models. Model IDs on OpenRouter change over time (added, renamed, retired) — if you get a `404 No endpoints found for <model>` error, the `LLM_MODEL` value is no longer valid; check the live list rather than trusting any hardcoded list.

#### Embedding Model Configuration

The embedding model runs locally and is configured via `EMBEDDING_MODEL`. Default is `all-MiniLM-L6-v2`.

```env
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

To change it:

1. Update `.env`:
   ```env
   EMBEDDING_MODEL=all-mpnet-base-v2
   ```
2. Restart the services:
   ```bash
   docker-compose restart api worker
   ```
3. **Important**: if you change the embedding model after uploading documents, you must re-upload them — different models produce incompatible vectors, and old vectors won't be replaced automatically.

Available embedding models (from [Sentence Transformers](https://www.sbert.net/docs/pretrained_models.html)):

| Model | Dimensions | Speed | Quality |
|-------|------------|-------|---------|
| `all-MiniLM-L6-v2` | 384 | Fast | Good (default) |
| `all-MiniLM-L12-v2` | 384 | Medium | Better |
| `all-mpnet-base-v2` | 768 | Slow | Best |
| `paraphrase-MiniLM-L6-v2` | 384 | Fast | Good for paraphrasing |

#### Complete `.env` Example

```env
# LLM Configuration
LLM_PROVIDER=openrouter          # or "ollama"
LLM_MODEL=google/gemini-2.0-flash-001
OPENROUTER_API_KEY=sk-or-v1-your-key-here

# Embedding Configuration
EMBEDDING_MODEL=all-MiniLM-L6-v2

# Optional: RAG tuning
RAG_TOP_K=5                      # Number of chunks to retrieve
CHUNK_TOKENS=500                 # Size of text chunks
CHUNK_OVERLAP_TOKENS=50          # Overlap between chunks

# Optional: Qdrant (defaults to the local docker-compose instance)
QDRANT_URL=http://qdrant:6333
QDRANT_API_KEY=                  # only needed for Qdrant Cloud
```

## Running

### Backend (Docker)

```bash
docker-compose up -d --build
```

This launches:

| Service | Port | Notes |
|---|---|---|
| Redis | 6379 | Celery broker/backend |
| Qdrant | 6333 | Vector database |
| Ollama | 11434 | Local LLM inference — only used if `LLM_PROVIDER=ollama` |
| API | 8000 | FastAPI backend |
| Worker | — | Celery background processing, no exposed port |

Or with the Makefile: `make up` / `make down` / `make build` / `make logs`.

### Frontend (Angular)

**Local development:**
```bash
cd frontend-angular
npm install
npm start
```
Runs at http://localhost:4200 and talks to the backend at http://localhost:8000.

**Production build:**
```bash
cd frontend-angular
npm run build
```
Output lands in `frontend-angular/dist/docrag-frontend/browser/`.

**Docker (optional):**
```bash
cd frontend-angular
docker build -t docrag-frontend .
docker run -p 8080:80 docrag-frontend
```

## Deployment

### Backend on Render

1. Create a new Web Service on Render, connect the repository.
2. Build command: `docker build -t app .`
3. Start command: as defined by the `Dockerfile`.
4. Add the environment variables from your `.env` file.

CORS is configured to accept requests from any origin (`allow_origins=["*"]`) so no extra config is needed on the frontend side.

### Frontend on Vercel

1. Import the repository, set the root directory to `frontend-angular`.
2. Framework preset: Angular. Build command: `npm run build`. Output directory: `dist/docrag-frontend/browser`.
3. Update the deployed backend URL in two places if it differs from the default:
   - `frontend-angular/src/app/services/api.service.ts` — `DEFAULT_API_URL` constant
   - `frontend-angular/src/index.html` — default input value

### Keep-Alive for Render's Free Tier

The frontend pings the backend every 14 minutes while open in a browser, to stop Render's free tier from sleeping after 15 minutes of inactivity. For uptime without the frontend open, point an external monitor (e.g. UptimeRobot) at `/api/v1/health`.

## Usage

### Web Interface

Open the frontend (local: http://localhost:4200, or your deployed URL). Features:

- API URL configuration with an online/offline status indicator
- Drag & drop PDF upload, with optional force re-processing
- Task status monitoring with auto-refresh
- Chat/query interface with citation display
- Recent tasks list with quick actions

**Quick start:**

1. **Upload a PDF** — drag & drop or click to select, then "Upload PDF". Note the returned `task_id` and `doc_id`.
2. **Monitor processing** — the task ID auto-fills for status checking; status moves PENDING → STARTED → SUCCESS.
3. **Query** — ask a question in the chat box, optionally scoped to one `doc_id`; view the answer plus its source citations.

### Command Line

```bash
./scripts/load_small_pdf.sh "/path/to/document.pdf"
./scripts/query_rag.sh "What is the main topic of the document?"
```

## API Reference

All endpoints are prefixed with `/api/v1`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/health` | GET | Health check |
| `/api/v1/upload` | POST | Upload a PDF for processing (rate-limited: 5/min) |
| `/api/v1/status/{task_id}` | GET | Check processing status |
| `/api/v1/chat` | POST | Query documents with natural language (rate-limited: 20/min) |

### Example API Calls

**Upload a PDF:**
```bash
curl -X POST "http://localhost:8000/api/v1/upload" \
  -F "file=@document.pdf"
```
```json
{
  "message": "File uploaded and processing started.",
  "doc_id": "3f9c2a1e...",
  "task_id": "b1a2c3d4-..."
}
```
`doc_id` is the SHA-256 hash of the file — re-uploading the same PDF is a no-op unless `force=true` is passed as a query param.

**Check status:**
```bash
curl "http://localhost:8000/api/v1/status/{task_id}"
```
```json
{ "task_id": "b1a2c3d4-...", "status": "SUCCESS", "result": { "...": "..." } }
```

**Chat query:**
```bash
curl -X POST "http://localhost:8000/api/v1/chat" \
  -H "Content-Type: application/json" \
  -d '{"query": "What is this document about?", "doc_id": "optional-doc-id"}'
```
```json
{
  "answer": "The document describes ...",
  "citations": [
    { "doc_id": "3f9c2a1e...", "page": 2, "filename": "3f9c2a1e....pdf", "text_snippet": "..." }
  ]
}
```

## Development

### Backend

```bash
docker-compose run api pytest    # run tests (tests/test_api.py)
docker-compose logs -f api       # API logs
docker-compose logs -f worker    # worker logs
docker-compose logs -f ollama    # LLM logs (if using Ollama)
```

### Frontend

Angular 17 with standalone components, signals for reactive state, SCSS for styling, Inter for typography.

Key files:
- `src/app/services/api.service.ts` — all API communication and shared client state
- `src/app/components/` — `upload`, `status`, `chat`, `recent-tasks`, `header`
- `src/styles.scss` — global styles and CSS variables

## Troubleshooting

**`model 'llama3' not found`** — Pull the model: `docker-compose exec ollama ollama pull llama3`

**API shows "Offline" in the frontend** — Check `docker-compose ps`, check `docker-compose logs api`, verify the API URL in the frontend header, and (if deployed) make sure the backend isn't asleep.

**CORS errors in the browser console** — CORS is enabled for all origins already; check for a trailing slash mismatch, or that the backend actually responded (may be sleeping on Render's free tier).

**OCR processing fails** — Check `docker-compose logs -f worker`.

**Status stays "PENDING"** — Confirm the worker is running (`docker-compose ps`), check its logs, and confirm Redis/Qdrant are healthy.

**Connection errors to Qdrant** — `curl http://localhost:6333/collections`

**No results when querying** — Confirm the document finished processing (status: SUCCESS), check that embeddings were generated (worker logs), try a more specific question.

**Changed embedding model but search doesn't work** — Old vectors are incompatible with a new embedding model; re-upload documents to regenerate them.

**Frontend build fails**
```bash
cd frontend-angular
rm -rf node_modules package-lock.json
npm install
npm run build
```

**Render backend sleeping** — First request after 15 minutes of inactivity can take 30–60 seconds. Keep the frontend open for the built-in keep-alive, or use an external monitor.
