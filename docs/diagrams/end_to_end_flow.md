# End-to-End Flow

```mermaid
flowchart TB
    subgraph SRC["Source delivery"]
        F1["Data files<br/>(several file types,<br/>delimited or JSON)"]
        D1["Spec documents<br/>and mapping rules"]
    end

    subgraph UC["Unity Catalog volume"]
        ROOT["Ingestion root<br/>source / file type / files"]
        DOCS["Docs area<br/>(outside the ingestion root)"]
    end

    subgraph REPO["Git repo"]
        REF["reference/<br/>spec docs for @ context"]
        ANALYSIS["project/docs/analysis/<br/>phase markdowns"]
    end

    REG["File registry<br/>(format, delimiter, target table,<br/>primary or supplemental, join keys)"]

    subgraph BRONZE["Bronze"]
        AL["Auto Loader<br/>Spark Declarative Pipeline"]
        BT["Bronze table<br/>VARIANT row + file metadata"]
    end

    subgraph MAP["Mapping"]
        PROF["Source profile"]
        GC["Genie Code<br/>(session, then job task)"]
        CTX["Unity Catalog context<br/>target column comments<br/>earlier approved mappings"]
        PROP["Proposed mappings"]
        REV{"Human review"}
        APPR["Approved mappings<br/>(versioned)"]
    end

    subgraph TARGET["Target model (dev schema)"]
        MDP["Metadata-driven<br/>Spark Declarative Pipeline"]
        TT["Target tables<br/>one per target table"]
        VAL["Validation"]
    end

    F1 --> ROOT
    D1 --> DOCS
    D1 --> REF
    ROOT --> AL
    REG -.-> AL
    AL --> BT
    BT --> PROF
    PROF --> GC
    BT -. samples .-> GC
    CTX -.-> GC
    REF -. "@" .-> GC
    REG -.-> GC
    GC --> PROP
    PROP --> REV
    REV -->|approve / edit| APPR
    APPR --> MDP
    BT --> MDP
    REG -. join keys .-> MDP
    MDP --> TT
    TT --> VAL
    APPR -. compounds .-> CTX
```

**Reading the diagram**
- Solid arrows are data flow; dotted arrows are context or configuration.
- Genie Code writes only to *Proposed mappings*. Only *Approved mappings* reach the pipeline.
- Approved mappings feed back into the context for the next source, which is how onboarding gets faster over time.
