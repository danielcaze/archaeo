# archaeo

Find fixes hidden in dependency issue trackers and repository history.

`archaeo` is a planned, self-hosted developer tool. Give it a Node.js stack trace and it will search resolved GitHub issues and fix pull requests from your dependencies, then check your repository's history for a matching repair. Results should cite the issue or pull request, name the version that fixed the problem, and include a documented workaround when one exists.

The planned interfaces are a CLI for developers and an MCP server for coding agents. Searches will account for installed dependency versions and include the current repository's history.

## Status

**Planning.** No runnable code yet. Repository and agent workflow setup are complete. The evidence contract is the next milestone, after setup and before implementation. Embeddings, pgvector, and retrieval design remain under evaluation.

## Evaluation goals

Measure retrieval against known issue and fix pairs. Compare direct GitHub search by accuracy, latency, and cost, then report whether results include useful citations and version details. No measurements or shipped capabilities exist yet.

## Project documents

The [project brief](docs/PROJECT_BRIEF.md) records scope, milestones, and open decisions. [AGENTS.md](AGENTS.md) describes contributor and agent workflow.

No license has been chosen. Choose the project's license before its first code release.
