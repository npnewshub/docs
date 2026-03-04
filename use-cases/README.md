## NP News Hub – Use Cases

This document explores how we can use the existing NP News Hub stack (crawler, core API, database, and embeddings) to deliver real features. It is **design-only** for now – no implementation.

We group use cases into four layers:

- **Core discovery** – how readers find and navigate news.
- **Analysis & intelligence** – deeper understanding of politics, media, and topics.
- **Trust & transparency** – helping users judge reliability and origin.
- **Personalization & editorial tools** – tailoring and supporting newsroom workflows.

---

## A. Core discovery

### 1. Semantic article search (question → relevant stories)

**Goal**: A user asks a question in Nepali or English and receives the most relevant articles/snippets.

- **Inputs**
  - Free-text query (e.g. “पर्सामा निर्वाचनको तयारी कति अगाडि बढ्यो?”).
  - Optional filters: category (राजनीति, अर्थ, खेल, …), date range, source.
- **Flow**
  - Core API `POST /v1/search/query` embeds the query via `LLMs/mpnet` `/embed`.
  - Search runs against `news.article_embedding` (chunk-level vectors).
  - Top‑K matches are joined back to `news.list` and returned with title, URL, category, and snippet.
- **Outputs**
  - Ranked list of stories with small excerpts.
- **Notes**
  - This is the primary consumer of the embedding pipeline you just designed.

---

### 2. “More like this” – related articles

**Goal**: From a single article page, show 3–10 highly related stories.

- **Inputs**
  - `list_id` or article URL.
- **Flow**
  - Fetch all chunks for the article from `news.article_embedding`.
  - Compute a representative vector (e.g., mean of all chunks or just middle chunks).
  - Search similar vectors in `news.article_embedding` while excluding the same `article_id`.
  - Aggregate by `article_id` and pick the top related stories.
- **Outputs**
  - List of related article metadata (title, URL, category, time).
- **Notes**
  - Can be exposed as `GET /v1/search/related?list_id=...`.

---

### 3. Topic overview / mini dossiers

**Goal**: Build a page that summarizes everything about a topic (e.g., “Madhesh election”, “विदेश नीति”) using existing articles.

- **Inputs**
  - A topic query string or a predefined topic label.
- **Flow**
  - Use the same semantic search API to retrieve top N relevant articles/snippets.
  - Optionally group results by:
    - Time buckets (e.g., last 7 days, last 30 days, older).
    - Category.
  - (Later) feed snippets into a separate summarization LLM to generate a human-readable overview.
- **Outputs**
  - Structured “topic page”: key articles, timeline, and (later) an auto-generated summary.
- **Notes**
  - Works well even before full LLM summaries; initially can be “curated list of most relevant articles”.

---

### 4. Trend and timeline exploration

**Goal**: See how coverage about a topic evolves over time (e.g., number of stories per week about a specific issue).

- **Inputs**
  - Topic query.
  - Time range (date_from, date_to).
- **Flow**
  - Run semantic search for the topic.
  - Bucket matched `news.list` items by date (using `published_date` or `crawled_at`).
  - Return aggregated counts per day/week/month with example headlines.
- **Outputs**
  - Time-series data (for charts) + representative headlines per bucket.
- **Notes**
  - Can be used to power dashboards or editorial tools.

---

### 5. Category and region insights

**Goal**: Answer questions like “which topics dominate राजनीति this week?” or “what is being written about Madhesh प्रदेश?”.

- **Inputs**
  - Category = राजनीति / अर्थ / खेल / … (from `news.list.category`).
  - Optional region keywords in the query (e.g., “मधेश”, “काठमाडौँ”, “जनकपुर”).
- **Flow**
  - Filter `news.article_embedding` by `category`.
  - Apply semantic search with the regional query.
  - Aggregate by `source`, `published_date`, or subtopics inferred from frequent terms.
- **Outputs**
  - Ranked list of regional stories within the category, plus simple stats (counts per day, per source).

---

## B. Personalization & proactive alerts

### 6. Alerting and monitoring (future)

**Goal**: Automatically detect when a new article is “close to” sensitive topics (elections, disasters, major corruption cases) and send alerts.

- **Inputs**
  - A set of “watch queries” defined by the editor (e.g., “महाभियोग”, “दुर्घटना”, key politician names).
- **Flow**
  - As new articles are crawled and embedded:
    - For each watch query, compute its embedding once.
    - Compare new article embeddings to watch-query embeddings.
    - If similarity exceeds a threshold, log an event or send a notification.
- **Outputs**
  - Alert events stored in a table (later surfaced as notifications or dashboards).
- **Notes**
  - Uses the same vector space; no new model required.

---

### 7. Personalized reading lists (future)

**Goal**: Recommend stories to a user based on what they have previously read or liked.

- **Inputs**
  - User’s reading history: list of `list_id`s or URLs.
- **Flow**
  - For each user, build a “user profile vector” by averaging embeddings of articles they read.
  - Periodically run a similarity search between this profile vector and `news.article_embedding` to find new, unseen content.
  - Filter out already-read articles or those outside desired time windows.
- **Outputs**
  - A personalized “For you” feed of recommended stories.
- **Notes**
  - Requires a user-layer and tracking, but reuses the same embeddings and search pipeline.

---

## C. Analysis & intelligence

### 8. Internal editorial tools

Even if there is no public UI yet, the system can help editors:

