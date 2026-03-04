## NP News Hub Embeddings & Semantic Search

This document describes how we structure, embed, and query news articles in **NP News Hub** to enable **local, semantic search** over Nepali content.

The design assumes:
- All raw article metadata lives in `news.list`.
- Full article bodies live in `news.article`.
- Dense vector embeddings live in `news.article_embedding`.

---

## Data model

### Core tables

- **`news.list`** – basic article metadata and processing status.
  - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - `url TEXT UNIQUE NOT NULL`
  - `title TEXT`
  - `author TEXT`
  - `source TEXT`
  - `language TEXT`
  - `category TEXT`
  - `published_date DATE`
  - `published_time TIME WITH TIME ZONE`
  - `crawled_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP`
  - `status INTEGER NOT NULL` (1 = pending, 2 = article ingested, etc.)

- **`news.article`** – full article payloads.
  - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - `list_id UUID NOT NULL REFERENCES news.list(id) ON DELETE CASCADE`
  - `url TEXT NOT NULL`
  - `article JSONB NOT NULL` (currently `{ "content": "..." }`)
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP`

- **`news.article_embedding`** – semantic vectors for chunks of article content  
  (defined in `db/schema/news/embedding.sql`).
  - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - `list_id UUID NOT NULL REFERENCES news.list(id) ON DELETE CASCADE`
  - `article_id UUID NOT NULL REFERENCES news.article(id) ON DELETE CASCADE`
  - `chunk_id INTEGER NOT NULL` – stable 0‑based index per article
  - `chunk_text TEXT NOT NULL` – exact text used for the embedding
  - `category TEXT` – denormalized from `news.list`
  - `language TEXT` – denormalized from `news.list`
  - `published_date DATE` – denormalized from `news.list`
  - `embedding VECTOR(768) NOT NULL` – pgvector column (dimension matches chosen model)
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP`

Indexes:
- `UNIQUE (article_id, chunk_id)` – one row per chunk per article.
- B‑tree index on `(category, published_date)` – fast filtering.
- IVFFlat index on `embedding` (pgvector) – approximate nearest‑neighbour search.

Only articles with `news.list.status = 2` (fully crawled and stored in `news.article`) are eligible for embedding.

---

## Local embedding model

We use a **local sentence-transformer** that handles Nepali via multilingual training:

- **Model**: `sentence-transformers/paraphrase-multilingual-mpnet-base-v2`
  - Embedding dimension: **768** (this is why `VECTOR(768)` is used in `news.article_embedding`).
  - Good performance for semantic similarity across many languages, including Nepali.
  - Runs on CPU; can later be moved to GPU without changing the schema or APIs.

### Python environment (embedding service)

Create a new module, e.g. `embedding/`, with a dedicated environment:

- `embedding/requirements.txt` (high-level):
  - `sentence-transformers`
  - `torch` (or `onnxruntime` if you want ONNX)
  - `psycopg2-binary` (or `asyncpg`) for Postgres
  - `python-dotenv` for configuration

Configuration (via `.env` or similar):
- `DATABASE_URL` – same Supabase connection string used in `db`/`core`.
- `EMBEDDING_BATCH_SIZE` – e.g. `32` or `64`.
- `EMBEDDING_MODEL_NAME` – default `sentence-transformers/paraphrase-multilingual-mpnet-base-v2`.

The embedding Python code will:
- Load the model once at process start.
- Reuse it across many batches to avoid repeated initialization.

---

## Embedding pipeline

### What gets embedded

We embed **only articles that are fully crawled**:

1. `news.list.status = 2`.
2. A corresponding row exists in `news.article` with non‑empty `article->>'content'`.
3. No existing embeddings for that `article_id` (or we treat existing rows as stale and overwrite).

### Chunking strategy

To get good semantic granularity and stable performance:

- Extract `content` from `news.article.article->>'content'`.
- Normalize whitespace (remove duplicate spaces and blank lines).
- Split into paragraphs by double newlines (`\n\n`).
- Merge very short neighbouring paragraphs to avoid ultra‑short chunks.
- Ensure each final `chunk_text` is roughly **512–1024 characters** (or ≲ 256–512 tokens).
- Number chunks per article starting at `chunk_id = 0`.

Each chunk gets:
- `list_id`, `article_id`, `chunk_id`
- `chunk_text`
- `category`, `language`, `published_date` copied from `news.list`
- `embedding` – vector from the model

### Batch processing

The indexing script (e.g. `embedding/index_articles.py`) should:

