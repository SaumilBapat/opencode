# Working Memory System v1

**Status:** In Progress
**Date:** 2026-02-13
**Author:** Saumil Bapat + Claude

## Problem

OpenCode (and all agents in the cluster) lack persistent, accurate memory of the user. The current Memory Gateway stores 72K raw events in SQLite with FTS5 keyword search. This fails because:

1. **Keyword search can't understand intent** — "which accounts do I support" doesn't match "moved to FINS, supports OnePay" because the words don't overlap
2. **Raw events are noisy** — 72K events include "hi", "whats my name", tool output dumps, near-duplicate support tickets, and LLM hallucinations (e.g., incorrect facts like "your name is John Smith")
3. **No temporal reasoning** — old facts ("supports Phreesia") and new facts ("moved to FINS") compete equally in search
4. **No deduplication** — the same fact appears hundreds of times across conversations in different phrasings

Embedding raw events would just give fast access to garbage. The vector space must contain only distilled, verified knowledge.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MEMORY GATEWAY (mini-1)                   │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │ SQLite        │    │ Postgres     │    │ Fact Store    │  │
│  │ (raw events)  │    │ (pgvector)   │    │ (active/      │  │
│  │ 72K events    │    │ embeddings   │    │  superseded)  │  │
│  │ FTS5 index    │    │ cosine sim   │    │              │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                    │                    │          │
│         │         ┌──────────┴────────────────────┘          │
│         │         │                                          │
│  ┌──────┴─────────┴──────────────────────────────────────┐  │
│  │              SEARCH ENDPOINT                           │  │
│  │  query → embed → cosine similarity on fact embeddings  │  │
│  │  + recency boost (last_confirmed timestamp)            │  │
│  │  + only active facts (not superseded)                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              PROFILE COMPILER (daemon)                  │  │
│  │  conversations → gpt-5-nano extraction → facts          │  │
│  │  → embed → soft dedup → LLM dedup → store              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

         ▲ search                          ▲ write
         │                                 │
