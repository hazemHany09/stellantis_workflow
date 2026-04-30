# 12 — Workflow Diagram

End-to-end view of every stage, gate, and actor involved in producing a deliverable for one car. This document is **descriptive** — it consolidates what the other business-logic files already say into a single visual reference. It is the bridge between the business logic (documents 00–11) and the workflow design phase that follows.

The diagrams stay at the **domain level**. They describe stages, gates, and actors as roles in the analysis — not as specific tools, file paths, or agent topologies. Implementation specifics (which subagents are spawned, which files are written under an internal workspace, which vendor tool is called) belong to the skill packages, not to this document.

> Legend:
> • **Rounded boxes** = analysis stages
> • **Diamonds** = gates / decisions
> • **Parallelograms** = external inputs or outputs
> • **Cylinders** = persistent stores (KB, approval store, deliverable store)
> • Solid arrows = sequential flow; dashed arrows = asynchronous / decoupled interactions

***

## A. End-to-end master flow (normal pipeline)

```mermaid
flowchart TD
    subgraph CLIENT[Client side]
        C1[/"Submit request<br/>(brand + model + year + market,<br/>optional source list)"/]
        C2{{"Approve / reject URLs<br/>via approval store UI"}}
        C3[/"Send 'resume' signal"/]
        C4[/"Review deliverable"/]
    end

    subgraph ANALYSIS[Analysis — single run, one car]
        A1["Stage 1 — Parse request<br/>extract 4 required fields"]
        A2{"All 4 fields<br/>extracted?"}
        A3["Prompt client for<br/>missing field"]
        A4["Stage 2 — Resolve car identity<br/>(auto-select prominent variant)"]
        A5["Stage 3 — Load reference parameter list<br/>and freeze for the run"]
        A6{"Source mode?"}
        A7A["Mode A — accept<br/>client-supplied URLs"]
        A7B["Mode B — web search<br/>on brand+model+year+market"]
        A8["Canonicalise + dedupe URLs<br/>(strip fragments, tracking,<br/>lowercase host)"]
        A9["Paywall filter<br/>(silently drop)"]
        A10["Cap list at 20 URLs<br/>(Mode B default)"]
        A11["Flag-for-approval<br/>(URL + title + excerpt +<br/>why-selected + car identity)"]
        A12[["APPROVAL GATE<br/>Analysis yields"]]
        A13["Retrieve-approved<br/>fetch final URL set"]
        A14["Download each URL<br/>(3 retries, backoff)"]
        A15["Upload to KB dataset<br/>with car + run metadata"]
        A16["Wait for ingestion<br/>until every document is queryable<br/>(60-min per-doc ceiling)"]
        A17["Partition the parameter list<br/>by category"]
        A18[/"Each partition retrieves evidence<br/>from the shared KB per parameter"/]
        A19["Each partition emits<br/>Presence / Status / Classification /<br/>Confidence + traceability blocks"]
        A20["Consolidate per-parameter records<br/>(consensus check, inverse-retrieval enforcement)"]
        A21["Generate deliverable<br/>(JSON primary + CSV secondary)"]
    end

    subgraph STORES[Persistent stores]
        S1[("Source approval store<br/>(Pending / Approved / Rejected)")]
        S2[("Knowledge base dataset<br/>(one per run)")]
        S3[("Deliverable store<br/>JSON + CSV")]
    end

    C1 --> A1 --> A2
    A2 -- no --> A3 --> C1
    A2 -- yes --> A4 --> A5 --> A6
    A6 -- Mode A --> A7A --> A8
    A6 -- Mode B --> A7B --> A8
    A8 --> A9 --> A10 --> A11
    A11 -.writes.-> S1
    A11 --> A12
    C2 -.reads/writes.-> S1
    C3 --> A13
    A13 -.reads.-> S1
    A13 --> A14 --> A15
    A15 -.writes.-> S2
    A15 --> A16
    A16 -.polls.-> S2
    A16 --> A17 --> A18
    A18 -.queries.-> S2
    A18 --> A19 --> A20 --> A21
    A21 -.writes.-> S3
    A21 --> C4
```

***

## B. Source lifecycle per URL

