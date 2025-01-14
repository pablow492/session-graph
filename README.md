# session-graph

**Turn your scattered AI coding sessions into a queryable knowledge graph.**

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![RDF](https://img.shields.io/badge/data-RDF%2FTurtle-orange.svg)](https://www.w3.org/RDF/)
[![SPARQL](https://img.shields.io/badge/query-SPARQL-green.svg)](https://www.w3.org/TR/sparql11-query/)
[![Apache Jena Fuseki](https://img.shields.io/badge/triplestore-Apache%20Jena%20Fuseki-red.svg)](https://jena.apache.org/documentation/fuseki2/)
[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

---

## The Problem

Developers use 5+ AI tools every day -- Claude Code, ChatGPT, Cursor, Copilot, Grok, DeepSeek, Warp. Each session is an isolated silo. Knowledge dies when the tab closes.

You have solved the same problem three times across different tools and cannot find any of them. You debugged a Supabase auth flow in Claude Code last Tuesday, discussed the same pattern in ChatGPT a month ago, and asked Grok about JWT refresh tokens somewhere in between. None of these tools talk to each other.

Existing solutions are single-platform and flat-file. They give you search over one tool's history, not structured relationships across all of them. A grep over session logs does not tell you that `FastAPI uses Pydantic` or that `Neo4j is a type of graph database`. It just gives you walls of text.

**session-graph** fixes this.

## The Solution

session-graph extracts structured knowledge triples -- `(subject, predicate, object)` -- from all your AI coding sessions, links entities to Wikidata for universal disambiguation, and loads everything into a SPARQL-queryable triplestore with full provenance back to the source conversation.

```
"What technologies have I used across all sessions?"  -->  SPARQL query  -->  structured answer
"How does FastAPI relate to Pydantic?"                 -->  FastAPI --uses--> Pydantic
"What sessions discussed authentication?"              -->  3 sessions across Claude Code + DeepSeek
```

The key insight: **a knowledge graph without relationships is just a tag cloud.** The minimum viable extraction unit is `(subject, predicate, object)`, not `[topic1, topic2, topic3]`.

### What makes this different

- **Multi-platform**: Ingests Claude Code, ChatGPT, DeepSeek, Grok, and Warp into a single unified graph. No other tool does this.
- **Formal ontology**: Composes 5 W3C/ISO standards (PROV-O, SIOC, SKOS, Dublin Core, Schema.org) instead of inventing a custom schema.
- **Wikidata linking**: Entities are disambiguated against 100M+ Wikidata items via `owl:sameAs`. "k8s", "kubernetes", and "K8s" all resolve to [Q22661306](https://www.wikidata.org/wiki/Q22661306).
- **Full provenance**: Every knowledge triple traces back to the exact source message, session, platform, and file path.
- **Federated queries**: SPARQL can query your local graph and Wikidata in a single query.

## Results

From real-world usage across 1,537 sessions:

| Metric | Value |
|--------|-------|
| Total triples in Fuseki | 2,027,178 |
| Sessions indexed | 1,537 |
| Knowledge triples extracted | 97,910 |
| Distinct entities | ~12,000+ |
| Wikidata-linked entities | ~5,000 (~33%) |
| Curated predicates | 24 (with <1% `relatedTo` fallback) |
| Platforms supported | 5 (Claude Code, ChatGPT, DeepSeek, Grok, Warp) |
| Entity linking precision | 7/7 (agentic ReAct linker) |
| Cost per 600 sessions | ~$0.60 (Vertex AI batch pricing) |

### Graph Preview

Real data from SPARQL — technologies, concepts, and session provenance linked across multiple Claude Code sessions:

![Knowledge Graph Preview](docs/graph-preview.png)

*Hub nodes (large blue) are highly connected technologies. Green nodes are concepts/outputs. Purple rectangles are session IDs with dashed provenance edges. The "W" badge indicates entities linked to Wikidata.*

## Architecture

```
Scattered Sources              Adapter Layer           Knowledge Graph
-----------------              -------------           ---------------
Claude Code (.jsonl)  --+
DeepSeek (.json zip)  --+     triple_extraction.py
Grok (.json zip)      --+--->  (LLM extracts s,p,o   ---> Apache Jena Fuseki
Warp (SQLite)         --+      from each assistant         (SPARQL endpoint)
ChatGPT (.json)       --+      message using 24                 |
Cursor (.jsonl)       --+      curated predicates)              |
                                     |                          v
                                     v                    SPARQL Queries
                            link_entities.py           (16 local templates
                             (LangGraph ReAct           + 6 Wikidata templates)
                              agent links to                    |
                              Wikidata QIDs)                    v
                                                        Claude Code Skill
                                                     (natural language -> SPARQL)

Real-time Loops:
  Claude Code session pause/end → stop_hook.sh → RabbitMQ → pipeline-runner → Fuseki
  pi session end/shutdown        → pi devkg-hook  → RabbitMQ → pipeline-runner → Fuseki
  Codex session file changes     → codex-publisher → RabbitMQ → pipeline-runner → Fuseki
  Cursor agent turn end (`stop`) → cursor_hook.sh → RabbitMQ → pipeline-runner → Fuseki
                                              (extract + Wikidata link + upload;
                                               triple/entity caches: 0 API calls for seen data)
```

### Pipeline in Detail

```
1. SOURCE PARSING (per platform --> RDF Turtle)
   Each parser reads a platform-specific format and produces
   PROV-O + SIOC session structure plus knowledge triples.

2. TRIPLE EXTRACTION (LLM-powered)
   Each assistant message --> LLM --> top 10 (subject, predicate, object) triples
   24 curated predicates | capped at 10 triples/message (prioritizes architecture)
   Closed-world vocabulary (deviations fuzzy-matched) | retry on JSON truncation

3. ENTITY FILTERING (two-level)
   Level 1: is_valid_entity() in triple_extraction.py -- rejects garbage at extraction
   Level 2: is_linkable_entity() in link_entities.py -- pre-filters before Wikidata
   Catches: filenames (*.py), hex colors (#8776f6), CLI flags (--force),
            ICD codes (j458), snake_case identifiers, DOM selectors, etc.
   48 whitelisted short terms bypass filters (ai, api, llm, rdf, sql, etc.)

4. ENTITY LINKING (context-aware, agentic)
   For each entity:
   +-- Normalize via entity_aliases.json (161 mappings: k8s-->kubernetes, etc.)
   +-- Frequency filter: --min-sessions 2 (default) -- only links entities
   |   appearing in 2+ sessions (~77% reduction)
   +-- Check SQLite cache
   +-- If miss --> LangGraph ReAct agent (LLM + Wikidata API tool)
   +-- Confidence threshold 0.7 --> owl:sameAs link
   +-- Entity dedup: same QID --> owl:sameAs between aliases

5. LOAD --> Apache Jena Fuseki (SPARQL endpoint)

6. QUERY --> SPARQL (via Claude Code skill or directly)
```

## Supported Platforms

| Platform | Parser | Format | Status |
|----------|--------|--------|--------|
| Claude Code | `jsonl_to_rdf.py` | JSONL | Production |
| DeepSeek | `deepseek_to_rdf.py` | JSON zip export | Production |
| Grok | `grok_to_rdf.py` | JSON (MongoDB export) | Production |
| Warp | `warp_to_rdf.py` | SQLite | Production |
| ChatGPT | `chatgpt_to_rdf.py` | JSON export | Production |
| Codex | `codex_to_rdf.py` | JSONL (`~/.codex/sessions`) | Production |
| Cursor | `cursor_to_rdf.py` | JSONL (`~/.cursor/projects/*/agent-transcripts`) | Production |
| VS Code Copilot | -- | JSON | Planned |

All parsers produce the same RDF schema. Entities merge by label across platforms.

## Quick Start

```bash
git clone https://github.com/robertoshimizu/session-graph.git
cd session-graph
./setup.sh
```

The setup script checks prerequisites, creates `.env` with your LLM provider, installs Python dependencies, starts Docker services (Fuseki + RabbitMQ), and runs a smoke test — all interactively.

After setup: **http://localhost:3030** (Fuseki SPARQL UI) and **http://localhost:15672** (RabbitMQ, devkg/devkg).

<details>
<summary>Manual setup (without setup.sh)</summary>

```bash
# 1. Configure
cp .env.example .env
# Edit .env with your LLM provider API key (see Provider Support below)

# 2. Install
python -m venv .venv && source .venv/bin/activate
# Choose one: requirements-gemini.txt, -openai.txt, -anthropic.txt,
# -fireworks.txt, or -ollama.txt
pip install -r requirements-gemini.txt

# 3. Create output directories
mkdir -p output/claude output/deepseek output/grok output/warp logs

# 4. Start all services (Fuseki + RabbitMQ + pipeline-runner)
docker compose up -d
# Fuseki SPARQL UI: http://localhost:3030
# RabbitMQ Management UI: http://localhost:15672 (devkg/devkg)

# 5. Process a single session (manual)
python -m pipeline.jsonl_to_rdf path/to/session.jsonl output/claude/session.ttl

# 6. Link entities to Wikidata
PYTHONUNBUFFERED=1 python -m pipeline.link_entities \
  --input output/*.ttl --output output/wikidata_links.ttl

# 7. Load into Fuseki (--auth required for Docker Fuseki)
python -m pipeline.load_fuseki output/*.ttl --auth admin:admin

# 8. Query at http://localhost:3030
```
</details>

### Automatic Processing (Recommended)

With Docker Compose running, new sessions are automatically processed:

```
Claude Code session ends
  → stop_hook.sh publishes to RabbitMQ (~33ms, non-blocking)

pi session ends
  → pi devkg extension publishes to RabbitMQ

Codex writes/updates a session file
  → codex-publisher container detects change and publishes to RabbitMQ

Cursor agent turn ends (local IDE `stop` hook)
  → cursor_hook.sh publishes to RabbitMQ (~33ms, non-blocking)

Then pipeline-runner consumes jobs:
  → Extracts triples (message UUID / line-id cache)
  → Links entities to Wikidata inline (entity cache first; agentic ReAct on misses, capped per job)
  → Writes .ttl and uploads to Fuseki
  → Failed jobs go to dead-letter queue for inspection
```

Set `DEVKG_SKIP_LINKING=1` on `pipeline-runner` to disable inline Wikidata linking (extract + Fuseki only). Batch `link_entities.py` remains available for catch-up / `--min-sessions` corpus passes.

Configure the Claude Code hook in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{"hooks": [{"type": "command", "command": "/path/to/hooks/stop_hook.sh", "timeout": 5}]}]
  }
}
```

Configure the Cursor hook in `~/.cursor/hooks.json`:

```json
{
  "version": 1,
  "hooks": {
    "stop": [
      {
        "command": "/path/to/hooks/cursor_hook.sh",
        "timeout": 10
      }
    ]
  }
}
```

Note: Cursor chats may also be queued accidentally via the Claude Code `Stop` hook when both run in the same environment. Prefer the native `cursor_hook.sh` path; the consumer routes `.cursor/projects` transcripts to `cursor_to_rdf.py` either way.

### Bulk Processing (Backfill Your History)

Once automatic processing is running, it only captures **new** sessions going forward. But you likely have weeks or months of past Claude Code sessions already sitting on disk — and that's where most of the value is.

Claude Code stores every session as a `.jsonl` file under `~/.claude/projects/`. Each project directory contains one file per session. A typical developer accumulates hundreds of sessions over a few months. Bulk processing lets you backfill all of them into the knowledge graph in one shot.

**This is optional but highly recommended.** The more sessions in the graph, the richer the connections — you'll find patterns and relationships you didn't know existed across your past work.

```bash
source .venv/bin/activate

# Option A: Batch (50% cheaper, parallel via Vertex AI — requires GCP setup)
python -m pipeline.bulk_batch submit --sort newest
python -m pipeline.bulk_batch status --wait --poll-interval 60
python -m pipeline.bulk_batch collect

# Option B: Sequential (simpler, works with any provider)
python -m pipeline.bulk_process --limit 50 --sort newest --skip-linking

# Then link entities to Wikidata (batch catch-up / --min-sessions filter)
# New RabbitMQ jobs already link inline; use this for history or corpus passes.
PYTHONUNBUFFERED=1 python -m pipeline.link_entities \
  --input output/claude/*.ttl --output output/claude/wikidata_links.ttl --workers 8

# Load into Fuseki (--auth required for Docker Fuseki)
python -m pipeline.load_fuseki output/claude/*.ttl output/claude/wikidata_links.ttl --auth admin:admin
```

After the backfill, automatic processing takes over — every future session is indexed and Wikidata-linked as you work, with no manual steps.

### Other Platforms (Cross-Platform Insights)

Most AI tools let you export your conversation history — DeepSeek and Grok offer JSON/zip downloads, Warp stores sessions in a local SQLite database. session-graph ingests all of them into the same knowledge graph, using the same ontology and entity vocabulary.

This is where it gets interesting: entities are linked **across platforms**. If you discussed "Kubernetes" in Claude Code, "k8s" in DeepSeek, and "container orchestration" in Grok, they all resolve to the same Wikidata entity and connect in the graph. You can query relationships that span tools you used months apart, on different projects, without remembering where you had each conversation.

```bash
# Export your chat history from each platform, then:
python -m pipeline.deepseek_to_rdf data/deepseek_export.zip output/deepseek/deepseek.ttl
python -m pipeline.grok_to_rdf data/grok_export.zip output/grok/grok.ttl
python -m pipeline.warp_to_rdf output/warp/warp.ttl --min-exchanges 5

# Link entities and load — same as Claude sessions
PYTHONUNBUFFERED=1 python -m pipeline.link_entities \
  --input output/**/*.ttl --output output/wikidata_links.ttl
python -m pipeline.load_fuseki output/**/*.ttl --auth admin:admin
```

### Querying from Claude Code

Once Fuseki has data, you don't need to write SPARQL by hand. session-graph ships with a **Claude Code skill** (`devkg-sparql`) that translates natural language questions into SPARQL queries, runs them against Fuseki, and returns formatted results.

The skill is automatically available when you work inside the session-graph repo. From any project, you can invoke it with `/devkg-sparql`:

```
You:   /devkg-sparql What technologies have I used the most?
Claude: [runs SPARQL hub detection query → returns top 20 entities by degree]

You:   /devkg-sparql How does FastAPI relate to Pydantic?
Claude: FastAPI --uses--> Pydantic (source: session abc123, Jan 15)

You:   /devkg-sparql What sessions discussed authentication?
Claude: [returns 3 sessions across Claude Code + DeepSeek with dates and source files]

You:   /devkg-sparql What do I know about Kubernetes?
Claude: [runs entity lookup, finds 12 relationships + Wikidata link to Q22661306]
```

The skill includes 16 local query templates (provenance-first topic+intent search, entity lookup, path discovery, hub detection, cross-session overlap, session insight packs, etc.) and 6 Wikidata traversal templates for enriching local entities with external knowledge. Prefer SPARQL over grep when Fuseki is up.

To use it from other projects, add the skill path to your Claude Code settings or symlink `.claude/skills/devkg-sparql/` into your project.

## Why RDF/SPARQL?

Most developer tools reach for Neo4j, vector databases, or JSON files. Here is why session-graph uses RDF and SPARQL instead.

### Formal ontology composition

session-graph does not invent a custom schema. It composes 5 battle-tested W3C/ISO standards:

| Standard | Role | Maturity |
|----------|------|----------|
| **PROV-O** | Provenance: who did what, when, derived from what | W3C Recommendation |
| **SIOC** | Conversation structure: messages, threads, containers | W3C Member Submission |
| **SKOS** | Taxonomy: topics, broader/narrower hierarchies | W3C Recommendation |
| **Dublin Core** | Metadata: dates, titles, creators | ISO 15836 |
| **Schema.org** | Cherry-pick: `SoftwareSourceCode` | De facto standard |

This same composition approach was validated by IBM's [GRAPH4CODE](https://arxiv.org/abs/2002.09440) project at 2 billion triples.

### Wikidata linking

Every entity in the graph can be linked to Wikidata via `owl:sameAs`. This gives you:

- **Universal disambiguation**: "k8s", "kubernetes", and "K8s" all resolve to the same Wikidata item.
- **Cross-language dedup**: "medication" and "medicamento" both map to [Q12140](https://www.wikidata.org/wiki/Q12140).
- **External enrichment**: Query Wikidata to discover that Neo4j is written in Java, or that fosfomycin is an antibiotic -- knowledge that does not exist in your local sessions.

### Lightweight triplestore

Apache Jena Fuseki runs as a single JAR file. No JVM tuning required. It handles 138K+ triples without breaking a sweat. Compare this to Neo4j (Docker + plugins + configuration) or a hosted vector database (monthly fees).

### Federated queries

SPARQL's `SERVICE` keyword lets you query your local graph and Wikidata in a single request:

```sparql
# Find what Wikidata knows about entities in your local graph
SELECT ?localLabel ?wikidataDescription WHERE {
  ?entity a devkg:Entity ;
          rdfs:label ?localLabel ;
          owl:sameAs ?wd .
  SERVICE <https://query.wikidata.org/sparql> {
    ?wd schema:description ?wikidataDescription .
    FILTER(LANG(?wikidataDescription) = "en")
  }
}
```

No other query language can do this.

### Provenance built-in

PROV-O gives you provenance for free. Every knowledge triple links back to:
- The exact **message** it was extracted from (with full text)
- The **session** it belongs to
- The **platform** (Claude Code, DeepSeek, Grok, Warp)
- The **source file** on disk

### No vendor lock-in

RDF is an ISO standard (W3C). Your data is portable. You can move it to any triplestore (Fuseki, Blazegraph, GraphDB, Stardog, Amazon Neptune) or convert it to Neo4j via n10s. Try doing that with a proprietary vector database.

## Provider Support

session-graph keeps `get_provider()` as its configuration adapter while returning native LangChain chat models. Every provider therefore supports the standard `invoke()`, `stream()`, `batch()`, and async interfaces plus automatic LangSmith tracing.

| Provider | Triple Extraction | Entity Linking | Batch Processing |
|----------|-------------------|----------------|------------------|
| Google Gemini (Vertex AI) | Yes | Yes | Yes (50% discount) |
| Google Gemini (AI Studio) | Yes | Yes | No |
| OpenAI | Yes | Yes | No |
| Anthropic (Claude) | Yes | Yes | No |
| Fireworks AI | Yes | Model must support tool calling | No |
| Ollama (local) | Yes | Model must support tool calling | No |

Configure your provider in `.env`:

```env
LLM_PROVIDER=gemini       # gemini, openai, anthropic, fireworks, or ollama
LLM_MODEL=gemini-2.5-flash

# Optional native LangSmith telemetry:
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=your-key
LANGSMITH_PROJECT=devkg
```

Fireworks requires an explicit model ID in `LLM_MODEL`. The benchmark-validated default is `accounts/fireworks/models/gpt-oss-120b` ($0.15/$0.60 per M tokens), which works for both triple extraction and Wikidata linking — approximately 2.5× cheaper than the previous Gemini 2.5 Flash + Gemini 3 Flash combination. Gemini uses Vertex AI automatically when `GOOGLE_CLOUD_PROJECT` or `ANTHROPIC_VERTEX_PROJECT_ID` is configured; otherwise it uses the Gemini API key.

LangSmith extraction and Wikidata-linking runs use stable names (`devkg.triple_extraction`, `devkg.wikidata_linking`) and filterable provenance metadata: `source_platform`, `session_id`, `message_id`, `source_file`, and `project`. Model identity uses LangSmith's canonical `ls_provider` and `ls_model_name` fields; platform tags use `platform:<name>`.

## Example SPARQL Queries

### What technologies have I used across all sessions?

```sparql
PREFIX devkg: <http://devkg.local/ontology#>
PREFIX rdfs:  <http://www.w3.o