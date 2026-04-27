# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository is a freshly-bootstrapped proof-of-concept (`poc/resume`). It currently contains only CI/agent configuration — no application source, build system, tests, or dependency manifest exists yet. IntelliJ project files (`.idea/php.xml`) indicate the intended stack is **PHP**, but no `composer.json` or PHP sources have been added.

When asked to "add a feature," confirm with the user whether they want you to also bootstrap the project skeleton (e.g. `composer init`, framework choice, directory layout) — do not silently invent one.

## Agent-driven workflow

The only meaningful configuration in this repo is two GitHub Actions in `.github/workflows/` that wire Claude Code into the GitHub issue/PR loop. Understand both before making changes that could affect them:

- **`agent-dev.yml`** — triggered when an issue gets the `agent` label. Removes the label, then runs Claude Code with the issue body as the prompt. Claude is expected to:
  1. Create a branch named `agent/issue-<N>`.
  2. Implement the issue.
  3. Open a PR referencing the issue.
  4. **Always** post a summary comment on the issue (PR link, what was done, what was skipped, blockers). This comment is the only signal the issue author gets — never skip it.

- **`agent-review.yml`** — triggered on `pull_request_review` events (any non-approved review) for branches matching `agent/issue-*`. Claude:
  1. Checks out the PR branch.
  2. Fetches all review comments via `gh api .../pulls/<N>/comments` and `.../reviews`.
  3. Applies every requested change, commits, and pushes to the **same** branch (do not open a new PR).
  4. **Always** posts a summary comment on the PR listing each review item as Applied / Skipped / Failed.

Both workflows pass `--model claude-opus-4-7 --max-turns 300 --allowedTools Bash,Read,Edit,MultiEdit,Write,Glob,Grep`. They use `secrets.GH_PROJECT_TOKEN` (not the default `GITHUB_TOKEN`) so that PRs/comments created by the agent can themselves trigger downstream workflows.

When editing these workflows, preserve the "always post a summary comment, even on partial runs" contract — the failure-notice steps at the bottom exist as a fallback when the Claude step itself crashes.

## Branch convention

Agent-authored branches must be named `agent/issue-<N>` — `agent-review.yml` only fires for branches matching that prefix.

## CV content source of truth

The canonical source for the CV's textual content (FR and EN) is this secret gist:

https://gist.github.com/maxence-drouhin/5bdaab1a511d3317744ee12eb386a80f

Workflow: the gist holds the French content in markdown; the user updates it (often from mobile) and treats it as the reference. Before any task that touches CV content in `index.html` (FR) or `en/index.html` (EN), fetch the gist with `gh gist view 5bdaab1a511d3317744ee12eb386a80f` and reconcile differences — propagate gist → HTML, never the other way around. Translate FR → EN for `en/index.html` when the gist changes content.

CSS, layout, structure, and print styles are owned by the HTML files themselves — the gist is content-only, so don't expect it to drive markup or styling decisions.
