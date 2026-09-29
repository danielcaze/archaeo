# archaeo project brief

> **Planning** (2026-09-28). No code yet. This temporary v0.1 scope will be revisited after a first run gives us results.
>
> As of 2026-09-29, embeddings and pgvector remain decisions to evaluate. The proposals and research below describe the current working direction until an ADR settles them. Milestones start with setup, then the evidence contract. See "Milestones".

## At a glance

`archaeo` is a planned, self-hosted, open-source tool with a CLI and an MCP server. It takes a **stack trace** and searches for **resolved GitHub issues and fix PRs** in your dependencies and your own repository's history. The current proposal uses retrieval-augmented generation over a local pgvector database, pending evaluation.

Your coding agent hits an error. It calls `archaeo` through MCP and gets the likely resolved issue with citations. The result also gives the fix version. A documented workaround appears when the source has one. Those citations point back to the issue or PR with the fix details.

The v0.2 follow-up is `archaeo why file:line`, which explains *why* a line of code exists by searching PRs, reviews, and issues.

## Why we are doing this

The goals below are listed in priority order:

1. **Show work for a Sr. Backend AI Engineer role.** The project should show judgment around RAG, embeddings, vector databases, and LLM integration. That means measuring retrieval quality, handling errors, adding observability, and stating trade-offs.
2. **Use it every day.** The author and other developers should be able to use the tool, then improve it from that experience.

A senior screen looks at retrieval quality, failure behavior, reasons for the design, and query cost. The project should answer those questions with measurements.

## How we got here (decision log)

| # | Idea considered | Decision | Reason |
|---|---|---|---|
| 1 | RAG CLI over a local docs folder | Rejected as-is | This is a common RAG tutorial shape. Evals, trade-offs, and failure handling would give it more engineering depth. |
| 2 | Obsidian plugin | Rejected | The space is crowded (Smart Connections, Copilot, RAG Chat, Analogy, and others). Obsidian's plugin runtime cannot host pgvector, hides backend work, takes weeks to review, and has a weak trust model because Obsidian cannot sandbox plugins. |
| 3 | Standalone service indexing the Obsidian vault | Rejected | The author already has this workflow through Obsidian MCP and ai-memory. |
| 4 | Research real developer pain points | Found three candidates | See "Problem research". |
| 5 | #1 "why is this code like this?" and #2 "stack trace to resolved issues" | **Chosen**, #2 primary | Developers are likely to need #2 across more errors. #1 mainly helps with older team repositories. |
| 6 | One program or two? | **One program** | Both use GitHub ingestion, chunking, hybrid search, pgvector, and evals. #2 also needs #1's linked-fix-PR machinery. |
| 7 | Form: skill, MCP, or CLI | **CLI + MCP**, optional tiny skill | A skill alone would send the agent to `gh` for searches without a persistent index. Use the CLI for demos, evals, and CI, then register MCP for daily agent use. |
| 8 | Vector DB | **Provisional, to evaluate:** pgvector via docker-compose | Metadata and vectors share one transactional store. SQL filters and hybrid search fit in one query, and AWS RDS/Aurora can run it. Docker works locally. |
| 9 | Ship a prebuilt DB? | **No**, ship empty and index on demand | A prebuilt database would go stale and could not include private repositories. An optional `pg_dump` snapshot can support demos and evals. |
| 10 | Does it need an LLM? | **Not in MCP mode** | The calling agent supplies the LLM. `archaeo` retrieves and returns cited evidence. The CLI may use an LLM for `--answer`, and evals may use one to judge results. |
| 11 | Embeddings | **Provisional, to evaluate:** local and in-process by default (for example, `bge-small` via transformers.js), with cloud as an option | The local route needs no API key or Ollama, keeps data on the machine, and has no API embedding fee. Use the same model each time so stored vectors and query vectors match. The agent cannot generate embeddings with a different model. |
| 12 | Auto-generate docs under `/docs`? | **Read `/docs` now, write later (v0.2)** | Indexing existing docs is cheap. Generating docs is a different product, harder to evaluate, and carries hallucination and doc-rot risks. In v0.2, `trace --save` can write a reviewed `docs/errors/<slug>.md` after a fix is confirmed. |

## Problem research (summary)

Most recurring pains came from blogs, Hacker News, and articles quoting Reddit. Reddit and X were not directly readable.

