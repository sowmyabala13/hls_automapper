# Components

Each component lists its purpose, inputs, outputs and requirements. Table and column names here are suggestions; adapt them to your naming standards, but keep the information each table carries.

---

## C1. Landing zone and file registry

**Purpose:** a predictable place for files to arrive, and configuration that tells the rest of the system what each file is.

**Volume layout requirements**
- Data files live under one ingestion root that Auto Loader watches, organized at least by source and file type, so each file's source and type can be read from its path.
- Spec documents, data dictionaries and anything that is not data live **outside** the ingestion root. Auto Loader reads every file under the root it watches, and a single non-data file can break the stream. Copies of spec documents also go in the repo's `reference/` folder so Genie Code can read them with `@`.
- Consider `processed/` and `rejected/` areas outside the ingestion root for files that have been handled or failed.
- Auto Loader skips files whose names start with `_` or `.` without raising an error. Flag any delivered file named like that, and rename it on landing (record both names).

**File registry (configuration table)**

One row per file type the system knows how to read. Keep the rows in a seed file in the bundle as the source of truth, and load them into the table with a job that validates the seed before writing (no duplicate keys, join keys present exactly for supplemental rows, column lists consistent with the expected field count). Suggested columns:

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
| `quote_char` | Quote character, or NULL if values are never quoted. State it explicitly; don't leave it to a parser default |
| `expected_header_cols`, `expected_fields_per_row` | Column count of the header and field count of each data row, measured from the delivered files. They can differ (trailing delimiter, unlabeled or missing fields) |
| `column_names` | Optional. An explicit column list to use instead of the file's header when the header doesn't match the data. NULL means use the file header |
| `unconfirmed_columns` | Optional. Columns whose position couldn't be confirmed from the data. Profiling flags them; mapping treats them as review-only |
| `parse_note` | Known quirks of the delivered layout, in plain words |
| `delivered_file_name`, `spec_file_name` | The name the file arrived with, and the name in the source's spec, when they differ |

Genie Code should propose the registry rows from the raw file inventory and the spec documents, and a person should confirm them. Never register a header that is known not to match the data: propose a corrected `column_names` list, mark what can't be confirmed in `unconfirmed_columns`, and ask the source for the real layout.

---

## C2. Bronze ingestion

**Purpose:** ingest every file, as it arrived, without imposing a schema.

**Inputs:** files under the ingestion root; the file registry.
**Output:** a bronze streaming table (or one per file type, if you prefer; justify the choice in the build notes).

**Requirements**
- Spark Declarative Pipelines with the modern Python API (`from pyspark import pipelines as dp`), defined in the bundle.
- Auto Loader, with format, delimiter, quoting, header handling and column names driven by the file registry rather than hard-coded.
- Read files line by line (or with a streaming reader). Never read a whole file into one value: real deliveries can be very large.
- No type inference: every source value is kept as a string, so leading zeros and unusual formats survive.
- All source columns of a row stored together in one VARIANT column.
- Metadata columns on every row: source, file type, file path, file name (decoded, not URL-encoded), file modification time, ingestion time, and a row identifier. Use the file metadata column Auto Loader provides for file details. If the row identifier is a hash of the file and row content, identical rows in one file share it; document that, and don't use it as a primary key.
- **Field count per row.** Measure each row's field count from the raw line and compare it with the registry's `expected_fields_per_row`, with a data quality expectation that flags mismatches and keeps the rows. Don't rely on rescued data for this: a permissive CSV reader can fill short rows with nulls and drop extra fields without any error, so layout drift in a later delivery would pass unnoticed.
- When the registry's `quote_char` is NULL, flag any line that contains a quote character. If a quote character is ever set and the parser can't honour it, fail rather than mis-parse.
- Header lines never land as data rows, including repeated headers. Positions that only exist because of a trailing delimiter are named in `column_names` and expected to be empty.
- Rescued data captured for malformed rows and unexpected values.
- Delta best practices for the table properties, including variant shredding if the workspace supports it.
- Bronze is append-only by construction: only the pipeline's own append flows write to it, and nothing updates or deletes rows. Don't set `delta.appendOnly = true` while full refreshes are still needed (it blocks the truncate a full refresh does). Set it, and block full refreshes of the table, once the layout is confirmed.
- Prove the field-count check with a bad-row test before relying on it (see phase 3 in `03_build_plan.md`).

