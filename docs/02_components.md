# Components

Each component lists its purpose, inputs, outputs and requirements. Table and column names here are suggestions; adapt them to your naming standards, but keep the information each table carries.

---

## C1. Landing zone and file registry

**Purpose:** a predictable place for files to arrive, and configuration that tells the rest of the system what each file is.

**Volume layout requirements**
- Data files live under one ingestion root that Auto Loader watches, organized at least by source and file type, so each file's source and type can be read from its path.
- Spec documents, data dictionaries and anything that is not data live **outside** the ingestion root. Auto Loader reads every file under the root it watches, and a single non-data file can break the stream. Copies of spec documents also go in the repo's `reference/` folder so Genie Code can read them with `@`.
- Consider `processed/` and `rejected/` areas outside the ingestion root for files that have been handled or failed.

**File registry (configuration table)**

One row per file type the system knows how to read. Suggested columns:

| Column | Meaning |
| --- | --- |
| `source_id` | The sender (client, vendor, system) |
| `file_type` | Logical file type, for example the kind of record it holds |
| `path_pattern` | How to recognize this file type from its path or name |
| `format`, `delimiter`, `has_header`, `encoding` | How to parse it |
| `target_table` | Which target table its columns map to |
| `role` | `primary` (produces rows in the target table) or `supplemental` (adds columns to rows produced by another file type) |
| `join_keys` | For supplemental files: the keys that join it to the primary file's rows |
| `is_active` | Whether it is in use |

Genie Code should propose the registry rows from the raw file inventory and the spec documents, and a person should confirm them.

---

## C2. Bronze ingestion

**Purpose:** ingest every file, as it arrived, without imposing a schema.

**Inputs:** files under the ingestion root; the file registry.
**Output:** a bronze streaming table (or one per file type, if you prefer; justify the choice in the build notes).

**Requirements**
- Spark Declarative Pipelines with the modern Python API (`from pyspark import pipelines as dp`), defined in the bundle.
- Auto Loader, with format, delimiter and header handling driven by the file registry rather than hard-coded.
- No type inference: every source value is kept as a string, so leading zeros and unusual formats survive.
- All source columns of a row stored together in one VARIANT column.
- Metadata columns on every row: source, file type, file path, file name, file modification time, ingestion time, and a row identifier. Use the file metadata column Auto Loader provides for file details.
- Rescued data captured for malformed rows.
- Delta best practices for the table properties, including variant shredding if the workspace supports it.
- Bronze is append-only and never modified.

---

## C3. Source profiling and exploration

**Purpose:** describe each source column so mapping has evidence to work from, and document how the files relate to each other.

**Output:** a source profile table, plus an exploration markdown.

**Profile requirements** (one row per source, file type and column)
- Column name and position, total rows, null and blank counts, distinct count, min and max, average and max length.
- Most frequent values and a few sample values.
- Detected value pattern (date formats, fixed-length numeric codes, identifiers, amounts, flags).
- A flag for columns that look sensitive (identifiers such as SSNs).
- A profile run identifier, so mapping can target one run.

**Exploration requirements**
- Candidate keys within each file type, and candidate join keys between file types.
- Overlapping or duplicated columns across file types.
- Anything in the data that disagrees with the spec documents.

---

## C4. Target model analysis

**Purpose:** understand what the mappings must produce.

**Output:** a target model markdown.

**Requirements**
- For each target table: columns, types, comments, keys, and constraints.
- How the tables are queried (common filters, joins and aggregations) and their upstream and downstream lineage.
- Gaps: target columns with no comment, ambiguous column names, types that need a format decision. Propose comments for a person to approve; do not alter the target tables.

---

## C5. Genie Code mapping

**Purpose:** propose a mapping for every source column.

**Inputs:** the source profile, sample bronze rows, the target model (column comments), the spec documents and mapping rules in `reference/`, earlier approved mappings, the file registry.
**Output:** rows in a proposed mappings table.

**Proposed mapping columns** (suggested)

| Column | Meaning |
| --- | --- |
| `mapping_run_id`, `source_id`, `file_type`, `source_column` | What is being mapped |
| `target_table`, `target_column` | Where it goes (NULL if unmapped) |
| `mapping_kind` | `copy` (value used as-is), `expression` (transformed), or `unmapped` |
| `expression` | The SQL expression that produces the target value from the source column. Required when `mapping_kind = expression` |
| `confidence` | 0.0 to 1.0 |
| `rationale` | One sentence on why |
| `review_status` | For example `auto_accept`, `review`, `unmapped` |

**Requirements**
- Every source column gets exactly one proposal, including unmapped ones.
- Follow the mapping rules in `reference/` exactly when they exist (naming conventions, renames, date formats per file type, required output forms). Rules from the source spec override inference.
- Use earlier approved mappings from other sources to inform proposals and confidence.
- Write only to the proposed mappings table. Never write to the target model or the approved mappings.
- Start interactively in a Genie Code session. Once the prompt is stable, run it as a Genie Code task in a job (see `04_lessons_learned.md` for how that task is configured).

---

## C6. Review

**Purpose:** a person decides what reaches the pipeline.

**Output:** rows in an approved mappings table, versioned.

**Approved mapping columns:** the proposal columns, plus `approved_by`, `approved_at`, `version`, `is_current`.

**Options**
- **Lightweight (default):** review the proposals in a notebook or dashboard, edit where needed, and promote them to the approved table with a reviewed SQL statement.
- **Optional app:** a Databricks App with screens to upload files, trigger processing, review proposals side by side with source samples, and approve. Adds the prerequisites listed in `05_prerequisites.md`.

**Requirements**
- At most one current approved mapping per source, file type and target column.
- Approvals are never overwritten: a change creates a new version and retires the old one.

---

## C7. Metadata-driven transformation pipeline

**Purpose:** apply approved mappings to bronze and produce the target tables.

**Inputs:** bronze, approved mappings (current only), the file registry.
**Output:** one table per target table, in a dev schema during the build.

**Requirements**
- Spark Declarative Pipelines, modern Python API, defined in the bundle.
- One output per target table, built from all sources and file types that map to it.
- Supplemental file types are joined to their primary rows on the registry's join keys.
- Every expression is checked before it is used: it parses, references only columns that exist, and does not repeat an alias.
- Duplicate approved mappings for the same target column are rejected or deduplicated deterministically, never silently unioned.
- Data quality expectations on keys and required columns, with failing rows kept for review rather than dropped silently.
- Lineage from each target row back to its source file.

---

## C8. Validation

**Purpose:** show the result is correct.

**Requirements**
- Row counts from bronze to each target table, per source and file type.
- Null and type checks on target columns.
- Where known-correct mappings exist, compare the proposals against them and report accuracy per file type.
