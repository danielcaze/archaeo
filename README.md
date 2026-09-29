# archaeo

Find fixes hidden in dependency issue trackers and repository history.

`archaeo` is a planned, self-hosted developer tool. Give it a Node.js stack trace and it will search for resolved GitHub issues and fix pull requests from your dependencies, then check your repository's history for a matching repair. Each result should cite the issue or pull request and name the version that fixed the problem. If the source documents a workaround, include that too.

The planned interfaces are a CLI for developers and an MCP server for coding agents. Searches will account for installed dependency versions and include the current repository's history.

## Status

**Planning.** No runnable code yet. Repository and agent workflow setup is complete. The next milestone is an agreement on the evidence contract, and implementation waits until we reach it. Embeddings, pgvector, and the retrieval design remain under evaluation.

## What we want to measure

Measure retrieval against known issue and fix pairs. Compare it with direct GitHub search by accuracy, latency, and cost, then check whether results give useful citations and version details.

These are evaluation goals. We have no measurements or shipped capabilities to report yet.

## Project documents

The [project brief](docs/PROJECT_BRIEF.md) records the scope, milestones, and open decisions, while [AGENTS.md](AGENTS.md) describes contributor and agent workflow.

No license has been chosen. We will choose the project's license before the first release that includes code.
