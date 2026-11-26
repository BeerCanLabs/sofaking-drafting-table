---
type: workspace-readme
title: "Sofa King DRAFT Workspace"
description: "Overview of Sofa King's DRAFT architecture catalog — what we deploy, how it connects, and which technologies are approved."
tags:
  - draft
  - sofaking
  - workspace
  - readme
timestamp: 2026-06-13T00:00:00-05:00
---
# Sofa King DRAFT Workspace

This is Sofa King's DRAFT (Deployable Reference Architecture Framework) workspace.

This provides the **company-wide architecture catalog**, documenting **what we deploy** (e.g., services, databases, queues, compute, etc.), **how they connect**, and **which technologies are approved** — focusing on deployment architecture and technology selection, not implementation details.

It vendors the public DRAFT framework under `.draft/framework/` so the workspace can validate and render without pointing the Draftsman back to the public repository at runtime. Framework refreshes are explicit and reviewed.

**What it provides:**
- Architecture as code: validated, versioned in Git, and reviewed in pull requests
- Source of truth for infrastructure and services across the company
- Approval gate for architecture decisions before implementation
- Guardrails ensuring solutions use approved technologies
- Machine-readable architecture for validation, queries, and eventual CI/CD automation

## ⚠️ Contributing? Read This First

Every PR is checked by the DRAFT validator — but **a green check alone does not
mean your entry is valid**: a YAML object with no `type:` field is silently
skipped (it "passes" with zero checks), and markdown under `catalog/` is never
read at all. (A *wrong* `type:` is different — a typo fails loudly.) Before
opening a PR, read **[CONTRIBUTING.md](CONTRIBUTING.md)** — a five-minute,
step-by-step guide to getting a catalog change validated and merged.

## Object Types

Application architectures are defined by composing different architectural object types:

- **`runtime_service`** = Infrastructure that runs things
  - Messaging platforms: AWS SNS/SQS, Confluent Cloud Kafka
  - Compute runtimes: Kubernetes clusters, AWS Lambda, ECS
  - Web/app servers, caches, worker runtimes
  - Delivery models: cloud-managed (PaaS), vendor-hosted (SaaS), or self-managed
- **`data_store_service`** = Data storage infrastructure (DynamoDB, RDS, S3, ElasticSearch, data lakes)
- **`edge_gateway_service`** = Boundary infrastructure (ALB, API Gateway, CloudFront, WAF, firewalls)
- **`product_component`** = Your custom code/applications that run ON the infrastructure above
- **`software_deployment_pattern`** = How all these pieces connect together for a specific product

## Example Workflow

Imagine you're building a new microservice called "Staff Pay Definition Service" that will run on Kubernetes, use SNS/SQS for messaging, DynamoDB for data storage, and deploy via CDK to AWS:

1. **Work with the AI Draftsman** to create YAML files describing your architecture:
   - A **`product_component`** for your custom microservice code (e.g., "Staff Pay Definition Service")
   - A **`runtime_service`** for Kubernetes (where your service will run)
   - A **`runtime_service`** (messaging, PaaS) for AWS SNS topics and SQS queues
     - Specify FIFO vs standard queue types and topic/queue names
     - *Don't* specify visibility timeouts or dead-letter queue configs
   - A **`data_store_service`** (database, PaaS) for a DynamoDB table
     - Specify table name and purpose
     - *Don't* specify attribute schemas or secondary indexes
   - A **`software_deployment_pattern`** describing how these components deploy together
     - CDK for infrastructure-as-code
     - Kubernetes manifests for service deployment
   - The Draftsman ensures you reference approved Technology Components from the catalog

2. **Open a PR** with the YAML files created by the Draftsman

3. **Architecture review** happens via PR review
   - If you proposed something non-standard (like self-hosted RabbitMQ when the approved standard is Confluent Kafka or AWS SQS), reviewers can identify and discuss it
   - Compliance requirements are checked automatically

4. **Once approved**, this catalog entry becomes the architectural source of truth
   - Other AI agents can reference it to understand what exists
   - Implementation work stays within the approved architectural boundaries
   - Changes to architecture require updating the catalog (keeping it accurate)
   - The structured data can potentially drive deployment automation and CI/CD pipelines

