# Repository guidance

## Work and verification

Track project scope, dependencies, acceptance criteria, and verification evidence in GitHub Issues. When planning milestones, keep implementation and focused tests in separate work units. Run the repository's ordinary verification for each change. Follow the [issue tracker conventions](docs/agents/issue-tracker.md) and [status labels](docs/agents/triage-labels.md).

Read [the project brief](docs/PROJECT_BRIEF.md) before planning project work. It records scope, milestones, and open decisions.

## Source and generated files

The planned CLI and MCP packages use TypeScript. Keep authored source and generated output distinct, and regenerate generated files with repository scripts. Do not edit generated output by hand.

Record consequential architecture decisions in `docs/adr/` when their trade-offs merit a durable record. Use [the domain glossary](CONTEXT.md) for settled terms.

## Configuration and hooks

Use environment variables for runtime configuration. Keep `.env.example` free of secrets. Use package scripts for verification and Lefthook if repository hooks are added.

## Repository copy

Use the humanize skill when creating or revising user-facing documents and repository copy. Preserve technical meaning and factual claims. Omit AI co-author and generated-by trailers from commit messages. Keep other attribution accurate. Record project workflow without personal information.

## Agent references

- **Issue tracker:** use GitHub Issues. See [docs/agents/issue-tracker.md](docs/agents/issue-tracker.md).
- **Triage labels:** use the project status labels. See [docs/agents/triage-labels.md](docs/agents/triage-labels.md).
- **Domain docs:** use one project context. See [docs/agents/domain.md](docs/agents/domain.md).