---

## C3. Source profiling and exploration

**Purpose:** describe each source column so mapping has evidence to work from, and document how the files relate to each other.

**Output:** a source profile table, plus an exploration markdown.

**Profile requirements** (one row per source, file type and column)
- Column name and position, total rows, null and blank counts, distinct count, min and max, average and max length. Take positions from the registry; a VARIANT column keeps values by name, not by position.
- Registry context on each row: whether the column is a placeholder, whether its position is unconfirmed (`unconfirmed_columns`), whether it appears in the delivered header, and the file type's parse note. Columns in the delivered header with no field in the data get a row too, marked as such.
- Min and max compared by type for dates and amounts (text order is wrong for formats such as `MM/dd/yyyy`).
- Most frequent values and a few sample values.
- Detected value pattern (date formats that actually parse, fixed-length numeric codes with leading zeros, identifiers, amounts, flags, free text), with the share of values that match it. Columns with no non-blank values get an explicit `all_blank` pattern. Check each date column against the format the spec gives for its file type.
- A flag and a class for columns that look sensitive, matched by whole-word column-name rules plus value checks. Suggested classes: direct identifiers (SSNs, member, subscriber, patient, policy and claim IDs, tax IDs, license and DEA numbers), personal details (names, street addresses, dates of birth and death, phone, fax, email), and geography below state level (city, ZIP, county). Decide with the source whether service dates count too. Provider identifiers that don't identify a patient (such as NPI) may stay unmasked; record the decision.
- **Masking for sensitive columns:** top values and samples keep only each value's shape (digits as `9`, letters as `X`), min and max are NULL, and counts, lengths and patterns are kept. Don't hash small-space identifiers such as SSNs: every value can be recovered by trying them all. Keep raw values out of the job's output and logs as well as the table.
- A profile run identifier, so mapping can target one run. Append each run; don't overwrite earlier ones.
- Before writing, check that every bronze column has exactly one row and that no masked column holds a raw value.

**Exploration requirements**
- Candidate keys within each file type, and candidate join keys between file types. With small samples many columns are unique by chance: rank identifier-like columns first, and say so.
- Classify every candidate cross-file pair as a key join, a duplicated column (the same field carried in two files), or a coincidental overlap (amounts, codes, the same attribute in another role). Document key joins and duplicates in detail; give counts for coincidences.
- Values that are identical where real data would differ (test or mock data artifacts). Note them, and don't use them as mapping evidence.
- Overlapping or duplicated columns across file types, and which source should be authoritative when both carry the same field.
- Anything in the data that disagrees with the spec documents.
- Report column names, counts and match rates only. No raw identifiers in the exploration notes.

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
| `mapping_kind` | `copy` (value used as-is), `expression` (transformed), or `unmapped`. These are defaults: if the mapping rules in `reference/` require different labels or output forms, use those instead |
| `expression` | The SQL expression that produces the target value from the source column. Required when `mapping_kind = expression` |
| `confidence` | 0.0 to 1.0 |
| `rationale` | One sentence on why |
| `review_status` | For example `auto_accept`, `review`, `unmapped` |

**Requirements**
- Every source column gets exactly one proposal, including unmapped ones.
- Follow the mapping rules in `reference/` exactly when they exist (naming conventions, renames, date formats per file type, required output forms and labels). Rules from the source spec override inference and override the pack's default labels.
- Columns listed in the registry's `unconfirmed_columns`, and placeholder columns (names starting with `_`), get proposals marked for review only; never auto-accept them.
- Map from one profile run: record the `profile_run_id` the proposals are based on.
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
