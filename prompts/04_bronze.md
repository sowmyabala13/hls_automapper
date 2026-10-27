# 04. Bronze ingestion (phase 3)

**When:** after phase 2's done check passes.
**Builds:** the bronze pipeline in the bundle, the bronze table, and `03_bronze_ingestion.md`.

The pipeline code is written in the pipeline editor. Expect Genie Code to hand over to that page partway through, and to come back to the bundle's Git folder to finish.

## Prompt

```
Phase 3 of @<REPO>/docs/03_build_plan.md.
Read project/docs/analysis/00_preflight.md through 02_landing_layout.md and
docs/04_lessons_learned.md first.

First settle the checks deferred to this phase in 00_preflight.md.
Then design bronze to meet C2 in docs/02_components.md: a Spark Declarative
Pipeline with the modern Python API, Auto Loader on the ingestion root, parsing
driven by the file registry, every value kept as a string in a VARIANT column, file
metadata on every row, a per-row field count checked against the registry, rescued
data captured, and table properties chosen per best practice. Decide one bronze
table or one per file type, and justify it.

Show me the design before building. After my OK, follow phase 3's order: define
the pipeline as a bundle resource and deploy it, then open it in the pipeline editor
to write and run the code. Run the bad-row test, clean up, then check phase 3's
"Done when" (expected rows: <ROWS PER FILE TYPE>), write 03_bronze_ingestion.md
from the bundle's Git folder, and commit locally only.
```

## Check before you approve

- The field count comes from the raw line, not from rescued data.
- Files are read line by line, and the quoting assumption is stated.
- The table is append-only by construction, with the `delta.appendOnly` setting deferred until full refreshes are no longer needed.
- No `input_file_name()`, no `import dlt`, no type inference.

**Next:** `05_profiling.md`.
