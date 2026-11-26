<!-- What does this PR add or change? One or two sentences. -->

## Catalog checklist

See [CONTRIBUTING.md](../CONTRIBUTING.md) for the full guide.

- [ ] Every new or changed catalog file is a **single YAML object with a recognized `type:`** (a missing `type:` is silently skipped, a wrong one fails — and markdown under `catalog/` is never read)
- [ ] The file lives in the right area: `catalog/engineering/<team-id>/`, `catalog/shared-services/<type-folder>/`, or `catalog/governance/<type-folder>/`
- [ ] `uid` was **generated** (Crockford Base32 via `uid_utils`), not invented
- [ ] `owner.team`, `deploymentTarget`, and `businessContext.pillar` use approved vocabulary values (pillar confirmed by the owning team, prefix included)
- [ ] `python .draft/framework/tools/validate.py --workspace .` runs locally — my file is **validated as its declared type** (not silently skipped) with no new warnings
- [ ] No runbooks/manuals in `catalog/` and no regenerated `docs/` output committed
