# 01. Orient

**When:** after setup, before building anything.
**Builds:** nothing. Genie Code reads the pack and your reference material, and tells you what it plans to build and what it needs from you.

## Prompt

```
@<REPO> Read the README, everything in docs/, and everything in reference/,
including git-ignored files.
Describe what this pack asks us to build: the design principles, components C1-C8,
and build phases 0-9. Then apply it to our reference material: which source files
map to which target tables, which files are primary or supplemental, and any rules
that override the pack's defaults.
List the open points and questions you need answered before phase 0.
Do not create or change anything yet.
```

## Check before moving on

- The summary covers every component and phase, not only the first few.
- It used the rules in `reference/` (renames, labels, date formats), not just the pack.
- Answer each open point before sending prompt 02. Write the answers in your reply so they become part of the record.

**Next:** `02_preflight_environment.md`.
