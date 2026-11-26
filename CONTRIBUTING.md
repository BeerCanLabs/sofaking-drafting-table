# Contributing — Getting a Catalog PR Validated and Merged

This repo is a **schema-validated architecture model**, not a docs site. Every
PR is checked by the DRAFT validator in CI and reviewed by humans against the
same rules. This page is the short version of what both expect.

## The one rule that matters

> Every file under `catalog/` must be a **single YAML object** with a `type:`
> field the framework recognizes (`software_deployment_pattern`,
> `runtime_service`, `data_store_service`, …).

**The validator silently skips any YAML object with no `type:` field.** A file
whose top-level object has a missing or empty `type:` "passes" CI with zero
checks, never appears in the architecture browser, and bypasses every
requirement check. A *recognized-but-wrong* `type:` is the opposite problem — a
typo like `type: runtime_servce` fails loudly with `no schema found for type
'runtime_servce'`, so it won't slip through silently. The trap is a *missing*
`type:`. Markdown files under `catalog/` are never read at all.

A green check does **not** by itself mean your entry is valid — it may mean it
was ignored. Step 7 below shows how to tell the difference.

## Step-by-step

### 1. Start from a template, not from scratch

Copy the matching template from [`.draft/templates/`](.draft/templates/)
(for example
[`software-deployment-pattern.yaml.tmpl`](.draft/templates/software-deployment-pattern.yaml.tmpl))
or model your file on a merged entry such as
[`sdp-cca-automation-platform.yaml`](catalog/governance/reference-architectures/sdp-cca-automation-platform.yaml).
Don't invent fields — the allowed fields per object type are defined in
[`.draft/framework/schemas/`](.draft/framework/schemas/). If a template and a
schema ever disagree, **the schema wins**: the schemas are the contract the
validator actually enforces, so cross-check field names against them (or
against a recently merged entry) before filling anything in.

### 2. Put one YAML file in the right place

| Content | Location |
|---|---|
| Team-owned systems (deployment patterns, product components) | `catalog/engineering/<team-id>/` |
| Shared infrastructure (runtimes, data stores, network services) | `catalog/shared-services/<type-folder>/` |
| Reference architectures, decision records, drafting sessions | `catalog/governance/<type-folder>/` |

No per-product folders. No markdown sidecars. One object, one YAML file.

### 3. Fill in the required header

```yaml
schemaVersion: '1.0'
uid: 01KQQ5JT1J-BRF4          # generated — see below
type: software_deployment_pattern
name: My Product — Pattern
catalogStatus: incomplete      # stub | incomplete | complete
lifecycleStatus: candidate     # candidate | preferred | existing-only | deprecated | retired
```

UIDs are generated Crockford Base32 (`XXXXXXXXXX-XXXX`) — never made up:

```bash
python -c "import sys; sys.path.insert(0, '.draft/framework/tools'); from uid_utils import generate_uid; print(generate_uid())"
```

### 4. Use approved vocabulary values only

- `owner.team` → an id from [`configurations/vocabulary/teams.yaml`](configurations/vocabulary/teams.yaml)
- `deploymentTarget` → an id from [`configurations/vocabulary/deployment-targets.yaml`](configurations/vocabulary/deployment-targets.yaml)
- `businessContext.pillar` → a full id from `.draft/workspace.yaml`, prefix
  included (e.g. `business-pillar.enterprise-enablement`), **confirmed by the
  owning team** — never guessed from the name

If the true answer isn't on a list, that's a *non-standard value*: say so in
the PR and submit a vocabulary proposal. Don't freehand it.

### 5. Answer the standard architecture questions

Deployment patterns are checked against the SoftwareDeploymentPattern
RequirementGroup: availability, data classification, failure domain, external
dependencies, inter-service connections, deployment targets, and pattern
conformance (`followsReferenceArchitecture`, or a `noApplicablePattern`
decision record). For conformant deployments, set these directly as fields on
the SDP object — the validator will pick them up without a `decisionRecords`
reference. Only use a `decisionRecords` entry when the deployment cannot meet
the standard requirement and needs to document why (non-conformance).

The validator's warnings name the exact key that satisfies each requirement —
read them. For a complete working example, see
[`sdp-cca-automation-platform.yaml`](catalog/governance/reference-architectures/sdp-cca-automation-platform.yaml).

### 6. Keep prose out of the catalog

- Architecture facts (integrations, storage, network zones) → structured YAML
  fields (`serviceGroups`, `externalInteractions`, `architectureNotes`)
- Setup guides, user manuals, runbooks → your product's own repo or Confluence
- Derived stats (service counts, connection counts) → computed by the tooling;
  don't hand-maintain them

### 7. Validate locally — and read the output

```bash
python -m pip install pyyaml
python .draft/framework/tools/validate.py --workspace .
```

Confirm two things before you push:

1. **Your file was actually validated, not silently skipped.** The validator
   prints `PASS <file>` for *every* file it doesn't fail — including files it
   skipped for having no `type:` — so a bare `PASS` alone proves nothing. Make
   sure your object declares a recognized `type:` (and a generated `uid:`); a
   missing `type:` is exactly what gets skipped, while a misspelled one fails
   loudly (`no schema found for type …`). For positive proof, regenerate the
   browser and confirm your entry appears — a skipped object never shows up:
   ```bash
   python .draft/framework/tools/generate_browser.py --workspace . --output docs/index.html
   ```
   The generated output is untracked (see `.gitignore`); regenerate to check
   your work, but leave the result out of your commit.
2. **No new warnings** compared to `main`.

CI runs this exact command on every PR
([`validate-catalog.yml`](.github/workflows/validate-catalog.yml)).

### 8. Open the PR

Reviewers check what the validator can't: that the content is true, that
vocabulary choices were confirmed by their owners, and that placeholders are
resolved. The architecture browser (`docs/index.html`) is regenerated
automatically after merge — don't commit regenerated browser output.

## The shortcut

Let the Draftsman do steps 1–6 for you. Any AI assistant connected to this
repo bootstraps from [`AGENTS.md`](AGENTS.md) and knows the schemas, paths,
and vocabularies:

- `/draft author` — interview-style authoring that writes schema-correct YAML
  in the right location
- `/draft validate` — runs the validator and explains every finding with fix
  advisory
