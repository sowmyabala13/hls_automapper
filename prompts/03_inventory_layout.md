# 03. Raw file inventory and landing layout (phase 2)

**When:** after the volume exists.
**Builds:** `01_raw_inventory.md`, `02_landing_layout.md`, the registry seed file and its loader job in the bundle, the file registry table, and the files uploaded into the agreed layout.

## Before you send it

Attach the raw files (or samples with the real header and some data rows). Header screenshots alone are not enough to compare header and row counts.

## Prompt

```
Phase 2 of @<REPO>/docs/03_build_plan.md.
Read project/docs/analysis/00_preflight.md and 00_environment.md, and
docs/04_lessons_learned.md first. Use @<REPO>/reference for the source's specs
and rules.

These are the raw files the source sent: <COUNT>, named <NAMES>.
Inventory them from the files themselves, not from any description: data vs spec
documents, format, delimiter, quoting, header row, encoding, size, rows, header
column count and fields per data row. Where the counts differ, check each column's
values against its header name before proposing a fix. Flag anything that disagrees
with the spec, and any file name starting with _ or .

Then propose a volume layout that meets C1 in docs/02_components.md, and the file
registry rows, including expected counts, parse notes, and corrected column names
or unconfirmed columns where the header doesn't match the data.
Write 01_raw_inventory.md and 02_landing_layout.md, and wait for my approval
before uploading anything or creating the registry.

After my OK: upload the files into the layout, put the registry rows in a seed file
in the bundle with a job that validates and loads it, deploy to dev, run it, check
phase 2's "Done when", and commit locally only.
```

## Check before you approve

- Counts were measured from the files, and header count was compared with fields per row for every file.
- Any header fix is backed by a column-by-column value check, and uncertain positions are marked, not guessed silently.
- Supplemental files have join keys, and you agree with them.
- Questions for the source are listed. Send them now; don't wait for later phases.

**Next:** `04_bronze.md`.
