# Prerequisites

Confirm all of these before the build session. Missing permissions are the most common reason a live session stalls, so have a workspace admin reachable during the session.

## Required

| Prerequisite | Needed for | How to check |
| --- | --- | --- |
| **Target model available** in the workspace, with SELECT on its tables and USE on its catalog and schema | Target model analysis, mapping, pipeline | The person driving the session can query the target tables |
| **Git repo** where the driver can create a feature branch and commit | Everything Genie Code produces | Git credentials linked in the driver's workspace settings; they can create a Git folder from the repo and push a test branch |
| **Development catalog** with USE CATALOG and CREATE SCHEMA (preferred), or USE CATALOG, USE SCHEMA and create privileges on an existing dev schema | Environment setup | `SHOW GRANTS` for the driver on the catalog or schema |
| **A volume**: CREATE VOLUME on the dev schema, or a pre-created volume with READ VOLUME and WRITE VOLUME | Landing files | Same as above |
| **Genie Code fully enabled**, including agent mode | Every phase | The driver can run an agent-mode task end to end |
| **Genie Code file upload** in chat | Raw file inventory | The driver can attach a file to a Genie Code message |
| **Serverless compute** for notebooks, jobs and pipelines | Ingestion, profiling, pipelines | A serverless notebook runs and a test pipeline can be created |
| **Bundles** can be created and deployed from the workspace | All resources | The driver can create an empty bundle and deploy it to a dev target |

## Privileges on the fallback dev schema

If the driver cannot create a schema, they need these on the dev schema they are given:

- CREATE TABLE (bronze streaming tables, mapping store tables)
- CREATE MATERIALIZED VIEW (if the target pipeline uses materialized views)
- CREATE VOLUME (unless the volume is pre-created)

Pipelines run as the person who runs them, so that identity also needs read access on the volume and target model, and write access to the dev schema.

## Needed for specific phases

| Prerequisite | Needed for |
| --- | --- |
| Column comments on the target model tables, or a data dictionary in `reference/` | Mapping quality. Without either, Genie Code maps from column names alone |
| Lineage visible for the target tables, and query history access for query-pattern analysis | Target model analysis |
| Variant shredding available in the workspace (may be in preview) | Bronze table settings |
| Genie Code automations available (used by a Genie Code job task) | Running the mapping step as a scheduled job task |

## Only if you build the optional review app

- Databricks Apps enabled, and permission to deploy an app
- A SQL warehouse the app can use
- Someone who can grant the app's service principal access to the catalog, schemas, volume, tables, job and pipelines
