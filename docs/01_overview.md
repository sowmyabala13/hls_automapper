# Overview: Intelligent File Auto-Mapper

## Problem

New data sources arrive as files whose layouts differ from the target data model: different column names, abbreviations, date formats, file splits and delimiters. Mapping each new source by hand takes weeks of analyst and engineering time, and the knowledge of how earlier sources were mapped is rarely reused.

## Goal

Cut the time to onboard a new source from weeks to days by having AI propose the mappings, a person approve them, and a deterministic pipeline apply them. Each approved mapping becomes context for the next source, so onboarding gets faster over time.

## Architecture

| Stage | What happens | Key output |
| --- | --- | --- |
| 1. Land | Source files are uploaded to a Unity Catalog volume, organized by source and file type. Spec documents are stored separately | Files in the volume |
| 2. Ingest | Auto Loader streams every file into a bronze Delta table. Each row is stored as a VARIANT, so no schema is imposed at this stage | Bronze table |
| 3. Profile | Each newly ingested file is profiled: column names, value patterns, null rates, cardinality, sample values | Source profile table |
| 4. Map | A Genie Code task proposes, for every source column, a target table and column plus the expression to produce it, using Unity Catalog context | Proposed mappings |
| 5. Review | A person approves, edits or rejects each proposal | Approved mappings (versioned) |
| 6. Apply | A metadata-driven Spark Declarative Pipeline reads the approved mappings and transforms bronze into the target model | Target tables in a dev schema |
| 7. Validate | Row counts, data quality checks, and comparison against known-correct mappings where available | Validation report |

See `diagrams/end_to_end_flow.md`.

## Design principles

1. **Bronze is schema-agnostic.** Files are stored as they arrived, as VARIANT, with every value as a string. Nothing is typed or renamed in bronze.
2. **AI proposes, people approve, pipelines apply.** Genie Code never writes to the target model. It only writes proposals. Only approved mappings reach the pipeline.
3. **Unity Catalog is the context.** Mapping quality depends on what Genie Code can read: column comments on the target tables, the source's spec documents, and earlier approved mappings. Invest in those before tuning prompts.
4. **Metadata-driven, not hand-coded per source.** One pipeline reads mapping rows and applies them. Adding a source means adding mappings, not writing a new pipeline.
5. **Everything is a bundle resource.** Schemas, volumes, pipelines and jobs are defined in the bundle so the same build can be deployed to dev, test and prod.
6. **Sources are not uniform.** A single delivery can contain several file types, each with its own layout, delimiter, date formats and target table. Some files only add columns to another file's target table. The design must handle all of this through configuration, not code changes.

## Scope

In scope: delimited and JSON files, multiple file types per delivery, multiple target tables, human review, metadata-driven transformation.

Out of scope for the first build: real-time APIs, HL7/FHIR feeds, a production-grade review app (an optional lightweight version is described in `02_components.md`).
