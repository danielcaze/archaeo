# archaeo

Find the fixes buried in your dependencies' issue trackers and your own
repository's history.

archaeo is a planned self-hosted developer tool that takes a Node.js stack
trace and retrieves relevant resolved GitHub issues and fix pull requests.
It aims to connect errors to cited evidence: what went wrong, which version
fixed it, and whether a documented workaround exists.

The planned interfaces are a CLI for developers and an MCP server for AI
coding agents. Retrieval will consider dependency versions and search both
dependency issue trackers and the current repository's history.

## Status

**Planning — no runnable code yet.** Repository and agent workflow setup
is complete. The next milestone is to agree on an evidence contract before
implementation begins. Embeddings, pgvector, and the retrieval architecture
remain decisions to evaluate.

## What we want to measure

- Retrieval quality against known issue/fix pairs.
- Accuracy, latency, and cost compared with direct GitHub search.
- Whether results provide useful citations and version information.

These are evaluation goals, not measured results or shipped capabilities.

## Project documents

See [docs/PROJECT_BRIEF.md](docs/PROJECT_BRIEF.md) for the current scope,
milestones, and open decisions. Contributor and agent workflow lives in
[AGENTS.md](AGENTS.md).

No license has been chosen yet. One will be selected before any code is
released.
