# Lessons Learned

Problems hit in a previous build of this pattern, and how to avoid them. These are about the platform and the design, not about any particular dataset.

## Landing and ingestion

- **Non-data files in the ingestion root break the stream.** Diagram files were copied into a subfolder of the folder Auto Loader watched. Auto Loader tried to parse them as data, the inferred schema absorbed a bogus column, and every retry failed. Recovery needed the files removed, the pipeline stopped, and a full refresh to reset the schema. Keep the ingestion root for data files only.
- **New columns cancel and restart the update.** With schema evolution set to add new columns, a file with columns not seen before makes the update stop and a new update start. Anything that triggers the pipeline and waits for it (an app, a job) must follow the follow-up update rather than treat the first one as failed.
- **Use the file metadata column, not `input_file_name()`.** `input_file_name()` is not supported on serverless with Unity Catalog. Auto Loader exposes file path, name and modification time through its file metadata column.
- **A unique ID is not a row number.** `monotonically_increasing_id()` is unique but not sequential per file. If you need a row position within a file, derive it explicitly.
- **Infer nothing in bronze.** Keep every value as a string so codes with leading zeros and unusual date formats reach profiling intact.
- **A field-count mismatch doesn't tell you where the drift is.** A header one column shorter than the data rows looked like one inserted field. Checking each column's values against its name showed two extra fields in one place and a missing field in another, so a single placeholder column would have left a run of columns under the wrong names. Compare header count with fields per row, then check values column by column, and never load a header that is known to be misaligned.
- **Rescued data doesn't catch layout drift.** A permissive CSV reader fills short rows with nulls and drops extra fields without an error. Count fields per row from the raw line and check the count against the registry with a data quality expectation.
- **Files whose names start with `_` or `.` are skipped silently.** A test file named that way never reached bronze, and nothing reported it. A real delivery named like that would be lost the same way.
- **`delta.appendOnly = true` blocks full refresh.** A full refresh truncates the table, which the property forbids, so the refresh failed twice. Keep bronze append-only by construction during the build, and set the property (with full refresh blocked) once the layout is settled.
- **Don't read whole files into one value.** It works on small samples and fails on real claim files. Read line by line.
- **Hash-based row IDs repeat for identical rows.** A hash of file and row content is stable, but two identical lines in one file share it. Don't treat it as a primary key.

## Profiling

- **Match sensitive column names on whole words.** Substring rules flagged an ethnicity code as a city (`ETHNICITY` contains `CITY`) and a state code as a license number (`STATE_OF_LICENSE`). Review which columns each rule matches before running it.
- **Don't hash small-space identifiers.** Every 9-digit SSN can be hashed and compared in minutes, so a hash hides nothing. Mask to the value's shape instead.
- **Masking the table isn't enough.** Notebook output, job logs and exploration notes can copy raw values too.
- **Small samples mislead.** With ten rows per file, dozens of columns were unique and hundreds of column pairs shared values across files. Rank identifier-like columns first and classify each candidate join before trusting it.
- **Test data has artifacts real data won't.** Mock files reused the same values across roles (one provider for every role, subscriber always equal to member). Note them, and don't use them as mapping evidence.

## Target model

- **Column comments must live where the table is defined.** In the previous build, the target table's DDL with column comments was written as a SQL file, but the table was actually created by a pipeline as a materialized view. The pipeline's definition replaced the DDL, and the live table ended up with no column comments, so the mapper worked from names alone. If a pipeline creates the table, put the comments in the pipeline's table definition, then confirm they appear in Catalog Explorer.
- **Check that the target definition you give the mapper is the one the pipeline produces.** Two descriptions of the target model with different column names and types existed side by side. Pick one source of truth.

## Mapping

- **Do not store the alias inside the expression.** The mapper wrote expressions like `col AS target`, and the pipeline appended `AS target` again, producing a parse error. Store only the expression; let the pipeline add the alias. Or have the pipeline strip a trailing alias before adding its own.
- **Duplicate approvals break the output.** Approving the same mapping twice produced two columns with the same name, and the union failed. Enforce one current approved mapping per target column at approval time, and deduplicate in the pipeline as a safeguard.
- **Naive string replacement of column names is fragile.** Rewriting an expression by replacing the source column name with a VARIANT path also replaces that name where it appears inside other identifiers. Replace whole identifiers only.
- **Validate every expression before the pipeline uses it,** for example by planning a query that applies it to a sample of bronze.

## Genie Code as a job task

- **The task is `genie_task`, not a prompt field.** A Genie Code job task points at a Genie Code automation through `genie_task.configuration_id`. Writing the prompt directly into the task definition (for example as `genie_code_task.prompt`) is rejected by the Jobs API.
- **The automation is created separately, then referenced.** Create a dedicated automation per job task, with no schedule of its own, holding the prompt. Its returned resource name is the `configuration_id`.
- **Pass run values as task parameters.** Put `{{name}}` placeholders in the automation's prompt and supply values through the task's `parameters`, which can reference job parameters.
- **Deleting the job does not delete the automation.** Clean it up separately.

## Pipelines and jobs

- **A pipeline task's error is in the pipeline event log,** not in the job run output. The job only reports that the workload failed.
- **Job retries can exhaust repair.** A run whose retries were used up may not be repairable; start a new run.
- **Ordering matters.** If the transformation pipeline runs straight after mapping, it only sees mappings that were already approved. New proposals reach the output only after review and a second pipeline run. Design the job, or the review step, accordingly.

## Working with Genie Code

- **Create the pipeline in the bundle before opening the pipeline editor.** Opening the editor first created a placeholder pipeline outside the bundle, and with auto-approve on, the editor could only edit that placeholder's folder, so it couldn't write the bundle's files. Deleting the placeholder afterwards was then blocked as a destructive action.
- **Go back to the bundle after the pipeline editor.** Do the done check, notes and commit from the bundle's Git folder.
- **Check for leftover files from interrupted turns.** Untracked files from an earlier attempt turned up mid-phase. Read them in full and compare with the approved design before using or replacing them.

## Permissions (optional app)

- **Use PATCH to add permissions.** A PUT on a job's or pipeline's permissions replaces the whole list and fails if it would remove the owner.
- **The app runs as its own service principal.** It needs explicit grants on the catalog, schemas, volume, tables, job and pipelines.
- **Use the SDK's statement execution API from an app.** The SQL connector fell back to an interactive login inside the app container; the SDK authenticated as the service principal without extra configuration.
