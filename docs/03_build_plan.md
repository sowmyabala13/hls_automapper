# Build Plan

Work through the phases in order, each in a Genie Code session in agent mode. This plan deliberately contains **no finished code**: each phase states the goal, the requirements, what it must produce, and how to tell it is done. Genie Code writes the code in your workspace, against your data.

**Every phase:**
- Starts by reading the previous phases' markdowns in `project/docs/analysis/`. Do not rely on chat history.
- Ends with a markdown in `project/docs/analysis/` recording what was built, decisions made and open questions, then a commit to the feature branch.
- Defines any schema, volume, table, pipeline or job as a bundle resource and deploys to the dev target.

Component details are in `02_components.md`. Known pitfalls are in `04_lessons_learned.md`; read it before phases 3, 6 and 8.

---

## Phase 0: Preflight

**Goal:** confirm the prerequisites before building anything.
**Requirements:** check every item in `05_prerequisites.md` that can be checked from the workspace (grants, serverless, bundle deploy, Git, target model access). Report what is missing with the exact grant or setting needed.
**Produces:** `00_preflight.md`
**Done when:** every required prerequisite is confirmed, or each gap has an owner.

## Phase 1: Environment

**Goal:** a schema and volume for the new ingestion, defined in the bundle.
**Requirements:**
- Use the catalog you are given. Check whether you can create a schema in it.
- If you can, define the schema and the volume as bundle resources.
- If you cannot, use the existing schema you are given and define only the volume.
- Deploy to the dev target.
**Produces:** bundle resources, deployed; `00_environment.md` recording what was created, where, and why.
**Done when:** the volume exists and you can write a test file to it.

## Phase 2: Raw file inventory and landing layout

**Goal:** know exactly what the source sent, and put it in the right place.
**Requirements:**
- From the attached files (or header screenshots) and the user's description: how many files, which are data and which are spec documents, format, delimiter, header row, encoding, size, row and column counts.
- Recommend a volume folder layout that meets the C1 requirements.
- Propose the file registry rows (C1) for a person to confirm.
- Copy spec documents into `reference/`.
**Produces:** `01_raw_inventory.md`, `02_landing_layout.md`, file registry table populated, files uploaded into the agreed layout.
**Done when:** every file is in its place and has a matching registry row.

## Phase 3: Bronze ingestion

**Goal:** every data file streamed into bronze.
**Requirements:** C2.
**Produces:** bronze pipeline in the bundle, bronze table(s), `03_bronze_ingestion.md` with table properties chosen and why.
**Done when:** row counts in bronze match the inventory for every file, and no row has rescued data that was not expected.

## Phase 4: Profiling and exploration

**Goal:** evidence for mapping, and an understanding of how the files relate.
**Requirements:** C3. Reference the spec documents in `reference/` with `@`.
**Produces:** source profile table, `04_bronze_exploration.md`.
**Done when:** every bronze column has a profile row, and candidate keys and joins are documented.

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