┌────────┴──────┐  ┌──────────────┐  ┌────┴──────────┐
│ OpenCode      │  │ Claude Code  │  │ Codex Workers │
│ (plugin)      │  │ (hooks)      │  │ (run-codex.sh)│
└───────────────┘  └──────────────┘  └───────────────┘
```

## Data Model

### facts (pgvector — the clean knowledge base)

```sql
CREATE TABLE memory_facts (
    fact_id         TEXT PRIMARY KEY,       -- stable hash of (category, canonical_text)
    text            TEXT NOT NULL,          -- the fact, <=160 chars
    category        TEXT NOT NULL,          -- bio, location, health, work, relationships, projects, tooling, constraints, communication_style, other
    confidence      FLOAT DEFAULT 0.6,     -- 0.0-1.0, increases with support count
    support         INT DEFAULT 1,         -- number of conversations confirming this fact
    status          TEXT DEFAULT 'active',  -- active | superseded
    superseded_by   TEXT,                   -- fact_id of the newer fact (if superseded)
    first_seen      TIMESTAMPTZ NOT NULL,
    last_confirmed  TIMESTAMPTZ NOT NULL,
    embedding       vector(1536),          -- text-embedding-3-small
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_facts_embedding ON memory_facts
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_facts_status ON memory_facts(status);
CREATE INDEX idx_facts_category ON memory_facts(category);
```

### fact_sources (provenance — links facts back to conversations)

```sql
CREATE TABLE fact_sources (
    fact_id         TEXT NOT NULL REFERENCES memory_facts(fact_id),
    conversation_id TEXT NOT NULL,          -- session_id from memory_events
    extracted_at    TIMESTAMPTZ DEFAULT NOW(),
    evidence        TEXT,                   -- <=120 chars quote from the conversation
    PRIMARY KEY (fact_id, conversation_id)
);
```

### Relationship to existing tables

- `memory_events` (SQLite) — raw event log, unchanged. Source of truth for conversations.
- `memory_facts` (Postgres/pgvector) — distilled knowledge extracted from conversations.
- `fact_sources` (Postgres) — many-to-many link between facts and source conversations.

A single fact can be linked to many conversations (confirming evidence). A single conversation can produce many facts.

## Pipeline

### Stage 1: Extract

For each conversation (grouped by `session_id`):

1. Load the full thread — all events in the session, ordered by timestamp
2. Concatenate into a single conversation context (all prompts + responses)
3. Send to gpt-5-nano with seed facts + the conversation
4. LLM returns structured JSON: `{facts: [...], preferences: [...]}`
5. Each item has: `text` (<=160 chars), `category`, `confidence` (0-1), `evidence` (<=120 char quote)

**Key design choice:** Full conversation thread is passed (not individual messages) to prevent misclassification of first-person drafting as personal facts. When the user drafts an email as someone else, the thread context makes it clear this isn't a personal fact.

**Model:** gpt-5-nano only. No other models.

**System prompt:**
```
You extract durable information about the user from their conversations.
Return ONLY JSON. A very short message can still be important (e.g., "I have ADHD").
Only extract items that help understand the user long-term:
(1) stable facts about the user (identity, background, health, family, job/role),
(2) long-term preferences/instructions for how the assistant should behave.
Do NOT extract: general trivia, one-off tasks, temporary plans, LLM hallucinations.
Only include a fact if it is explicitly stated by the user or clearly evidenced.
Output schema: {facts: [...], preferences: [...]} where each item is an OBJECT with:
text (<=160 chars), category (string), confidence (0..1),
evidence (<=120 chars quote/summary of the user's wording).
```

### Stage 2: Embed + Soft Dedup on Insert

For each extracted fact:

1. Generate embedding via `text-embedding-3-small`
2. Cosine search existing facts in pgvector
3. If similarity > 0.95 — treat as confirmation:
   - Increment `support` count
   - Update `last_confirmed` timestamp
   - Add conversation to `fact_sources`
   - Keep the more detailed text version
4. If similarity 0.85-0.95 — insert as new fact, flag for LLM dedup review
5. If similarity < 0.85 — insert as new fact

### Stage 3: LLM Dedup Pass (Automated)

Periodically (after backfill, or on schedule), run a dedup sweep:

1. Batch all facts (or all facts flagged for review)
2. Send groups of similar facts to gpt-5-nano
3. LLM classifies each pair/group:
   - **Identical meaning** → merge (keep most complete version, combine source links)
   - **Contradiction with timestamps** → supersede (mark older as `superseded_by: <newer_fact_id>`, newer becomes active)
   - **Subset** ("Works at Twilio" vs "Solutions Engineer at Twilio") → merge into the more detailed version
   - **Related but distinct** → keep both
4. Execute merge/supersede operations automatically. No human review.

**Rules for automated decisions:**
- Identical meaning: merge, keep the most complete text, union all source conversations
- Contradiction: newer `last_confirmed` wins, older gets `status=superseded`
- Subset: merge into the superset, union sources
- Related but distinct: keep both, no action

### Stage 4: Search (Runtime)

When an agent queries memory:

1. Embed the query via `text-embedding-3-small`
2. Cosine similarity search in pgvector (`WHERE status = 'active'`)
3. Score = `cosine_similarity * 0.7 + recency_boost * 0.3`
   - `recency_boost = 1.0 / (1.0 + age_hours / 720.0)` (30-day half-life)
4. Return top-k results with timestamps, sorted chronologically
5. Inject into system prompt with framing: "sorted oldest→newest, most recent supersedes earlier"

## Backfill Plan

### Input
- 72,508 events across 3,041 conversations in the Memory Gateway SQLite DB
- Source: `memory_events WHERE project_id = 'personal-memory'`

### Conversation size distribution

| Bucket | Conversations | Events | Avg Size |
|--------|--------------|--------|----------|
| 1-5 msgs | 968 | 2,982 | 7 KB |
| 6-20 msgs | 1,151 | 12,763 | 56 KB |
| 21-50 msgs | 551 | 17,726 | 216 KB |
| 51-100 msgs | 231 | 15,997 | 505 KB |
| 100+ msgs | 140 | 23,040 | 1.1 MB |

### Processing
- Each conversation passed to gpt-5-nano as a single request (no chunking, full thread)
- Seed facts loaded once, passed with each request
- Conversations processed chronologically (oldest first) so facts accumulate naturally
- Progress tracked: `last_processed_session_id` cursor in a state table

### Cost estimate
- ~150M input tokens + ~1.5M output tokens + ~2-5M dedup tokens
- gpt-5-nano: $0.10/1M input, $0.40/1M output
- **Total: ~$17-20 one-time**
- Embedding calls: ~$0.01 (negligible)

### Runtime (post-backfill)
- New conversations embed incrementally on each upsert (or batched every N minutes)
- Dedup pass runs daily or weekly
- Cost: negligible (~$0.01/day for typical usage)

## Existing Artifacts

The extraction pipeline already exists in `cluster-brain`:

| File | Purpose |
|------|---------|
| `bin/extract-openai-user-facts.py` | Extract facts from OpenAI export format. Includes embedding prefilter, chunked LLM extraction, stable ID dedup, incremental JSONL output. |
| `bin/consolidate-user-facts-jsonl.py` | Merge multiple extraction JSONL runs into deduplicated set. |
| `bin/validate-openai-user-facts.py` | LLM validation pass — classifies each fact as keep/discard based on durability and "about the user" criteria. |

These scripts target OpenAI export format. For v1, they need to be adapted to read from the Memory Gateway SQLite DB (conversations grouped by `session_id`) instead of the OpenAI JSON export.

## Integration Points

### OpenCode Plugin (`unified-memory-bridge.js`)
- Currently: searches FTS5 via gateway `/v1/memory/history/search`
- After: searches pgvector facts via new endpoint `/v1/memory/facts/search`
- Identity prefetch changes from hardcoded query to "return all active facts in category bio/work/location"
- Per-turn search uses embedded query against fact store

### Claude Code (hooks / claude-mem)
- claude-mem continues capturing observations independently
- New: conversations written to gateway also get fact-extracted on a delay

### Memory Gateway API
New endpoints:
- `POST /v1/memory/facts/search` — vector similarity search on fact store
- `GET /v1/memory/facts` — list all active facts (with optional category filter)
- `POST /v1/memory/facts/upsert` — manual fact insertion (for corrections)
- `POST /v1/memory/compiler/run` — trigger extraction for unprocessed conversations
- `GET /v1/memory/compiler/status` — check backfill progress

### Superseded facts
- Not deleted — retained with `status=superseded` and `superseded_by` pointer
- Queryable for timeline view ("show me how my accounts changed over time")
- Default search excludes them; optional `include_superseded=true` parameter

## What This Does NOT Cover

- **Real-time extraction** — v1 runs extraction as a batch/daemon, not inline on every message
- **Multi-user** — assumes single user (Saumil). No user_id scoping on facts.
- **Conflict resolution UI** — all dedup decisions are automated via LLM. No human review step.
- **Cross-agent fact sync** — facts live in Postgres on mini-1. Agents query via gateway API. No local caching of facts on workers.

## Implementation Order

1. Create Postgres tables (`memory_facts`, `fact_sources`) on mini-1
2. Adapt `extract-openai-user-facts.py` to read from gateway SQLite grouped by session_id
3. Run backfill: extract → embed → soft dedup → store
4. Run LLM dedup pass on the resulting fact set
5. Add `/v1/memory/facts/search` endpoint to gateway (pgvector cosine similarity)
6. Update `unified-memory-bridge.js` to query fact endpoint instead of FTS5
7. Add incremental extraction for new conversations (daemon or cron on mini-1)
