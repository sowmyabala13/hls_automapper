# 02. Preflight and environment (phases 0 and 1)

**When:** after orient, once its open points are answered.
**Builds:** `00_preflight.md`; a schema and volume as bundle resources, deployed to dev; `00_environment.md`.

## Prompt

```
Phases 0 and 1 of @<REPO>/docs/03_build_plan.md.
Read docs/04_lessons_learned.md first.

Phase 0: check every item in docs/05_prerequisites.md that can be checked from the
workspace, including access to the target model at <TARGET_CATALOG>.<TARGET_SCHEMA>
and lineage and query-history access on it. Defer checks that can only be proven
later to a named phase instead of marking them passed. Write
project/docs/analysis/00_preflight.md.

Phase 1: use catalog <CATALOG>. Check whether I can create a schema in it. If I can,
define schema <SCHEMA> and volume <VOLUME> as resources in the bundle at
<REPO>/project; if I can't, use the existing schema <FALLBACK_SCHEMA> and define only
the volume. Follow the dev target naming rule in phase 1.

Show me the preflight result and the bundle changes before deploying. After my OK,
deploy to dev, write, read back and delete a test file in the volume, record the
actual deployed names in project/docs/analysis/00_environment.md, check both phases'
"Done when", and commit locally only.
```

## Check before you approve

- Every gap in the preflight has an owner or a named later phase.
- The schema and volume are bundle resources, not created by hand.
- If Genie Code proposes changing the dev target naming, it says what that trades away.

**Next:** `03_inventory_layout.md`.
