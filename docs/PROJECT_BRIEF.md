# archaeo — Project Brief

> Status: **planning** (2026-09-28). No code yet. This is a temporary v0.1 scope to see real results; we will revisit direction after it runs.
>
> Update (2026-09-29): embeddings and pgvector are **decisions to evaluate**, not committed architecture. The proposals and research below stay as the current provisional direction until an ADR resolves them. The milestone sequence now starts with the opening setup, followed by the evidence contract; see "Milestones".

## TL;DR

`archaeo` is a self-hosted, open-source tool (CLI + MCP server) that turns a **stack trace** into **resolved GitHub issues and fix PRs** — from your dependencies and from your own repo's history — using RAG over a local pgvector database.

Primary use: your AI coding agent hits an error → calls `archaeo` via MCP → gets the matching resolved issue, fix version, and workaround with citations, instead of guessing.

Secondary (v0.2): `archaeo why file:line` — explain *why* a line of code exists from PRs, reviews, and issues.

## Why we are doing this

Two goals, in order:

1. **Proof of work** for a Sr. Backend AI Engineer role (RAG, embeddings, vector DB, LLM integration). The project must demonstrate *production* thinking, not tutorial-level RAG: measured retrieval quality, error handling, observability, explicit trade-offs.
2. **Daily use.** A tool the author (and other developers) actually uses, so it gets dogfooded and improved.

A senior screen does not test "can you call an embeddings API". It tests: *How do you know retrieval works? What happens on failure? Why this design? What does it cost?* The project is designed to answer those questions with numbers.

## How we got here (decision log)

| # | Idea considered | Decision | Reason |
|---|---|---|---|
| 1 | RAG CLI over a local docs folder | Rejected as-is | Most common RAG tutorial. Too shallow without evals, trade-offs, failure handling. |
| 2 | Obsidian plugin | Rejected | Crowded (Smart Connections, Copilot, RAG Chat, Analogy…). Plugin runtime can't host pgvector, hides backend skills, weeks of review, weak trust model (Obsidian can't sandbox plugins). |
| 3 | Standalone service indexing the Obsidian vault | Rejected | Author already has this in their workflow (Obsidian MCP + ai-memory). |
| 4 | Researched real developer pain points | → found three candidates | See "Problem research". |
| 5 | #1 "why is this code like this?" + #2 "stack trace → resolved issues" | **Chosen**, #2 primary | #2 is useful more often; #1 only shines in old team repos. |
| 6 | One program or two? | **One program** | Same core (GitHub ingest, chunking, hybrid search, pgvector, evals). #2 already needs #1's machinery (linked fix PRs). |
| 7 | Form: skill / MCP / CLI | **CLI + MCP**, optional tiny skill | Skill alone can't hold an index (it'd be agentic `gh` search, not RAG). CLI = demo/evals/CI. MCP = daily use by agents. |
| 8 | Vector DB | **Provisional, to evaluate:** pgvector via docker-compose | Metadata + vectors in one transactional store, SQL filters, hybrid search in one query, runs on AWS RDS/Aurora. Docker locally is fine. |
| 9 | Ship a prebuilt DB? | **No** — ship empty, index on demand | A shipped DB is stale and can't contain private repos. Optional `pg_dump` snapshot only for demo/evals. |
| 10 | Does it need an LLM? | **Not in MCP mode** | The calling agent *is* the LLM; archaeo only retrieves and returns cited evidence. LLM optional for CLI `--answer` and for eval judging. |
| 11 | Embeddings | **Provisional, to evaluate:** local, in-process (e.g. `bge-small` via transformers.js) by default; cloud optional | No API key, no Ollama, data never leaves the machine, free. Embeddings can't be delegated to the agent: vectors must come from the same model every time. |
| 12 | Auto-generate docs under `/docs`? | **Read `/docs` now, write later (v0.2)** | Indexing existing docs is cheap. Generating docs = different product, hard to eval, hallucination risk, doc rot. v0.2: `trace --save` writes a reviewed `docs/errors/<slug>.md` after a confirmed fix. |

## Problem research (summary)

Recurring pains found (mostly blogs/HN/articles quoting Reddit; Reddit and X were not directly readable):

- **Lost "why"**: teams forget *why* code exists; `git blame` gives a cryptic commit; reasoning lived in Slack threads that vanished.
- **Repeated debugging**: runbooks/postmortems get buried; the same errors are debugged from scratch.
- **Lost saved content**: bookmarks/Reddit saves unsearchable → rejected (crowded, weak backend signal).
- **Agent memory across sessions**: rejected (author already uses ai-memory).

