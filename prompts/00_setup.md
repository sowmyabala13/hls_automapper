# 00. Setup

**When:** once, right after you clone this repo into your workspace as a Git folder.
**Builds:** a feature branch, an empty bundle in `project/`, and your reference material in `reference/`. Nothing is deployed.

Replace everything in `<angle brackets>`. `<REPO>` is the full workspace path of the Git folder, for example `/Workspace/Users/<you>/<repo name>`.

## Before you send it

- Collect the source's reference material: file specs, data dictionaries, mapping rules (renames, naming conventions, required output labels, date formats per file), and notes on your target model. Paste text into the prompt or attach the files.
- Do not include data files. Those land in the volume in prompt 03.

## Prompt

```
Set up @<REPO> for a new build. Do not deploy anything.
1. Create a local feature branch named <BRANCH> from main.
2. Create an empty bundle in <REPO>/project: a databricks.yml with bundle name
   <BUNDLE_NAME>, one dev target in development mode, and no resources or sample code.
   Validate it.
3. Save the reference material below (or attached) into <REPO>/reference/, one file
   per document, exactly as written. reference/.gitignore keeps these files out of Git;
   confirm they are ignored and still readable.
4. Commit locally only. Do not push.
Then report what you created and anything you couldn't do.

<PASTE OR ATTACH REFERENCE MATERIAL>
```

## Check before moving on

- `project/` contains only `databricks.yml`, with one dev target and no resources.
- `git status` shows the reference files as ignored, not untracked.
- The commit contains only `project/databricks.yml` (and nothing from `reference/` except its README and `.gitignore`).

**Next:** `01_orient.md`.