- **Story mapping**: See which articles are clustered together in vector space (same event from different angles).
- **Coverage gaps**: Given a topic query, see days with minimal or no coverage.
- **Duplicate detection**: Spot articles whose content is nearly identical (syndicated or repeated stories).

Implementation ideas:
- A simple web dashboard that calls the core search endpoints and visualizes:
  - Lists of stories.
  - Simple charts (counts by date, by source, by category).

---

### 9. Political intelligence engine

**Goal**: Give analysts and journalists a live map of the political landscape – which issues, parties, leaders and regions are getting attention, and how that changes over time.

- **Inputs**
  - Topic queries (e.g., “संघीयता”, “प्रदेश २ चुनाव”, party names, key leaders).
  - Category filters (राजनीति) and date ranges.
- **Flow**
  - Use embeddings to group articles into issue-centric clusters for each query.
  - Aggregate by source, region keywords in content, and time buckets.
  - Compute simple indicators: volume of coverage, sentiment proxy (later), co-mentioned entities.
- **Outputs**
  - Dashboards like:
    - “Top political issues this week”.
    - “Coverage by party vs outlet”.
    - “Regional focus vs central politics”.

---

### 10. Media bias analyzer

**Goal**: Help users and editors see how different outlets cover the same topic differently (angles, emphasis, omissions).

- **Inputs**
  - Topic query (e.g., “मितव्ययिता”, “समितिहरूको रिपोर्ट”).
  - List of sources (from `news.list.source`).
- **Flow**
  - For a topic, retrieve a balanced set of articles/snippets per source using semantic search.
  - Compare:
    - Which subtopics appear per source (e.g., corruption vs development).
    - How much coverage each source gives to the same entities.
  - (Later) run a sentiment/stance classifier to quantify tone by outlet.
- **Outputs**
  - Side‑by‑side views:
    - “How Kantipur vs Nagarik vs OnlineKhabar are covering topic X”.
  - Simple bias indicators (e.g., which sources are mostly neutral vs consistently positive/negative on an entity).

---

### 11. Trend forecasting engine

**Goal**: Identify topics that are starting to gain momentum before they are obvious in headlines.

- **Inputs**
  - Continuous feed of new article embeddings.
  - Topic seeds or unsupervised clusters in the embedding space.
- **Flow**
  - Maintain rolling windows (e.g., last 7 / 30 days) of topic clusters.
  - Track:
    - Growth rate of mentions per cluster.
    - New clusters emerging around previously rare combinations of entities/keywords.
  - Flag clusters whose frequency or diversity of sources crosses a threshold.
- **Outputs**
  - “Emerging stories” panel for editors.
  - Early warning lists: issues that are likely to become major in the coming days.

---

### 12. Entity relationship graph

**Goal**: Visualize and explore how people, parties, organizations, and places are connected across news coverage.

- **Inputs**
  - Named entities extracted from article content (NER step on top of `news.article.content`).
  - Article-level and chunk-level embeddings to disambiguate similar names.
- **Flow**
  - For each article/snippet, detect entities (e.g., person, party, ministry, location).
  - Build a graph:
    - Nodes = entities.
    - Edges = co-occurrence in the same article or tight semantic neighborhood.
    - Edge weights = frequency / recency.
  - Support filters by time, category, region, or source.
- **Outputs**
  - Interactive graph for:
    - “Who appears with whom?”.
    - “What organizations are always mentioned with this politician?”.
  - Useful for investigative work and political analysis.

---

### 13. Election monitoring system

**Goal**: Track election-related coverage in near real time – candidates, parties, issues, incidents – with regional breakdowns.

- **Inputs**
  - List of election entities: candidates, parties, constituencies, commissions.
  - Watch queries for election processes: campaigning, मतदान, मतगणना, परिणाम, etc.
- **Flow**
  - As new articles arrive:
    - Run semantic search against election watch queries.
    - Run NER and map mentions to known candidates/parties/regions.
    - Store events in a dedicated election monitoring table (who, where, what, when, source, link).
  - Build views per constituency, party, candidate.
- **Outputs**
  - Dashboards:
    - “Top issues by constituency”.
    - “Media coverage per party / candidate”.
    - Timeline of key events (debates, incidents, announcements).

---

## D. Trust, transparency & misinformation support

### 14. Transparency, origin & misinformation support

**Goal**: Help ordinary readers and journalists understand where a story comes from, how it spread, and whether it might be misleading or taken out of context.

- **Inputs**
  - A single article URL or `list_id` a user is unsure about.
- **Flow**
  - Use embeddings to:
    - Find earliest articles with similar content (approximate “origin” within your corpus).
    - Find how many sources and which outlets are carrying similar narratives.
  - Show:
    - First seen time in your database (crawled_at, earliest published_date).
    - Number and diversity of sources reporting the same or similar story.
    - Related context articles (background pieces, official statements).
  - (Future) Integrate external fact-check feeds and highlight if similar claims were debunked elsewhere.
- **Outputs**
  - A “transparency panel” for each article:
    - “First seen on X at time Y.”
    - “Also reported by these outlets.”
    - “Context: related background articles.”
    - (Later) “Fact-check status” from trusted partners.

---

## Next steps

For now this document is **conceptual**. Concrete implementation steps could be:

1. Finalize the search API design (`/v1/search/query` and `/v1/search/related`).
2. Implement the embedding indexer script and keep `news.article_embedding` up to date.
3. Build a minimal internal “search console” UI for editors using the existing core + `LLMs/mpnet` service.

