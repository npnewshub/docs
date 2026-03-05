# Workflow: From question to answer

Step-by-step flow when you type a question in the chat and hit **Send**.

---

## 1. Web (browser → Search service)

| Step | What happens |
|------|-------------------------------|
| 1.1 | You submit the form. The app adds your message to the list and shows "Thinking...". |
| 1.2 | Frontend sends **POST** to the Search service: `POST {SEARCH_API}/ask` (default `http://127.0.0.1:8901/ask`). |
| 1.3 | Body: `{ "query": "<your question>", "limit": 8 }`. |

---

## 2. Search service (port 8901)

| Step | What happens |
|------|-------------------------------|
| 2.1 | Search service receives **POST /ask**, parses `query` and `limit` (default 10, max 20). |
| 2.2 | It calls the **Core** API: **POST** `{CORE_API_URL}/v1/news/search/query` (default `http://127.0.0.1:8701/v1/news/search/query`). |
| 2.3 | Body: `{ "query": "<same question>", "limit": <limit> }`. |
| 2.4 | Waits for Core’s JSON response (list of search results). |

---

## 3. Core (port 8701)

| Step | What happens |
|------|-------------------------------|
| 3.1 | Core receives **POST /v1/news/search/query**, validates `query` and `limit`. |
| 3.2 | **Embed query:** Core sends **POST** to the **MPNet** service: `{MPNET_BASE_URL}/embed` (default `http://127.0.0.1:8801/embed`). |
| 3.3 | Body: `{ "texts": ["<your question>"] }`. MPNet returns a **768-dimensional vector** for the question. |
| 3.4 | **Similarity search:** Core uses the **Search** module to query the database. |
| 3.5 | SQL runs against `news.article_embedding` (and `news.list` for metadata): nearest-neighbour search by **L2 distance** between the query vector and stored `embedding` vectors, ordered by distance, limited to `limit` rows. |
| 3.6 | Core returns JSON: `{ "success": true, "results": [ { "id", "chunk_text", "title", "url", "published_date", "score", ... }, ... ] }`. |

---

## 4. Search service (again)

| Step | What happens |
|------|-------------------------------|
| 4.1 | Search service receives Core’s results. |
| 4.2 | Builds **sources**: one entry per result (title, url, chunk_text, published_date, score). |
| 4.3 | Builds **answer**: concatenation of the excerpt texts only (no LLM). Format: `[1] <chunk_text>\n\n[2] <chunk_text>...`. If there are no results, answer is: *"No matching excerpts found in the news database."* |
| 4.4 | Returns to the browser: `{ "success": true, "answer": "<excerpts>", "sources": [ ... ] }`. |

---

## 5. Web (browser)

| Step | What happens |
|------|-------------------------------|
| 5.1 | Frontend receives the JSON response. |
| 5.2 | Appends an assistant message with `content = data.answer` and `sources = data.sources`. |
| 5.3 | Renders the answer text and, below it, source links (title/URL) from `sources`. |
| 5.4 | Loading state is cleared. |

---

## Flow diagram (high level)

```
[You]  →  Web (3000)  →  Search (8901)  →  Core (8701)  →  MPNet (8801)  [embed query]
                              ↓                    ↓
                        [build answer       DB: news.article_embedding
                         from results]       (vector similarity)
                              ↓
[You]  ←  Web (3000)  ←  Search (8901)  ←  Core (8701)
         (answer + sources)
```

---

## Dependencies

- **Web** needs **Search** (8901) to be up.
- **Search** needs **Core** (8701) to be up.
- **Core** search needs **MPNet** (8801) and the **database** (with `news.article_embedding` and `news.list` populated).

See [Ports list](../ports/list.md) for ports and run order.
