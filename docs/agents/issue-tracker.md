# Issue tracker

Track project outcomes, dependencies, acceptance criteria, and verification evidence in [danielcaze/archaeo GitHub Issues](https://github.com/danielcaze/archaeo/issues). Use `gh` from this repository, or pass `-R danielcaze/archaeo` elsewhere.

## Conventions

- Create issues with `gh issue create --title "..." --body-file <utf8-file>`.
- Read issues with `gh issue view <number> --comments`. Fetch labels with `--json labels` when needed.
- List open issues with `gh issue list --state open --json number,title,body,labels,assignees` and relevant filters.
- Comment with `gh issue comment <number> --body-file <utf8-file>`.
- Add or remove labels with `gh issue edit <number> --add-label <name>` or `--remove-label <name>`. Preserve other labels.
- Close an issue with a multiline explanation by commenting from a body file, then running `gh issue close <number>`.
- Record the outcome, scope, dependencies, acceptance criteria, and verification evidence on each issue.
- Keep implementation and focused tests in separate work units when planning project milestones.

Use GitHub sub-issues and issue dependencies where supported. Otherwise, link related issues and write `Blocked by: #<number>` in the issue body. Update that dependency statement when a blocker clears.

PRs are not a request surface.
