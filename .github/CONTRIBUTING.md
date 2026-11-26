---
type: guide
title: "Contributing to the Sofa King DRAFT Drafting Table"
description: "Contributor guide for Sofa King's DRAFT architecture workspace — branching, validation, and PR workflow."
tags:
  - draft
  - sofaking
  - contributing
  - guide
timestamp: 2026-06-13T00:00:00-05:00
---
# Contributing to the Sofa King DRAFT Drafting Table

This repository is Sofa King's DRAFT architecture workspace. It is a
**living architecture catalog** that is continuously validated and published
(the DRAFT browser deploys from `main` to GitHub Pages). This document is the
canonical contribution standard for both human contributors and AI assistants.

> AI assistants: the AI bootstrap ([AGENTS.md](../AGENTS.md)) points here for
> branching and pull-request process. Follow this file when creating branches,
> commits, and pull requests.

## Branching strategy: GitHub Flow

This repo uses **GitHub Flow** — a single, always-deployable trunk with
short-lived branches. We deliberately do **not** use Git Flow
(`develop`/`release`/`hotfix`); there is no released, versioned artifact to
manage in parallel, and the DRAFT tooling (validation, browser generation,
Pages deploy, framework-update bot) is built around `main` as the single
integration and publish branch.

- **`main`** is protected, always valid, and always publishable. Never commit
  to it directly.
- Create a **short-lived branch** off `main` for each change, and open a pull
  request back into `main`.
- Branches are deleted automatically on merge.

### Branch naming

Use a type prefix and, where applicable, the issue number:

| Prefix | Use for |
|---|---|
| `feature/` | New catalog content or capabilities |
| `fix/` | Corrections to existing content |
| `chore/` | Maintenance, CI, config, tooling |
| `docs/` | Documentation-only changes |
| `draft/` | DRAFT framework updates (often bot-authored) |

Example: `fix/issue-25-codeowners-paths`.

## Pull request workflow

1. Branch from `main`.
2. Make your change in the correct workspace path (`catalog/`,
   `configurations/`, or `.draft/workspace.yaml` — see
   [AGENTS.md](../AGENTS.md) for the workspace boundary). Do **not** edit
   `.draft/framework/**`.
3. Validate locally before opening the PR:
   ```bash
   python .draft/framework/tools/validate.py --workspace .
   ```
   You may regenerate the browser locally to preview or verify your change:
   ```bash
   python .draft/framework/tools/generate_browser.py --workspace . --output docs/index.html
   ```
   Do **not** commit the result. The generated files are untracked (see
   `.gitignore`) and are rebuilt from the catalog at deploy time by
   `deploy-pages.yml`. Committing them causes PRs to overwrite each other's
   objects in the artifact.
4. Open a pull request into `main`.
5. The PR must pass the required **`validate`** status check and receive **one
   code-owner approval** (`.github/CODEOWNERS` routes the review).
6. Merges are **squash-only**; the branch is deleted automatically. Enable
   auto-merge if you want it to land as soon as the checks and review clear.

## Commit and PR conventions

- Write commit subjects in conventional-commit style:
  `feat:`, `fix:`, `chore:`, `docs:`, `ci:`.
- Reference the issue the work closes (e.g. `— closes #25`).
- Keep a PR scoped to one logical change; squash means one PR becomes one
  commit on `main`.

## Reviews and ownership

`.github/CODEOWNERS` determines required reviewers by path. A code-owner
approval is required before merge. You cannot approve your own PR, so coordinate
with another owner when needed.

## Framework changes

The vendored framework under `.draft/framework/**` and `.draft/framework.lock`
is not edited by hand. Framework updates go through the explicit update workflow
(`/draft update` or the scheduled framework-update PR) and are reviewed as
ordinary Git changes.
