# 05. Profiling and exploration (phase 4)

**When:** after phase 3's done check passes.
**Builds:** the profile job and source profile table in the bundle, and `04_bronze_exploration.md`.

## Prompt

```
Phase 4 of @<REPO>/docs/03_build_plan.md.
Read project/docs/analysis/00_preflight.md through 03_bronze_ingestion.md and
docs/04_lessons_learned.md first, and use @<REPO>/reference for the specs and rules.

Design profiling to meet C3 in docs/02_components.md, reading the bronze table and
the file registry (including parse notes and unconfirmed columns). Define the
profile table and the job that fills it as bundle resources. Flag sensitive columns
and mask their top and sample values; keep raw values out of the job's output too.
Tell me which sensitivity decisions you need from me.

Show me the profile design before running it. After my OK, deploy to dev and run
the job. Then explore: candidate keys per file type, and every candidate join
between file types classified as a key join, a duplicated column, or a coincidence.
Note values that look like test-data artifacts and anything that disagrees with the
spec. Write 04_bronze_exploration.md with column names, counts and match rates only,
record the profile run ID for phase 6, check phase 4's "Done when", and commit
locally only.
```

## Check before you approve

- Masking keeps only the shape of sensitive values, and small identifiers such as SSNs are not hashed.
- Sensitive-column name rules match whole words (review which columns each rule catches).
- Columns with no values are labelled as such, and columns with unconfirmed positions are flagged.

**Next:** phases 5 to 9 follow `docs/03_build_plan.md` directly. Prompts for them will be added once those phases have been tested.