This keeps architecture decisions visible, governed, queryable, and version-controlled in Git.

## Start With This Prompt

Copy this into your preferred AI tool:

```text
I want a Draftsman session for Sofa King DRAFT Workspace.

Use the DRAFT workspace repository https://github.com/BeerCanLabs/sk-sofaking-drafting-table.
Read and follow the repository bootstrap instructions, starting with AGENTS.md.
Use the vendored framework copy under .draft/framework/ as the approved source
of Draftsman instructions, schemas, docs, and tools. Assume the Draftsman role
defined by the repo instead of inventing your own workflow.

Inspect the existing workspace, catalog, and configurations first. Then start
the Draftsman session with the next useful action. Ask only the first question
needed to move the work forward.

If you cannot connect to the repo, inspect its files, or write changes back to
it, stop and tell the user exactly what they need to enable so the Draftsman
session can work end to end.
```

## Commands

These commands work in any AI IDE. Claude Code users see them in `/` autocomplete;
all other tools respond when you type them in chat.

| Command | What it does |
|---|---|
| `/draftsman [intent]` | Start a Draftsman session. Provide an optional intent (e.g. `/draftsman add the org-unit service`) or leave it blank to be prompted. |
| `/draft-session [topic]` | Start or resume a guided Drafting Session for a specific system or product area. |
| `/draft-validate` | Run the DRAFT validator and get a grouped, actionable report of any errors or warnings. |
| `/draft-updateframework` | Check for DRAFT framework updates and guide a safe upgrade (Draft Admin only). |
| `/draft-triage [filter]` | Pull open GitHub issues and work through selected backlog items. |
| `/draft-review [scope]` | Expert consultant review of the DRAFT framework (Framework maintainers only). |

If your AI tool does not respond to slash commands, phrase the request naturally —
*"Act as a Draftsman and help me draft the org-unit service."* — and any
AI reading `AGENTS.md` will pick up the Draftsman role automatically.

## AI-First Bootstrap

`AGENTS.md` is the canonical bootstrap for AI assistants connected to this repo.
`CLAUDE.md`, `GEMINI.md`, `.github/copilot-instructions.md`, and `llms.txt`
point tool-specific assistants back to that contract. All of them direct agents
to read the vendored framework instructions under `.draft/framework/`, write
Sofa King architecture content only under `catalog/` or company-owned
`configurations/`, and leave `.draft/framework/**` plus `.draft/framework.lock`
to explicit framework refresh workflows.

Requirement Groups are resolved from the vendored framework first, optional third-party provider packs second, and Sofa King-owned configurations last. Sofa King activates the workspace-mode groups it architects against in `.draft/workspace.yaml`; the current active company group is `requirement-group.sofaking-hearthstone`.

Capabilities are first-class configuration objects. The framework ships the shared capability vocabulary with empty implementation lists, and Sofa King maps those capabilities to approved Technology Components through `configurations/object-patches/` or Sofa King-owned capability files in `configurations/capabilities/`.

The workspace includes the optional DRAFT framework update workflow at
`.github/workflows/draft-framework-update.yml`. It checks for newer DRAFT
Framework version tags and opens an update pull request. If validation fails,
the pull request is left open as blocked so the catalog can be repaired on the
update branch.

## Layout

- `catalog/` contains Sofa King architecture inventory.
- `configurations/` contains Sofa King-owned capabilities, Requirement Groups, domains, and framework object patches.
- `.draft/framework/` contains the vendored DRAFT framework copy used by local tools and GitHub Actions.
- `.draft/providers/` is reserved for optional third-party control packs.
- `.github/workflows/draft-framework-update.yml` checks for newer DRAFT Framework releases and prepares update PRs.
- `docs/index.html` is generated from the workspace catalog for GitHub Pages.
- `docs/assets/` contains generated browser data plus copied browser UI assets.
- `docs/user-manual.html` is generated from the framework-owned Markdown manual.