```mermaid
stateDiagram-v2
    [*] --> Proposed: discovered or supplied
    Proposed --> Canonicalised: strip tracking, fragments, lowercase host
    Canonicalised --> Dropped_Paywall: paywall detected
    Canonicalised --> Dropped_Duplicate: already in list
    Canonicalised --> Flagged: flag-for-approval
    Flagged --> PendingApproval
    PendingApproval --> Approved: client approves
    PendingApproval --> Rejected: client rejects
    Approved --> Downloaded: success within 3 retries
    Approved --> Dropped_DownloadFail: 3 retries exhausted
    Downloaded --> Ingesting: uploaded to KB
    Ingesting --> Ingested: ingestion confirmed
    Ingesting --> Dropped_IngestTimeout: 60 min elapsed
    Ingested --> Queryable
    Queryable --> DeletedFromKB: removed (duplicate / off-target)
    Rejected --> [*]
    Dropped_Paywall --> [*]
    Dropped_Duplicate --> [*]
    Dropped_DownloadFail --> [*]: reported in footer
    Dropped_IngestTimeout --> [*]: reported in footer
    Queryable --> [*]
```

***

## C. Decision-rule branching per parameter (harness interface)

```mermaid
flowchart TD
    P["Evidence set for parameter P"] --> TAX["Classify sources<br/>per taxonomy"]
    TAX --> ACT{"Any active<br/>sources?"}
    ACT -- no --> INV["Inverse-retrieval test<br/>(query for negation language)"]
    INV -- absence evidence found --> R5P["Promote to Rule 5<br/>(No, Success, Empty)"]
    INV -- no negative evidence --> R4["Rule 4 — Silent-all<br/>Presence = No Information Found<br/>Status = Unable to Decide<br/>Classification = Empty<br/>inverse_retrieval_attempted = true<br/>Traceability = 0 blocks"]
    ACT -- yes --> CLR{"Any clear<br/>active sources?"}
    CLR -- no --> R3["Rule 3 — Unable to Decide<br/>Presence = Yes<br/>Status = Unable to Decide<br/>Classification = Empty<br/>Traceability ≥ 1 vague block"]
    CLR -- yes --> AGR{"Clear sources<br/>agree?"}
    AGR -- no --> CONF{"Disagreement<br/>type?"}
    CONF -- presence vs absence --> R2B["Rule 2b — Presence conflict<br/>Presence = Disputed<br/>Status = Conflict<br/>Traceability ≥ 2 clear blocks"]
    CONF -- level vs level --> R2A["Rule 2a — Level conflict<br/>Presence = Yes<br/>Status = Conflict<br/>Classification = Empty<br/>Traceability ≥ 2 clear blocks"]
    AGR -- yes --> STANCE{"Stance of<br/>agreed clear<br/>sources?"}
    STANCE -- present + valid level --> UGCCHK{"Only source-type<br/>= forum_or_ugc?"}
    UGCCHK -- yes --> R3DEM["Demote to Rule 3<br/>(Yes, Unable to Decide, Empty)"]
    UGCCHK -- no --> R1["Rule 1 — Success (present)<br/>Presence = Yes<br/>Status = Success<br/>Classification = agreed level<br/>confidence ∈ {consensus, single-source}<br/>Traceability ≥ 1 clear block"]
    STANCE -- present + undefined tier only --> R3B["Invariant 7 → Rule 3<br/>Undefined-tier evidence<br/>does not snap to nearest level"]
    STANCE -- explicit absent --> R5["Rule 5 — Success (absent)<br/>Presence = No<br/>Status = Success<br/>Classification = Empty<br/>Traceability ≥ 1 clear block"]
    STANCE -- mixed: valid level + undefined tier --> R1X["Q-HARN-11 → Rule 1<br/>Undefined-tier source disregarded<br/>Out-of-schema advisory recorded internally"]
```

***

## D. Confidence assignment after rule selection

```mermaid
flowchart LR
    REC["Record at Rule 1 / Rule 5"] --> CAT{"Distinct source-type<br/>categories backing<br/>the verdict?"}
    CAT -- ≥ 2 categories --> CONS["confidence = consensus"]
    CAT -- exactly 1 category --> SS["confidence = single-source"]
    SS --> LATER{"Later cycle adds<br/>source from new category?"}
    LATER -- yes --> CONS
    LATER -- no --> SS
```

***

## E. Hard phase boundary (ARCH-4)

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Analysis
    participant S as Approval store
    participant K as KB

    Note over A: PHASE 1 - Source discovery
    C->>A: Submit request
    A->>A: Resolve car identity
    A->>A: Web search (Mode B) or accept URLs (Mode A)
    A->>A: Canonicalise + dedupe + paywall-filter
    A->>S: flag-for-approval (per URL)
    Note over A: Analysis yields; web-search capability never used again this run

    Note over C,S: GATE - asynchronous
    C->>S: Approve / reject URLs (may take minutes to days)
    C->>A: Resume classification signal

    Note over A: PHASE 2 - Ingestion + classification
    A->>S: retrieve-approved
    S-->>A: Approved URL set (fixed)
    A->>K: download + upload each URL
    A->>K: wait for ingestion confirmation
    K-->>A: all queryable
    A->>A: partition parameters, query KB per parameter, run inverse-retrieval test on silent parameters
    A->>A: consolidate + emit deliverable
    A->>C: deliverable.json + deliverable.csv

    Note over A: If client wants more sources post-gate: only option is a new full run (PHASE 1 restart) or separate gap-fill workflow.
