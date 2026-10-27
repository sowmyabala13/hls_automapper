# Lessons Learned

Problems hit in a previous build of this pattern, and how to avoid them. These are about the platform and the design, not about any particular dataset.

## Landing and ingestion

- **Non-data files in the ingestion root break the stream.** Diagram files were copied into a subfolder of the folder Auto Loader watched. Auto Loader tried to parse them as data, the inferred schema absorbed a bogus column, and every retry failed. Recovery needed the files removed, the pipeline stopped, and a full refresh to reset the schema. Keep the ingestion root for data files only.
- **New columns cancel and restart the update.** With schema evolution set to add new columns, a file with columns not seen before makes the update stop and a new update start. Anything that triggers the pipeline and waits for it (an app, a job) must follow the follow-up update rather than treat the first one as failed.
- **Use the file metadata column, not `input_file_name()`.** `input_file_name()` is not supported on serverless with Unity Catalog. Auto Loader exposes file path, name and modification time through its file metadata column.
- **A unique ID is not a row number.** `monotonically_increasing_id()` is unique but not sequential per file. If you need a row position within a file, derive it explicitly.
- **Infer nothing in bronze.** Keep every value as a string so codes with leading zeros and unusual date formats reach profiling intact.

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

