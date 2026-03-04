## Embedding model design

This document goes deeper into the **embedding model** we use for NP News Hub, why we chose it, and how it is used in both **offline indexing** and **online semantic search**.

---

## Model choice

**Model**: `sentence-transformers/paraphrase-multilingual-mpnet-base-v2`

- **Type**: Sentence transformer (bi-encoder) based on multilingual MPNet.
- **Languages**: Trained on many languages and works well for **Nepali** because it is multilingual and supports Devanagari.
- **Embedding dimension**: **768** – this is why `news.article_embedding.embedding` is defined as `VECTOR(768)`.
- **Usage style**: Encode each text (query or article chunk) independently and compare using cosine similarity / L2 distance.
- **License**: Permissive for on-prem / local use (check upstream repo for details before production).

We intentionally avoid any hosted API dependency so that:
- All article content stays inside your own infrastructure.
- Latency and cost are predictable.
- You can later swap the model for a better Nepali-specific one without schema changes (still 768-d vectors).

---

## How the model is used

We use the same model in **two places**:

1. **Offline indexing** – embedding article chunks into `news.article_embedding`.
2. **Online search** – embedding user queries at request time.

Using one shared model ensures that:
- The vector space is consistent between articles and queries.
- Similar content in Nepali ends up close together geometrically.

### 1. Offline indexing (articles)

The indexing script will:

1. Load the model once at startup, for example:

   ```python
   from sentence_transformers import SentenceTransformer

   model_name = "sentence-transformers/paraphrase-multilingual-mpnet-base-v2"
   model = SentenceTransformer(model_name)
   ```

2. For each **chunk** of article text:

   ```python
   import numpy as np

   # chunks: List[str]
   embeddings = model.encode(
       chunks,
       batch_size=BATCH_SIZE,
       show_progress_bar=False,
       convert_to_numpy=True,
       normalize_embeddings=False,  # use raw vectors; similarity is controlled in SQL
   )
   ```

3. Convert `embeddings` to a suitable format for Postgres:
   - Directly pass as Python lists / arrays and let the DB driver map to `VECTOR(768)`.
   - Or convert to Python list via `embeddings.tolist()` and use parameterized `INSERT`.

4. Insert into `news.article_embedding` with all metadata.

### 2. Online search (queries)

We do **not** load the model directly in Go. Instead we run a small Python HTTP service (embedding sidecar) that:

- Loads the same model at startup.
- Exposes an endpoint like `POST /embed`:

  ```python
  from fastapi import FastAPI
  from pydantic import BaseModel
  from sentence_transformers import SentenceTransformer

  app = FastAPI()
  model = SentenceTransformer("sentence-transformers/paraphrase-multilingual-mpnet-base-v2")


  class EmbedRequest(BaseModel):
      texts: list[str]


  @app.post("/embed")
  def embed(req: EmbedRequest):
      vectors = model.encode(
          req.texts,
          batch_size=len(req.texts),
          convert_to_numpy=True,
          normalize_embeddings=False,
          show_progress_bar=False,
      )
      return {"vectors": vectors.tolist()}
  ```

The Go core service:
- Receives a user query string.
- Sends `[query]` to `/embed`.
- Receives a single 768-d vector.
- Uses that vector in the Postgres similarity query against `news.article_embedding.embedding`.

---

## Normalization & similarity

There are two common choices:

- **Option A: L2 distance (current design)**  
  - Store **raw embeddings** (no per-vector normalization).  
  - Use `embedding <-> :q` (L2 distance) in the `ORDER BY` clause with pgvector.  
  - Simple and works well in practice for this model.

- **Option B: Cosine similarity**  
  - Normalize vectors to unit length before storage and for queries.  
  - Use cosine distance operator in pgvector.  
  - Slightly more work, but very standard.

At the moment, the design assumes **Option A** for simplicity. If tuning later suggests cosine works better, you can:
- Add a migration that re-encodes and re-writes vectors as normalized.
- Switch the index operator class and SQL ordering.

---

## Performance considerations

- **Batch size**:
  - For CPU-only environments, start with `BATCH_SIZE = 16–32`.
  - Increase until CPU or RAM become limiting.
- **Throughput**:
  - Indexing is offline; a slightly slower batch is acceptable.
  - Query-time embeddings should be fast (a single short query usually encodes in a few ms on a modern CPU).
- **Caching**:
  - Optional: cache embeddings for frequently repeated queries if you notice patterns.

---

## Model replacement strategy

To upgrade the embedding model in the future (e.g., to a Nepali-specific model):

1. Choose a new model with the **same embedding dimension** (768) if you want to reuse the existing `VECTOR(768)` column.
2. Deploy the new model in both:
   - Indexing script.
   - Embedding HTTP service.
3. Rebuild all embeddings:
   - Truncate `news.article_embedding` (if acceptable) and re-run the indexing script.
   - Or version embeddings with an extra column (e.g., `model_version`) and migrate gradually.

Because the schema is already vector-friendly, swapping the model is a data task, not a schema redesign.

---

## Summary

- We use `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` as a strong multilingual model with Nepali support.
- Dimension 768 is reflected directly in the `news.article_embedding.embedding` column (`VECTOR(768)`).
- The same model powers both offline article indexing and online query embedding via a small Python HTTP service.
- The design keeps the door open to future model upgrades without major schema changes.

