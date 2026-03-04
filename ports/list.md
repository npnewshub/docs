# Ports used in NP News Hub

| Component | Default port | Env / config | Notes |
|-----------|--------------|--------------|--------|
| **Core** | **8701** | `PORT` | Go API (news list, article, embedding, search). Crawler, engine, and search call this. |
| **Crawler** | — | — | No server. Script calls Core at `CORE_API_URL` (default 8701). |
| **Engine** | — | — | No server. Script calls Core (8701) and LLMs/MPNet (8801). |
| **LLMs (MPNet)** | **8801** | `MPNET_PORT` | Python embedding service. Core search and engine use it for embeddings. |
| **Search** | **8901** | `PORT` | Go service: semantic search + Gemini. Frontend calls this for Q&A. |
| **Web** | **3000** | Next.js default | Next.js app (`next dev`). Talks to Search (8901) for /ask. |

## Quick reference

- **8701** – Core API (DB, news, embeddings ingest, semantic search)
- **8801** – MPNet embedding model (embed text → vector)
- **8901** – Search service (core + Gemini; frontend → `/ask`)
- **3000** – Web UI (Next.js)

## Run order (local)

1. Core: `cd core && go run main.go` → 8701  
2. MPNet: `cd LLMs/mpnet && python service.py` → 8801  
3. Search: `cd search && go run main.go` → 8901  
4. Web: `cd web && npm run dev` → 3000  

Crawler and engine are run on demand; they use Core (and engine uses MPNet) and do not listen on a port.
