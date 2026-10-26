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
├── reference/                YOUR source spec docs and target model notes go here
└── project/                  created by you in step 2: the bundle Genie Code builds into
```

Everything Genie Code produces (bundle resources, pipeline code, analysis markdowns, mapping files) goes under `project/`. The `docs/` folder is reference material and should not be edited during a build.

## How to use it

1. **Bring the repo into your workspace.** Clone it into your home folder as a Git folder, then create a feature branch for this build.
2. **Create an empty bundle** in the `project/` folder of the repo.
3. **Add your reference material** to `reference/`: source file specs, mapping rules, data dictionaries, and notes on your target model. Do not put data files here.
4. **Open Genie Code in agent mode** and work through the prompts below. Each prompt references this folder with `@` so Genie Code reads the design docs.

### Opening prompts

Replace everything in `<angle brackets>`.

**1. Orient**
```
@<path to this repo> Read the README and everything in docs/ and reference/.
Describe what this pack asks us to build, the components, and the build phases.
Do not create or change anything yet.
```

**2. Environment**
```
Using @<path to this repo>/docs/03_build_plan.md phase 1:
Use catalog <CATALOG>. Create schema <SCHEMA> and volume <VOLUME> for a new data
ingestion, defined as resources in the bundle at <path to this repo>/project.
First check whether I can create a schema in <CATALOG>. If I can't, use the
existing schema <FALLBACK_SCHEMA> and create only the volume there.
Deploy the bundle to the dev target, then record what was created and why in
project/docs/analysis/00_environment.md.
```

**3. Inventory the raw files** (attach the files, or screenshots of their headers)
```
These are the raw files the source sent: <count>, named <names>.
Following phase 2 of @<path to this repo>/docs/03_build_plan.md, inventory them:
how many, what types (data vs spec documents), formats and delimiters, header rows,
sizes. Then recommend a folder layout in the volume so they can be auto-loaded
into bronze, with spec documents kept out of the ingestion path.
Write the results to project/docs/analysis/01_raw_inventory.md.
```

**4. Plan the build**
```
Using @<path to this repo>/docs and our outputs so far in project/docs/analysis,
propose a build plan for the remaining phases, adapted to our files and our
target model in <TARGET_CATALOG>.<TARGET_SCHEMA>. Show me the plan before building.
```

From there, build and iterate phase by phase. Each phase in `docs/03_build_plan.md` lists what it must produce and how to tell it is done.

## Ground rules for the build

- Work on a feature branch. Commit after each phase.
- Define schemas, volumes, pipelines and jobs as bundle resources, not by hand.
- Never write to the target model's production tables during a build. Use a dev schema.
- Each phase starts by reading the previous phase's markdown in `project/docs/analysis/`, so any session or person can pick up the work.
