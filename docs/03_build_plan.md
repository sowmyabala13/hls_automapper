# Build Plan

Work through the phases in order, each in a Genie Code session in agent mode. This plan deliberately contains **no finished code**: each phase states the goal, the requirements, what it must produce, and how to tell it is done. Genie Code writes the code in your workspace, against your data.

**Every phase:**
- Starts by reading the previous phases' markdowns in `project/docs/analysis/`. Do not rely on chat history.
- Ends with a markdown in `project/docs/analysis/` recording what was built, decisions made and open questions, then a commit to the feature branch.
- Defines any schema, volume, table, pipeline or job as a bundle resource and deploys to the dev target.
- Shows the design and waits for approval before creating or deploying anything.
- Commits to the feature branch locally. Push only when the user asks.
- Writes its phase notes in one layout: what was built, decisions made, the done check and its result, open items, and what carries into the next phase.

Component details are in `02_components.md`. Known pitfalls are in `04_lessons_learned.md`; read it before phases 2, 3, 4, 6 and 8.

---

## Phase 0: Preflight

**Goal:** confirm the prerequisites before building anything.
**Requirements:**
- Check every item in `05_prerequisites.md` that can be checked from the workspace (grants, serverless, bundle deploy, Git, target model access). Report what is missing with the exact grant or setting needed.
- Check lineage and query-history access on the target model; phase 5 needs both.
- A check that can only be proven later is deferred to a named phase, not marked as passed. For example: serverless pipelines and variant shredding (phase 3), Genie Code automations (phase 6).
**Produces:** `00_preflight.md`
**Done when:** every required prerequisite is confirmed, or each gap has an owner.

## Phase 1: Environment

**Goal:** a schema and volume for the new ingestion, defined in the bundle.
**Requirements:**
- Use the catalog you are given. Check whether you can create a schema in it.
- If you can, define the schema and the volume as bundle resources.
- If you cannot, use the existing schema you are given and define only the volume.
- Dev target naming: keep development mode on the dev target by default, so each person who deploys gets their own prefixed schema and resources. Use exact (shared) names only if the user asks for them, and say what that trades away (everyone who deploys the target shares one schema).
- Deploy to the dev target.
**Produces:** bundle resources, deployed; `00_environment.md` recording what was created, where, and why, including the actual deployed names.
**Done when:** the volume exists, and a test file can be written to it, read back and deleted.

## Phase 2: Raw file inventory and landing layout

**Goal:** know exactly what the source sent, and put it in the right place.
**Requirements:**
- From the attached files (or header screenshots) and the user's description: how many files, which are data and which are spec documents, format, delimiter, header row, encoding, quoting, size, row and column counts. Measure these from the files themselves, not from the description.
- For every file, compare the header's column count with the number of fields in each data row.
- When they differ, check each column's values against its header name (codes, identifiers, amounts, flags, dates) before choosing a fix. A net difference of one can hide several inserted and missing fields; don't assume a single inserted field.
- Never load a header that is known to be misaligned. Propose a corrected column list for a person to confirm, and mark any positions that can't be confirmed from the data.
- Flag any file whose name starts with `_` or `.`: Auto Loader skips those files without an error.
- Record each file's delivered name and its name in the spec separately when they differ.
- Recommend a volume folder layout that meets the C1 requirements.
- Propose the file registry rows (C1) for a person to confirm.
- Copy spec documents into `reference/`.
**Produces:** `01_raw_inventory.md`, `02_landing_layout.md`, a registry seed file in the bundle (the source of truth) with the job that loads it, file registry table populated, files uploaded into the agreed layout.
**Done when:** every file is in its place and has a matching registry row, and each file's header count and fields per row match its registry row.

## Phase 3: Bronze ingestion

**Goal:** every data file streamed into bronze.
**Requirements:** C2.
**Order:**
1. Define the pipeline as a bundle resource, with its source folder inside the bundle, and deploy it to the dev target.
2. Then open that pipeline in the pipeline editor to write and run the pipeline code. Opening the pipeline editor before the pipeline exists creates a placeholder pipeline outside the bundle, and the editor may only be able to edit that placeholder's folder.
3. After the pipeline runs, go back to the bundle's Git folder for the done check, the phase notes and the commit.

**Bad-row test:** land a temporary file with one row that has an extra field and one that is missing a field. Confirm both rows are flagged by the field-count check and kept, not dropped. Then remove the file and full-refresh bronze.
**Produces:** bronze pipeline in the bundle, bronze table(s), `03_bronze_ingestion.md` with table properties chosen and why, the bad-row test result, and when the append-only setting is turned on.
**Done when:** row counts in bronze match the inventory for every file, every row's field count matches the registry, no test rows remain, and no row has rescued data that was not expected.

## Phase 4: Profiling and exploration

**Goal:** evidence for mapping, and an understanding of how the files relate.
**Requirements:** C3. Reference the spec documents in `reference/` with `@`.
- Read the registry's parse notes and its list of columns whose position is unconfirmed, and carry both into the profile.
- Mask the top and sample values of sensitive columns (C3), and keep raw values out of the job's output as well as the table.
- Separate real joins from coincidental overlaps: classify each candidate pair as a key join, a duplicated column, or a coincidence. Small samples make many columns look unique, and many values match by chance.

**Produces:** source profile table, `04_bronze_exploration.md`.
**Done when:** every bronze column has a profile row, candidate keys and joins are documented, no masked column holds a raw value, and the profile run ID that phase 6 should use is recorded.

## Phase 5: Target model analysis

**Goal:** understand what the mappings must produce.
**Requirements:** C4.
**Produces:** `05_target_model_analysis.md`, including proposed column comments for any target columns that lack them.
**Done when:** every target column has a documented meaning and expected format, either from its comment or from the analysis.

## Phase 6: Mapping

**Goal:** a proposal for every source column.
**Requirements:** C5. Create the proposed and approved mapping tables (C5, C6) as part of this phase.
**Steps:**
1. Draft the mapping prompt and run it interactively for one file type. Review the output with the user.
2. Refine the prompt and run it for every file type.
3. Optional: configure the prompt as a Genie Code task in a job, after the profiling task.
**Produces:** proposed mappings, `06_mapping_plan.md` with the source-to-target mappings summarized per target table, and a diagram of source files to target tables.
**Done when:** every source column has exactly one proposal, and every proposal with `mapping_kind = expression` has an expression.

## Phase 7: Review

**Goal:** a person approves what reaches the pipeline.
**Requirements:** C6. Use the lightweight option unless the optional app was planned.
**Produces:** approved mappings, `07_review_log.md` recording what was changed and why.
**Done when:** there is exactly one current approved mapping per source, file type and target column.

## Phase 8: Transformation pipeline

**Goal:** target tables produced from approved mappings.
**Requirements:** C7. Write to a dev schema, never to production target tables.
**Produces:** transformation pipeline in the bundle, target tables in dev, `08_pipeline_build.md`.
**Done when:** the pipeline runs end to end and every target table has rows.

## Phase 9: Validation

**Goal:** show the output is correct.
**Requirements:** C8.
**Produces:** `09_validation.md`.
**Done when:** counts reconcile, checks pass or failures are explained, and accuracy is reported where known-correct mappings exist.

---

## Optional extensions

- **Orchestration:** a job that runs profiling, the Genie Code mapping task and the transformation pipeline when new files arrive.
- **Review app:** see C6.
- **CI/CD:** deploy the bundle to test and prod targets from the main branch.