### Prior art

| Project | What | Overlap | Our angle |
|---|---|---|---|
| [Unblocked](https://getunblocked.com/) | Commercial Q&A over code, PRs, Slack, Jira, with citations + MCP | High for #1 | Validates the problem. We are open-source, self-hosted, local-first, with public evals. |
| [git-why](https://hexapode.github.io/git-why/) | Stores AI reasoning traces in `.why/` at commit time | Low | Prospective only; we explain *existing* history. |
| [Lore (arXiv)](https://arxiv.org/html/2603.15566v1) | Commit messages as a knowledge protocol for agents | Low | Convention, not retrieval. |
| [repowise](https://docs.repowise.dev/intelligence/git-history) | Git history / co-change analytics | Partial | Not "why" Q&A. |
| [stacktale](https://github.com/stacktale/stacktale) | Java: similar errors from your own app | Medium for #2 | Java-only, own errors only. |
| [Stack-Trace-to-Context-Bundler](https://github.com/vedant-2701/Stack-Trace-to-Context-Bundler) | Bundles trace + code + blame for AI | Low | No retrieval. |
| [Stack-fix](https://github.com/Dhanya562004/Stack-fix) & similar | Paste error → LLM explains | Low | No retrieval → hallucinated fixes. |

**Gap we fill:** Node stack trace → your `package.json`/lockfile → **resolved issues + fix PRs from your dependencies at your installed version**, plus your own repo's past fixes. Found nothing doing this.

**Real competitor:** an agent (Claude Code) using `gh` CLI directly. archaeo must beat it on accuracy, latency, tokens/cost. This comparison is part of the evals.

## What we are building

### User-facing surface

```
archaeo init                 # register current repo, index its closed issues/PRs
archaeo sync                 # incremental update (cursor on updated_at)
archaeo trace < err.log      # main: stack trace → resolved issues + fixes
archaeo trace --answer       # optional LLM-written summary (needs API key)
archaeo trace --feedback good|bad   # dogfooding signal
archaeo eval                 # run the eval suite, print comparison table
archaeo why file:line        # v0.2
```

MCP tools: `trace_error(stacktrace, cwd)`, later `why(file, line, cwd)`.

### Daily flow

```
Agent runs tests → error with node_modules frames
  → CLAUDE.md rule / tiny skill: "dependency error? call trace_error first"
  → archaeo returns top-3 issues, fix version, workaround, links
  → agent bumps version / applies workaround, cites the issue in the commit
```

Human: `npm test 2>&1 | archaeo trace`.

### How `trace` works

1. Parse the stack trace. Frames under `node_modules/<pkg>` → suspect packages; lockfile → exact versions.
2. Package → GitHub repo via npm registry `repository` field.
3. Candidate fetch: GitHub search on that repo (error message + key identifiers) → 30–50 issues.
4. Embed + hybrid rank (vector + keyword, RRF). Boost closed issues with linked fix PRs. Version check (fixed in > installed).
5. Cache everything in pgvector → next similar error hits the cache, not GitHub.
6. Also search the current repo's own indexed history ("we fixed this in PR #812").

### Scope / data layers (one global install)

| Layer | Scope |
|---|---|
| Dependency issues | Global cache shared across all repos |
| Own repo history | Per repo, detected from git remote of `cwd` |
| Org (optional, later) | Across repos of the same org (microservices) |

One Postgres container, `repo_id` on every row, data in a Docker volume. MCP registered at user scope (`claude mcp add --scope user`) so it works in every session.

### Architecture

```
packages/core   ingest (GitHub GraphQL, npm registry), chunking, embeddings,
                hybrid retrieval, ranking, OTel instrumentation
apps/cli        thin wrapper
apps/mcp        thin wrapper (MCP TS SDK)
evals/          golden-set miner, runner, results table
docker-compose  postgres + pgvector (+ optional Jaeger)
docs/adr/       architecture decision records
```

## How we evaluate

**Golden set, no hand labeling:** GitHub issues **closed as duplicate** in popular Node libraries (fastify, prisma, axios, pg, aws-sdk-js-v3…).
- Query = the duplicate issue's stack trace.
- Ground truth = the canonical issue it duplicates (+ its fix PR).
- 50–100 pairs, mined by script.

**Avoid leakage:** remove the query issue from the index; **time split** — index issues before date T, query with issues after T.

| Layer | Metrics |
|---|---|
| Retrieval | recall@5, MRR |
| Answer | correct fix/version found (LLM judge + manual spot-check), hallucination rate |
| System | p50/p95 latency, tokens, $/query |

**Baselines:** (1) GitHub keyword search with raw error, (2) LLM alone without retrieval, (3) **Claude Code + `gh` CLI**.
**Ablations:** vector-only vs hybrid, ± reranker, ± version filter.
**Dogfooding:** `--feedback` → useful-rate in README.

## Production concerns (build these in, not bolt on)

- **Ingest:** idempotent upserts, content-hash dedupe, cursor-based incremental sync, delete propagation, GitHub rate-limit handling (honor `retry-after`, GraphQL cost), bounded concurrency.
- **Embeddings:** store `embedding_model` + `dim` + `chunker_version` per row; refuse mixed-model queries; cache by content hash.
- **Errors:** typed errors, retry only 429/5xx with exponential backoff + jitter, `AbortController` timeouts, request deadline budget, graceful degradation (reranker down → skip; GitHub down → cache only).
- **Observability:** OpenTelemetry span per stage, structured logs (pino) with trace ID, metrics for latency/tokens/cost/cache-hit. No prompt/content logging by default.
- **Privacy/trust:** local embeddings by default, no telemetry, API bound to localhost, token from `gh auth token`, README states exactly what leaves the machine in each mode.

## Sr-level pitfalls to document in the README

1. No eval = no claim.
2. Naive fixed-size chunking breaks stack traces and code blocks.
3. Embedding model drift (mixing models/dims silently breaks search).
4. Stale index / ghost records.
5. Pure vector search misses exact identifiers (error codes, function names) → hybrid.
6. Context stuffing (large top-k hurts cost, latency, accuracy).
7. HNSW + metadata filter recall issues (post-filter returns < k).
8. Prompt injection via retrieved issue text — retrieved content is untrusted data.
9. Secrets/PII in indexed content.
10. Unbounded external calls (no timeout/retry budget/cost cap).
11. LLM-as-judge bias — spot-check by hand.
12. Non-deterministic tests — unit-test deterministic parts, LLM behavior goes in evals.
13. Over-abstraction vs lock-in — one interface, one implementation.
14. Scaling limits stated honestly (single Postgres, sync ingest → queue + workers later).

## Milestones (small, learn-as-you-go)

Each milestone is small enough to plan, build, and discuss in one sitting. Before starting one: review its learning goal and open questions. After finishing: note what surprised you.

Order: opening setup first, evidence contract second, then the learning sequence (Phases A to F) as previously proposed. Each milestone tracks implementation complete and learning demonstrated separately; passing implementation checks does not complete a learning milestone.

### Opening milestones

| M | Deliverable | Learn | Ownership | Done when |
|---|---|---|---|---|
| **Setup** | Git repository on `main`, public GitHub repo, agent workflow docs (`AGENTS.md`, `docs/agents/`), five triage labels, minimal README | How tasks, agent instructions, and domain documentation guide development | Assistant: setup, configuration, checks. Learner: review the workflow and demonstrate how it protects learner exercises | Implementation: initial commit pushed, labels exist. Learning: learner explains the workflow (for example, how `ready-for-human` affects assistant behavior) |
| **Evidence contract** | To be defined. Discuss and agree on its outcome before beginning | To be agreed | To be agreed | To be agreed |

TypeScript tooling and pre-commit hooks arrive with the first code milestone (M0). A license is selected before any code is released.

Embeddings and pgvector remain decisions to evaluate; the milestones below that name them describe the provisional direction.

### Phase A — Foundations

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M0** | Repo skeleton: TS strict, pnpm workspace, vitest, eslint, docker-compose with pgvector, CI running tests | Project hygiene baseline | `pnpm test` green locally and in CI; `docker compose up` gives a Postgres with `vector` extension |
| **M1** | Schema + migrations: `repos`, `documents`, `chunks(embedding vector(N), tsv tsvector)`, HNSW index | Vector column types, distance metrics (cosine/L2/IP), HNSW vs IVFFlat | Insert a hand-made vector and query nearest neighbors in SQL |
| **M2** | In-process embeddings module (transformers.js, small model), batch + content-hash cache | What embeddings are, dimensions, normalization, why model version must be stored | Embed 3 sentences; similar ones have higher cosine similarity; unit tests pass |

### Phase B — Ingest

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M3** | GitHub client: GraphQL pagination for issues + comments of one repo, typed errors, retry/backoff, rate-limit awareness | Robust external API ingestion | Fetch all closed issues of a small repo without hitting limits; retry tested with a fake 429 |
| **M4** | Chunker for issues: title + body + comments, stack trace/code block aware, metadata (repo, number, state, labels, linked PR, dates) | Chunking strategies and why structure matters | Unit tests: stack traces and code blocks never split |
| **M5** | Idempotent ingest pipeline: upsert, hash dedupe, cursor sync | Idempotency, incremental sync | Running ingest twice changes nothing; new issue appears after `sync` |

### Phase C — Retrieval + evals (evals before optimization)

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M6** | Vector search + `archaeo search "<text>"` CLI | kNN in pgvector, top-k | Returns sensible issues for a pasted error |
| **M7** | Golden-set miner (duplicate-closed issues, time split) | Eval design, leakage | 50+ pairs saved as JSON |
| **M8** | Eval runner: recall@5, MRR + keyword-search baseline | Retrieval metrics | First results table printed |
| **M9** | Hybrid search (tsvector + vector, RRF) | Hybrid retrieval, fusion | Ablation row shows delta vs vector-only |

### Phase D — The `trace` feature

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M10** | Stack trace parser + dependency resolver (frames → packages → lockfile versions → npm → GitHub repo) | Parsing, npm registry | Unit tests on real Node traces |
| **M11** | `archaeo trace`: lazy fetch → embed → rank → cache, version-aware boost | On-demand RAG, caching | Real error returns the right issue; second run is served from cache |
| **M12** | Own-repo layer: `archaeo init` / `sync` for current repo; merged results | Multi-scope retrieval | Trace shows both dependency and own-repo hits |

### Phase E — Daily use

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M13** | MCP server with `trace_error`, registered at user scope + CLAUDE.md rule | MCP protocol, tool design for agents | Claude Code calls it automatically on a dependency error |
| **M14** | Observability: OTel spans, pino logs, latency/cache-hit metrics, Jaeger in compose | Tracing a RAG pipeline | Jaeger screenshot of one trace with all stages |
| **M15** | `--feedback` logging + start dogfooding | Online vs offline metrics | Useful-rate visible after a week of use |

### Phase F — Proof

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M16** | Agent baseline eval: Claude + `gh` vs Claude + archaeo (accuracy, latency, tokens) | Evaluating agents, cost measurement | Comparison table with real numbers |
| **M17** | Optional LLM: `trace --answer` + LLM judge in evals (+ optional reranker ablation) | LLM integration, judge bias | Answer faithfulness metric in table |
| **M18** | README (pitfalls, limitations, prior art, results), ADRs, demo `pg_dump` snapshot, publish | Communicating trade-offs | Public repo; stranger can run `archaeo demo` in minutes |

### v0.2 backlog

- `archaeo why file:line` (PR/review/commit ingestion for own repo)
- `trace --save` → reviewed `docs/errors/<slug>.md`
- Index repo `/docs` and ADRs as an extra source (may pull into v0.1 if cheap)
- Org scope across repos
- Cloud embedding provider option; sqlite-vec zero-setup mode (ADR)

## Open questions (decide later)

- Whether to use embeddings and pgvector at all, and in what form — evaluate before committing; record the outcome as an ADR.
- Exact local embedding model (quality vs speed on CPU) — decide in M2 with a tiny benchmark.
- Which libraries form the eval corpus — decide in M7 based on duplicate-issue counts.
- Positioning for OSS launch: "guardrail/context for AI agents" vs "debugging tool".
- Revisit overall direction after v0.1 results.

## Sources

- Unblocked: https://getunblocked.com/ · MCP: https://getunblocked.com/unblocked-mcp/
- git-why: https://hexapode.github.io/git-why/
- Lore: https://arxiv.org/html/2603.15566v1
- repowise: https://docs.repowise.dev/intelligence/git-history
- stacktale: https://github.com/stacktale/stacktale
- Stack-Trace-to-Context-Bundler: https://github.com/vedant-2701/Stack-Trace-to-Context-Bundler
- Stack-fix: https://github.com/Dhanya562004/Stack-fix
- Obsidian plugin security: https://obsidian.md/help/plugin-security
- Obsidian RAG plugins: https://community.obsidian.md/plugins/rag-chat, https://community.obsidian.md/plugins/analogy-rag-in-your-vault
- Lost "why" in code: https://github.com/secrinlabs/secrin, https://github.com/jsomers/git-getpull-1, https://dev.to/leena_malhotra/a-better-way-to-document-your-engineering-thought-process-2c6a
