# NP News Hub — Documentation

**NP News Hub** is a **news intelligence system** for Nepali (and similar) news. It helps readers, journalists, and analysts **find**, **understand**, and **trust** news by searching over real articles, answering questions from your own database, and (over time) building a **knowledge graph** that tracks who said what, where a story came from, and whether past claims held up.

This folder explains what the project does, what problems it solves, who it’s for, and how the system is designed.

---

## What is NP News Hub?

NP News Hub is:

- **A searchable news archive** — Articles from selected sources are crawled, stored, and indexed by **meaning** (semantic embeddings), not just keywords.
- **Question‑answering over your data** — You ask in Nepali or English; the system returns answers and sources from the **indexed articles only**, so responses are grounded in what was actually published.
- **A path to a cognitive news graph** — The design includes entities (people, parties, places), provenance (who published first), narrative and bias signals, and a verification loop so the system can improve over time.

So it’s both a **working tool today** (semantic search + Q&A) and a **platform** for deeper intelligence (transparency, bias analysis, fact-checking support) as more pieces are built.

---

## What problems does it solve?

### 1. “I can’t find relevant articles when I search in Nepali.”

**Problem:** Keyword search fails when the exact words don’t match, or when you think in concepts (e.g. “election preparation in Parsa”) rather than specific phrases.

**What NP News Hub does:** **Semantic search** turns your question into a vector and finds the most **meaningfully similar** chunks in the database. So you get relevant excerpts even if the wording is different. You can ask in Nepali or English and get ranked results with titles, URLs, and snippets.

**Example:** Query *“पर्सामा निर्वाचनको तयारी कति अगाडि बढ्यो?”* returns articles about election preparedness in Parsa, with excerpts and source links.

---

### 2. “I want to ask a question and get an answer from the news, not from the whole internet.”

**Problem:** General chatbots can hallucinate or mix in unrelated information. You want answers that are **only** from your trusted, indexed news.

**What NP News Hub does:** The **Q&A flow** (Web → Search → Core) runs your question against the **same semantic index**. The answer is either a **natural-language summary** (when an LLM is configured) or the **relevant excerpts** themselves, plus **source links** to the original articles. So you always see which stories the answer came from.

**Example:** You ask *“What’s going on in Nepal today?”* and get a short answer plus a list of source headlines and URLs (e.g. election prep, protests, policy announcements) — all from the database.

---

### 3. “I don’t know if this story is original or copied from somewhere.”

**Problem:** The same narrative appears on many sites; it’s unclear who reported it first and who simply republished.

**What NP News Hub is designed for:** The **provenance and lineage** pillar (in the knowledge-graph design) uses **content hashing** and **discovery time** to identify the **first source** (“Patient Zero”) and to group duplicate or near-duplicate articles. So you can see “first seen at outlet X at time Y” and “also carried by these outlets.” *(This is in the design; full implementation is in progress.)*

---

### 4. “I can’t tell how different outlets are covering the same topic.”

**Problem:** It’s hard to compare framing, emphasis, or omission across Kantipur, Nagarik, Onlinekhabar, etc.

**What NP News Hub is designed for:** **Story clustering** groups articles about the same event; **narrative and bias** analysis (tone, intent, omission) can show how each outlet framed the story. Use cases include “How Kantipur vs Nagarik vs Onlinekhabar are covering topic X” and simple bias indicators. *(Design and use cases are documented; implementation is planned.)*

---

### 5. “I want to know if a politician’s or outlet’s past claims came true.”

**Problem:** Promises and predictions are rarely tracked systematically; trust in sources is hard to measure.

**What NP News Hub is designed for:** The **verification loop** registers **predictions** (e.g. “Budget will be passed by March”) with a **target date**, then re-evaluates them later and can update **publisher trust scores** based on outcomes. So the system can support “this outlet’s past claims: 80% came true.” *(Described in the knowledge-graph docs; implementation is planned.)*

---

## Who is it for?

| Audience | What they get |
|----------|----------------|
| **Readers** | Ask questions in Nepali or English and get answers with source links; discover relevant articles by meaning. |
| **Journalists & editors** | Semantic search, topic overviews, “more like this,” trend and timeline views; (future) story mapping, coverage gaps, duplicate detection. |
| **Analysts & researchers** | (Future) political intelligence dashboards, entity–relationship graphs, election monitoring, media bias comparison. |
| **Developers** | Clear docs on architecture, workflows, schema, and use cases so they can extend or integrate the system. |

---

## Use cases (summary)

The system supports (or is designed to support) these kinds of use cases:

- **Core discovery**  
  Semantic article search (question → relevant stories), “more like this” related articles, topic overviews and mini dossiers, trend and timeline exploration, category and region insights.

- **Analysis & intelligence**  
  Political intelligence (issues, parties, regions, coverage volume), media bias comparison (how different outlets cover the same topic), entity relationship graphs (“who appears with whom?”), election monitoring (candidates, parties, constituencies, incidents), trend forecasting (emerging topics).

- **Trust & transparency**  
  Origin of a story (“first seen at X”), which outlets carried the same or similar content, (future) fact-check and verification support, and (planned) verification of past claims.

- **Personalization & editorial tools**  
  (Future) Alerting on sensitive topics, personalized reading lists, internal editorial dashboards (story mapping, coverage gaps, duplicate detection).

Detailed use cases and flows are in [use-cases/README.md](use-cases/README.md).

---

## How does it work at a high level?

1. **Crawler** discovers and enqueues article URLs; **Core** stores list and article content.
2. **Engine** chunks article text, gets **embeddings** from an embedding service (e.g. MPNet), and writes them into the database.
3. **Core** exposes **semantic search**: a question is embedded and matched against stored vectors; results are returned with metadata (title, URL, snippet).
4. **Search** service sits in front of Core: it handles the **/ask** endpoint used by the Web UI, calls Core for search, and optionally uses an LLM to turn excerpts into a short answer. Responses are always tied to **sources** from the database.
5. **Web** is a simple chat interface: you type a question and get an answer plus source links.

So: **Crawler → Core (DB + embeddings) → Engine (embedding pipeline) → Search (Q&A) → Web.**  
For a step-by-step “question to answer” flow, see [workflow/ask-workflow.md](workflow/ask-workflow.md). For ports and run order, see [ports/list.md](ports/list.md).

---

## Where to go next

| If you want to… | Read |
|-----------------|------|
| Understand **what problems we solve** and **who it’s for** | You’re here. |
| See **detailed use cases** (search, trends, political intelligence, bias, etc.) | [use-cases/README.md](use-cases/README.md) |
| See **how a question becomes an answer** (step by step) | [workflow/ask-workflow.md](workflow/ask-workflow.md) |
| Run the system locally (ports, run order) | [ports/list.md](ports/list.md) |
| Understand the **knowledge graph** (entities, provenance, statements, verification) | [knowledge-graph/README.md](knowledge-graph/README.md) |
| Dive into **entity categories** (people, orgs, locations, events, topics) | [knowledge-graph/entity/categories.md](knowledge-graph/entity/categories.md) |
| See **graph relationships** (nodes, edges, tables) | [knowledge-graph/graph/README.md](knowledge-graph/graph/README.md) |

---

*This documentation is intended for the public: readers, journalists, analysts, and developers who want to understand what NP News Hub is and what it can do.*
