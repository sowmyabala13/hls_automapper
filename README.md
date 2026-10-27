# Auto-Mapper Context Pack

Design context for building an AI-assisted file auto-mapper on Databricks with Genie Code.

This repo is **not** a finished product to install. It describes what to build, why, and what to watch out for. You build it in your own workspace by pointing Genie Code at this folder and working through the build plan with it.

## What gets built

```
Client files land in a UC volume
  -> Auto Loader ingests them into a bronze Delta table (VARIANT)
  -> Each new file is profiled
  -> A Genie Code task proposes source-to-target mappings using Unity Catalog context
  -> A person reviews and approves the mappings
  -> A metadata-driven Spark Declarative Pipeline applies approved mappings into the target model
```

See `docs/diagrams/end_to_end_flow.md` for the diagram.

## Repo layout

```
.
├── README.md                 this file
├── docs/
│   ├── 01_overview.md        problem, architecture, design principles
│   ├── 02_components.md      each component: purpose, inputs, outputs, requirements
│   ├── 03_build_plan.md      phases with goals, requirements and done-checks
│   ├── 04_lessons_learned.md problems hit in a previous build, and how to avoid them
│   ├── 05_prerequisites.md   what must be in place before you start
│   └── diagrams/
├── prompts/                  one prompt per step, from setup to profiling
├── reference/                YOUR source spec docs and target model notes go here (git-ignored)
└── project/                  created by prompts/00_setup.md: the bundle Genie Code builds into
```

Everything Genie Code produces (bundle resources, pipeline code, analysis markdowns, mapping files) goes under `project/`. The `docs/` folder is reference material and should not be edited during a build.

## How to use it

1. **Bring the repo into your workspace.** Clone it into your home folder as a Git folder.
2. **Open Genie Code in agent mode** and work through the files in `prompts/`, in order. Open each file, replace everything in `<angle brackets>`, and send the prompt. Each file also says what to check before you approve the next step.

| Prompt | Phases | What it does |
| --- | --- | --- |
| `prompts/00_setup.md` | | Feature branch, empty bundle in `project/`, your reference material saved to `reference/` |
| `prompts/01_orient.md` | | Genie Code reads the pack and your reference material, and lists its open questions |
| `prompts/02_preflight_environment.md` | 0, 1 | Prerequisite checks; schema and volume as bundle resources |
| `prompts/03_inventory_layout.md` | 2 | Raw file inventory, landing layout, file registry, upload |
| `prompts/04_bronze.md` | 3 | Bronze ingestion pipeline |
| `prompts/05_profiling.md` | 4 | Source profile and exploration |

Every prompt follows the same pattern: read the earlier phase notes, meet the component spec, show the design and wait for approval, build and deploy to dev, check "Done when", write the phase notes, commit locally.

For phases 5 to 9, work from `docs/03_build_plan.md` directly, using the same pattern. Each phase lists what it must produce and how to tell it is done.

## Ground rules for the build

- Work on a feature branch. Commit after each phase.
- Define schemas, volumes, pipelines and jobs as bundle resources, not by hand.
- Never write to the target model's production tables during a build. Use a dev schema.
- Each phase starts by reading the previous phase's markdown in `project/docs/analysis/`, so any session or person can pick up the work.