```

***

## F. Error-handling overlay

```mermaid
flowchart TD
    E1["URL fails download<br/>(HTTP error, timeout)"] --> E1R["Retry 3×<br/>1s / 2s / 4s backoff"]
    E1R -- still failing --> E1D["Drop URL from active set<br/>Record in deliverable footer<br/>sources_excluded"]

    E2["Doc exceeds<br/>60-min ingestion window"] --> E2D["Drop doc from active set<br/>Record in footer<br/>stage: ingestion"]

    E3["Required input field missing"] --> E3P["Prompt client<br/>for that field only"]

    E4["Ambiguous car tuple<br/>(facelift / rebadge / variant)"] --> E4R["Auto-resolve to most<br/>prominent variant<br/>Record resolution in header"]

    E5["Single URL approved"] --> E5C["Continue — no minimum"]

    E6["All sources silent on parameter,<br/>inverse retrieval also silent"] --> E6R4["Rule 4<br/>Unable to Decide / No Information Found / Empty<br/>inverse_retrieval_attempted = true<br/>(no traceability blocks)"]

    E7["Clear sources disagree"] --> E7C["Emit Status = Conflict<br/>No tiebreak attempted"]

    E8["Evidence targets<br/>undefined tier"] --> E8R["If only evidence: Rule 3<br/>If mixed with valid: Rule 1<br/>+ out-of-schema advisory recorded internally"]

    E9["Concurrent run on same car"] --> E9I["Allowed; fully independent<br/>(separate run ID / KB / deliverable)"]

    E10["Rule 1 verdict backed only by<br/>forum_or_ugc source"] --> E10D["Demote to Rule 3<br/>Yes / Unable to Decide / Empty"]
```

***

## G. Document-cross-reference map

Use this to navigate between the diagram above and the business-logic documents that define each element.

| Element in diagram                          | Authoritative document                            |
| :------------------------------------------ | :------------------------------------------------ |
| Input extraction, four required fields      | `03-inputs.md`, `11-assumptions.md` Q-INPUT-1..8  |
| Car-identity resolution                     | `03-inputs.md`, Q-INPUT-6                         |
| Reference parameter list load + freeze      | `02-parameters-and-levels.md`, Q-SCOPE-2          |
| Mode A / Mode B source selection            | `04-sources.md`                                   |
| URL canonicalisation                        | `04-sources.md`, Q-SRC-7                          |
| Paywall filter                              | Q-SRC-4                                           |
| Candidate cap                               | Q-SRC-9                                           |
| flag-for-approval metadata                  | Q-SRC-11                                          |
| Approval gate + resume signal               | `04-sources.md`, Q-SRC-6                          |
| Download retries                            | Q-KB-4                                            |
| Ingestion timeout                           | Q-KB-5                                            |
| Fresh dataset per run                       | Q-KB-2                                            |
| Source lifecycle                            | `04-sources.md`, Q-SRC-10 (Retired dropped)       |
| Parameter partition                         | Q-HARN-13                                         |
| Decision rules 1 / 2 / 3 / 4 / 5            | `10-decision-rules.md`, `06-harness-interface.md` |
| Source-type categories + consensus          | `10-decision-rules.md`                            |
| Inverse retrieval (Rule 4 → Rule 5 path)    | `10-decision-rules.md`                            |
| UGC demotion                                | `10-decision-rules.md` invariant 10               |
| Presence / Status / Classification contract | `06-harness-interface.md`                         |
| Confidence + inverse_retrieval_attempted    | `06-harness-interface.md`, `09-deliverable.md`    |
| Traceability blocks                         | `09-deliverable.md`                               |
| Deliverable format (JSON + CSV)             | Q-DEL-1                                           |
| Deliverable header + summary                | Q-DEL-5, Q-DEL-6                                  |
| Footer `sources_excluded`                   | Q-DEL-8                                           |
| Re-run patterns                             | Q-DEL-3, Q-DEL-10                                 |
| Gap-fill workflow                           | Q-HARN-5, Q-KB-6, Q-DEL-10                        |
| Concurrent runs                             | Q-SCOPE-3                                         |
| Out-of-list findings                        | Q-SCOPE-5                                         |