- **Lost "why".** Teams forget why code exists. `git blame` gives a cryptic commit, and the reasoning may have vanished with an old Slack thread.
- **Repeated debugging.** Runbooks and postmortems get buried, so the same errors are debugged from scratch.
- **Lost saved content.** Bookmarks and Reddit saves are hard to search. The category is crowded, and it offers a weak backend engineering signal, so this idea was rejected.
- **Agent memory across sessions.** The idea was rejected because the author already uses ai-memory.

### Prior art

| Project | What | Overlap | Our angle |
|---|---|---|---|
| [Unblocked](https://getunblocked.com/) | Commercial Q&A over code, PRs, Slack, and Jira, with citations and MCP | High for #1 | It validates the problem. Our plan is open-source, self-hosted, local-first, with public evals. |
| [git-why](https://hexapode.github.io/git-why/) | Stores AI reasoning traces in `.why/` at commit time | Low | It captures new reasoning as it is written. We search history that already exists. |
| [Lore (arXiv)](https://arxiv.org/html/2603.15566v1) | Commit messages as a knowledge protocol for agents | Low | A writing convention for commits, rather than a retrieval system. |
| [repowise](https://docs.repowise.dev/intelligence/git-history) | Git history and co-change analytics | Partial | Git analytics without Q&A about why. |
| [stacktale](https://github.com/stacktale/stacktale) | Java tool that finds similar errors in your app | Medium for #2 | Java only, and limited to your own errors. |
| [Stack-Trace-to-Context-Bundler](https://github.com/vedant-2701/Stack-Trace-to-Context-Bundler) | Bundles a trace, code, and blame for AI | Low | It gathers context without retrieval. |
| [Stack-fix](https://github.com/Dhanya562004/Stack-fix) and similar tools | Paste an error for an LLM explanation | Low | Without retrieval, a proposed fix can be hallucinated. |

Follow a Node stack trace through `package.json` and the lockfile to find **resolved issues and fix PRs from dependencies at the installed version**. Then search the current repository's history for past fixes. We found no tool that does both.

Claude Code with the `gh` CLI gives developers another way to search for an error. The goal is for `archaeo` to beat that baseline on accuracy, latency, and token and query costs, with evals showing whether it does.

## What we are building

### User-facing surface

```
archaeo init                 # register this repo and index its closed issues/PRs
archaeo sync                 # update incrementally from an updated_at cursor
archaeo trace < err.log      # search a stack trace for resolved issues and fixes
archaeo trace --answer       # optional LLM summary, requires an API key
archaeo trace --feedback good|bad   # record a dogfooding signal
archaeo eval                 # run evals and print a comparison table
archaeo why file:line        # planned for v0.2
```

The MCP server will expose `trace_error(stacktrace, cwd)`, followed later by `why(file, line, cwd)`.

### Daily flow

```
Agent runs tests and hits an error in a node_modules frame
  → A CLAUDE.md rule or small skill says, "For dependency errors, call trace_error first"
  → archaeo returns the top 3 issues, fix version, workaround, and links
  → agent upgrades the dependency or applies the workaround, then cites the issue in the commit
```

A person can also run `npm test 2>&1 | archaeo trace`.

### How `trace` works

1. Parse the stack trace. Frames under `node_modules/<pkg>` point to candidate packages. The lockfile supplies exact versions.
2. Resolve each package to a GitHub repository through the npm registry's `repository` field.
3. Search that repository on GitHub with the error message and key identifiers. Fetch 30 to 50 issue candidates.
4. Embed and rank with vector and keyword search, then fuse results with RRF. Boost closed issues linked to fix PRs. Compare the fixed version with the installed version.
5. Cache fetched data in pgvector. A similar trace can reuse the cache instead of calling GitHub again.
6. Search the current repository's indexed history too, for fixes such as "we fixed this in PR #812".

### Scope / data layers (one global install)

| Layer | Scope |
|---|---|
| Dependency issues | A global cache shared by all repositories |
| Own repo history | One repository at a time, detected from the git remote for `cwd` |
| Org (optional, later) | Repositories in the same organization, such as microservices |

The current sketch uses one Postgres container, a `repo_id` on every row, and a Docker volume for data. Register MCP at user scope (`claude mcp add --scope user`) to make it available in every session.

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

Build a golden set without hand labeling. Mine GitHub issues **closed as duplicate** in popular Node libraries (fastify, prisma, axios, pg, aws-sdk-js-v3…).

- **Query.** The duplicate issue's stack trace.
- **Ground truth.** The canonical issue it duplicates, plus its fix PR.
- **Size.** 50 to 100 pairs, mined by script.

Keep the query issue out of the index. Split by time: index issues from before date T, then query with issues from after T.

| Layer | Metrics |
|---|---|
| Retrieval | recall@5, MRR |
| Answer | Correct fix and version found (LLM judge plus manual spot-check), hallucination rate |
| System | p50/p95 latency, tokens, cost per query |

Compare three baselines: GitHub keyword search with the raw error, an LLM alone without retrieval, and **Claude Code + `gh` CLI**.

Run ablations for vector-only versus hybrid search, with and without a reranker, and with and without a version filter.

Use `--feedback` to track the useful rate in the README.

## Production concerns (build these in from the start)

- **Ingest.** Make upserts idempotent, dedupe by content hash, sync from a cursor, and propagate deletes. Respect GitHub rate limits, including `retry-after` and GraphQL cost, and bound concurrency.
- **Embeddings.** Store `embedding_model`, `dim`, and `chunker_version` per row. Reject queries that mix models, and cache by content hash.
- **Errors.** Use typed errors. Retry only 429 and 5xx responses, with exponential backoff and jitter. Add `AbortController` timeouts and a request deadline budget. If the reranker fails, skip it. If GitHub is down, use the cache only.
- **Observability.** Add an OpenTelemetry span for each stage. Use structured pino logs with a trace ID, and track latency, tokens, cost, and cache hits. Do not log prompts or content by default.
- **Privacy and trust.** Use local embeddings by default and collect no telemetry. Bind the API to localhost and get the token from `gh auth token`. The README must state exactly what leaves the machine in each mode.

## Sr-level pitfalls to document in the README

1. Publish a performance claim only after an eval supports it.
2. Fixed-size chunking can split stack traces and code blocks.
3. Mixing embedding models or dimensions can silently break search.
4. A stale index can return records that should have been removed.
5. Vector search alone misses exact error codes and function names, so combine it with keyword search.
6. A large top-k stuffs the context and raises cost and latency while hurting accuracy.
7. HNSW with metadata filters can return fewer than k results after post-filtering.
8. Retrieved issue text can contain prompt injection. Treat it as untrusted data.
9. Indexed content can contain secrets or personal information.
10. Bound external calls with timeouts, retry budgets, and cost caps.
11. LLM judges can be biased. Spot-check their ratings by hand.
12. Tests can be nondeterministic. Unit-test deterministic parts and put LLM behavior in evals.
13. Weigh abstraction costs against provider lock-in. Start with one interface and one implementation.
14. State scaling limits plainly: one Postgres instance and synchronous ingestion. Add a queue and workers later if needed.

## Milestones (small, learn-as-you-go)

One sitting per milestone. Before starting, review its learning goal and open questions, and agree on prerequisites, ownership, exercise, and completion checks. Afterward, write down what surprised you.

Setup comes first. Discuss and agree on the evidence contract next, before moving into Phases A through F, and record implementation completion separately from the learner's demonstration of each milestone's learning objective. Passing implementation checks alone leaves the learning review open.

### Opening milestones

| M | Deliverable | Learn | Ownership | Done when |
|---|---|---|---|---|
| **Setup** | Git repository on `main`, public GitHub repo, agent workflow docs (`AGENTS.md`, `docs/agents/`), five triage labels, minimal README | How tasks, agent instructions, and domain documentation shape development | Assistant handles setup, configuration, and checks. Learner reviews the workflow and explains how it protects learner exercises. | **Implementation:** initial commit pushed and labels exist. **Learning:** learner explains the workflow, such as how `ready-for-human` changes assistant behavior. |
| **Evidence contract** | To be defined. Discuss and agree on its outcome before starting. | To be agreed | To be agreed | To be agreed |

TypeScript tooling and pre-commit hooks arrive with M0, the first code milestone. Choose a license before releasing any code.

The embeddings and pgvector decisions remain under evaluation. Milestones that name them describe the provisional direction.

### Phase A: Foundations

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M0** | Repo skeleton: TS strict, pnpm workspace, vitest, eslint, docker-compose with pgvector, CI running tests | Project hygiene baseline | `pnpm test` passes locally and in CI. `docker compose up` starts Postgres with the `vector` extension. |
| **M1** | Schema and migrations: `repos`, `documents`, `chunks(embedding vector(N), tsv tsvector)`, HNSW index | Vector column types, distance metrics (cosine/L2/IP), HNSW vs IVFFlat | Insert a hand-made vector and query its nearest neighbors in SQL. |
| **M2** | In-process embeddings module (transformers.js, small model), batch processing, and content-hash cache | What embeddings are, dimensions, normalization, and why the model version must be stored | Embed 3 sentences. Similar ones should have higher cosine similarity, and unit tests should pass. |

### Phase B: Ingest

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M3** | GitHub client: GraphQL pagination for one repo's issues and comments, typed errors, retry/backoff, rate-limit awareness | External API ingestion that handles failures and rate limits | Fetch all closed issues from a small repo without hitting limits. Test a retry with a fake 429. |
| **M4** | Issue chunker for titles, bodies, and comments, aware of stack traces and code blocks, with metadata for repo, number, state, labels, linked PR, and dates | Chunking strategies and why structure matters | Unit tests show that stack traces and code blocks stay intact. |
| **M5** | Idempotent ingest pipeline: upsert, hash dedupe, cursor sync | Idempotency and incremental sync | Run ingest twice with no changes. Then add an issue and confirm it appears after `sync`. |

### Phase C: Retrieval + evals (evals before optimization)

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M6** | Vector search and `archaeo search "<text>"` CLI | kNN in pgvector, top-k | Returns sensible issues for a pasted error. |
| **M7** | Golden-set miner (duplicate-closed issues, time split) | Eval design and leakage | Save 50 or more pairs as JSON. |
| **M8** | Eval runner: recall@5, MRR, and keyword-search baseline | Retrieval metrics | Print the first results table. |
| **M9** | Hybrid search (tsvector + vector, RRF) | Hybrid retrieval and fusion | An ablation row shows the change versus vector-only search. |

### Phase D: The `trace` feature

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M10** | Stack trace parser and dependency resolver (frames → packages → lockfile versions → npm → GitHub repo) | Parsing and npm registry | Unit tests cover real Node traces. |
| **M11** | `archaeo trace`: lazy fetch → embed → rank → cache, with version-aware boost | On-demand RAG and caching | A real error returns the right issue. The second run comes from cache. |
| **M12** | Own-repo layer: `archaeo init` and `sync` for the current repo, with merged results | Multi-scope retrieval | A trace shows hits from both dependencies and the current repo. |

### Phase E: Daily use

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M13** | MCP server with `trace_error`, registered at user scope, plus a CLAUDE.md rule | MCP protocol and tool design for agents | Claude Code calls the tool automatically on a dependency error. |
| **M14** | Observability: OTel spans, pino logs, latency/cache-hit metrics, Jaeger in compose | Tracing a RAG pipeline | Save a Jaeger screenshot showing one trace with all stages. |
| **M15** | `--feedback` logging and start of dogfooding | Online vs offline metrics | The useful rate is visible after a week of use. |

### Phase F: Proof

| M | Deliverable | Learn | Done when |
|---|---|---|---|
| **M16** | Agent baseline eval: Claude + `gh` vs Claude + archaeo (accuracy, latency, tokens) | Evaluating agents and measuring cost | Publish a comparison table with real numbers. |
| **M17** | Optional LLM: `trace --answer` and LLM judge in evals, with an optional reranker ablation | LLM integration and judge bias | Add an answer-faithfulness metric to the table. |
| **M18** | README (pitfalls, limitations, prior art, results), ADRs, demo `pg_dump` snapshot, publish | Communicating trade-offs | Public repo. A stranger can run `archaeo demo` in minutes. |

### v0.2 backlog

- `archaeo why file:line` (PR, review, and commit ingestion for the current repo)
- `trace --save` to write a reviewed `docs/errors/<slug>.md`
- Index repo `/docs` and ADRs as another source. Pull this into v0.1 if it is cheap.
- Org scope across repositories
- Cloud embedding provider option and sqlite-vec zero-setup mode (ADR)

## Open questions (decide later)

- Do we need embeddings and pgvector? If so, in what form? Evaluate before committing and record the outcome in an ADR.
- Which local embedding model gives the best balance of quality and CPU speed? Run a small benchmark in M2.
- Which libraries belong in the eval corpus? Use duplicate-issue counts to decide in M7.
- How should we position the OSS launch: "guardrail/context for AI agents" or "debugging tool"?
- Revisit the overall direction after v0.1 results.

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
