# nautilus_data: development workflow and layout

Use branch `feature/ayu_develop`. The actual upstream default branch and all fetched branch/tag identities are recorded in the maintenance sync evidence. Merge existing work; do not reset divergent branches or replace conflicting tags.

## Source synchronization

```sh
git fetch origin
git fetch upstream
git switch feature/ayu_develop
git merge origin/feature/ayu_develop
# Merge the actual default branch documented in the sync result.
```

This maintenance used namespaced remote tags before comparing tag identities. Missing upstream tags/branches were pushed to the fork; divergent branches were merged when conflict-free, and conflicting identities were retained for explicit reconciliation. The feature branch contains the default branch plus the maintenance changes. Source snapshot and architecture are documented in [ayu-develop.md](ayu-develop.md).

## Repository layout

- `docs/ayu-develop.md` and `docs/ayu-develop.html`: engineering guide with architecture, workflow, class and sequence diagrams and implementation links.
- `docs/development.md`: toolchain, layout and contributor workflow.
- `docs/validation.md`: exact executed commands, outcomes and limitations.

## Retired repository

Upstream decommissioned this repository in May2026. The maintenance branch follows the README-only default and does not reintroduce the deleted Python package or uv project. Compilation and package tests are not applicable. Use the main NautilusTrader repository's current testkit/catalog data workflow. Historical docs that mention the retired package do not describe the current branch's API.

## Verification and documentation

Use the commands and results in [validation.md](validation.md). Markdown is the editable source of the guide; HTML embeds rendered SVG diagrams for offline browsing. Keep actual source links valid after moving modules. Preserve existing cases/assertions while correcting incompatible APIs, genuine resource leaks or invalid fixtures. No additional test cases were requested or added.

## Historical reference

The [previous development guide](archive/development-before-maintenance.md) is retained for source history. Its retired APIs and earlier toolchains are not the current branch instructions.