1. Connect to Postgres using `DATABASE_URL`.
2. Select a batch of candidate articles:
   - `status = 2`
   - Without rows in `news.article_embedding` (e.g. LEFT JOIN with `news.article_embedding` and filter `WHERE article_embedding.article_id IS NULL`).
   - Limit by a configurable batch size (e.g. 100 articles at a time).
3. For each article in the batch:
   - Load `content`, `category`, `language`, `published_date`.
   - Produce a list of `(chunk_id, chunk_text)` pairs.
4. For all chunks across the batch:
   - Run embeddings in model‑sized batches (`EMBEDDING_BATCH_SIZE`).
   - Insert rows into `news.article_embedding` using a single `INSERT ... VALUES ...` per batch.
5. Commit the transaction; repeat until there are no more candidates.

The script should be **idempotent**: if it crashes mid‑batch, re‑running it will simply skip already‑embedded articles because of the uniqueness constraint on `(article_id, chunk_id)`.

### Scheduling

There are two options:

- **External scheduler**: Run `python embedding/index_articles.py` periodically using Windows Task Scheduler or a cron‑like service.
- **Crawler‑triggered**: After each crawler cycle (once `status` flips to 2), trigger the embedding script (synchronously or asynchronously).

The design does not force one or the other; both reuse the same script.

---

## Query‑time semantic search

### High‑level flow

1. Client sends a query to the core API, e.g. `POST /v1/search/query` with:
   - `query` (string)
   - Optional filters: `category`, `from_date`, `to_date`, `language`, `max_results`.
2. The core API obtains a **query embedding** using the same model.
3. Core runs a similarity search over `news.article_embedding.embedding` in Postgres.
4. For the top‑K hits, core joins back to `news.list` (and optionally `news.article`) and returns ranked results with snippets.

### Embedding the query

We keep the embedding logic in Python and expose it to the Go core via a lightweight HTTP service:

- Small FastAPI/Flask app (e.g. `embedding/service.py`) that:
  - Loads the same `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` model.
  - Exposes `POST /embed`:
    - Request: `{ "texts": ["query text 1", "query text 2", ...] }`
    - Response: `{ "vectors": [[...768 floats...], ...] }`
- Core Go service calls this endpoint for user queries and receives a single `[]float32` embedding.

This keeps Go free of heavy ML deps and ensures both offline indexing and online search use **identical** embeddings.

### Postgres similarity query

Given a query embedding `:q` (a 768‑dimensional vector), search might look like:

- Base condition:
  - `FROM news.article_embedding ae`
  - Optional filters:
    - `WHERE (COALESCE(ae.category, '') = :category OR :category IS NULL)`
    - `AND (ae.language = :language OR :language IS NULL)`
    - `AND (ae.published_date BETWEEN :from_date AND :to_date OR (:from_date IS NULL AND :to_date IS NULL))`
- Similarity ordering (pgvector):
  - `ORDER BY ae.embedding <-> :q`
- Limit:
  - `LIMIT :k` (e.g. 20–50).

For each matching row we fetch:
- `ae.list_id`, `ae.article_id`, `ae.chunk_id`
- `ae.chunk_text` (used as the snippet)
- Similarity `score` (can be derived from `<->`).
- Joined metadata from `news.list`: `title`, `url`, `category`, `published_date`.

The core API returns a JSON array of results ordered by `score`.

---

## Planned API surface (core)

We will add a minimal search module to the Go core, e.g. `core/v1/search`:

- **Endpoint**: `POST /v1/search/query`
  - **Request**:
    - `query` (string, required)
    - `category` (string, optional)
    - `language` (string, optional; default `"ne"`)
    - `from_date` / `to_date` (ISO dates, optional)
    - `max_results` (int, optional; default 20)
  - **Response**: JSON with:
    - `results`: list of:
      - `list_id`
      - `article_id`
      - `score` (float)
      - `title`
      - `url`
      - `category`
      - `published_date`
      - `snippet` (from `chunk_text`)

Implementation outline in core:
- Handler:
  - Validate input and defaults.
  - Call embedding service `/embed` with `[query]`.
  - Run SQL similarity query against `news.article_embedding`.
  - Marshal rows into the response shape above.
- Config:
  - `EMBEDDING_SERVICE_URL` for the Python service.

---

## Summary

- Articles are crawled into `news.list` and `news.article`.
- We create semantic **chunk‑level embeddings** in `news.article_embedding` using a local multilingual sentence‑transformer.
- A Python indexing script keeps embeddings in sync with new fully‑crawled articles.
- A lightweight Python embedding service provides query embeddings to the Go core.
- The core API performs vector similarity search in Postgres and returns ranked, filtered results with meaningful Nepali snippets.

