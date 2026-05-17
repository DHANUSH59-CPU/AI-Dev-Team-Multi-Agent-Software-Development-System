# AI Dev Team — System Architecture

A multi-agent autonomous software development system. Takes a plain-English requirement and produces a working application through 27 specialized AI agents orchestrated by LangGraph.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph User["USER LAYER"]
        U[User Browser]
    end

    subgraph Frontend["FRONTEND - React + Vite :5173"]
        UI[Mission Control Dashboard]
        PV[Pipeline Visualizer]
        LS[Log Stream]
        OP[Output Panel]
        HIP[Human Input Panel]
        TBB[Token Budget Bar]
        ZS[Zustand Store]
    end

    subgraph Backend["BACKEND - Node.js + Express :3000"]
        REST[REST API /api/projects]
        WS[WebSocket Server /ws]
        GR[Graph Runner Service]
    end

    subgraph Core["CORE - LangGraph Orchestration"]
        SG[State Graph - 27 Nodes]
        STATE[(Shared Agent State)]
        CP[Checkpointer]
    end

    subgraph External["EXTERNAL SERVICES"]
        GEM[Google Gemini API]
        REDIS[(Redis - State Persistence)]
        DOCK[Docker Sandbox]
    end

    U -->|HTTP| UI
    UI <--> ZS
    UI -->|REST| REST
    UI <-->|WebSocket| WS
    REST --> GR
    WS --> GR
    GR --> SG
    SG <--> STATE
    SG <--> CP
    CP <--> REDIS
    SG -->|LLM calls| GEM
    SG -->|exec code| DOCK

    style User fill:#1e293b,stroke:#3b82f6,color:#fff
    style Frontend fill:#0f172a,stroke:#10b981,color:#fff
    style Backend fill:#0f172a,stroke:#f59e0b,color:#fff
    style Core fill:#0f172a,stroke:#ef4444,color:#fff
    style External fill:#0f172a,stroke:#8b5cf6,color:#fff
```

---

## 2. The 7-Phase Pipeline (End-to-End Flow)

```mermaid
flowchart LR
    START([User Requirement]) --> P1

    subgraph P1["PHASE 1: PM"]
        PM[PM Agent]
        HI[Human Input]
        PM <--> HI
    end

    subgraph P2["PHASE 2: ARCHITECT"]
        A1[Step 1: Entities]
        A2[Step 2: DB Schema]
        A3[Step 3: API Endpoints]
        A4[Step 4: Frontend Pages]
        A5[Step 5: Folder Structure]
        BV{Blueprint Validator}
        A1 --> A2 --> A3 --> A4 --> A5 --> BV
        BV -->|invalid| A2
    end

    subgraph P3["PHASE 3: PLANNER"]
        PL[Planner Agent]
        SS[Setup Sandbox]
        SHC{Health Check}
        PL --> SS --> SHC
    end

    subgraph P4["PHASE 4: DEV LOOP"]
        direction TB
        SNT{Select Next Task}
        CB[Context Builder]
        CA[Coder Agent]
        UR[Update Registry]
        RA{Reviewer}
        EA{Executor}
        DA{Debugger}
        SM[Snapshot Manager]
        ST[Simplify Task]
        HE[Human Escalation]

        SNT --> CB --> CA --> UR --> RA
        RA -->|approved| EA
        RA -->|rejected| CB
        RA -->|3rd fail| ST
        EA -->|pass| SM --> SNT
        EA -->|fail| DA
        DA -->|fix| CB
        DA -->|escalate| HE
        HE -->|guide| CB
        HE -->|skip| SNT
        ST --> SNT
    end

    subgraph P5["PHASE 5: QUALITY"]
        PV[Phase Verification]
        PE[Pattern Extractor]
        SC[State Compactor]
        PV --> PE --> SC
    end

    subgraph P6["PHASE 6: DEPLOY"]
        DV{Deployment Verifier}
        PTU[Present to User]
    end

    PM -->|spec_ready| A1
    BV -->|valid| PL
    SHC -->|healthy| SNT
    SNT -->|phase done| PV
    SC --> SNT
    SNT -->|all done| DV
    DV -->|pass| PTU
    DV -->|fail| DA
    PTU --> END([Working App])

    style P1 fill:#1e3a8a,color:#fff
    style P2 fill:#065f46,color:#fff
    style P3 fill:#78350f,color:#fff
    style P4 fill:#7f1d1d,color:#fff
    style P5 fill:#581c87,color:#fff
    style P6 fill:#0c4a6e,color:#fff
```

---

## 3. The Dev Loop — Self-Healing Mechanism (Phase 4 Detail)

```mermaid
stateDiagram-v2
    [*] --> SelectTask
    SelectTask --> ContextBuilder: task available
    SelectTask --> PhaseVerification: phase complete
    SelectTask --> DeploymentVerifier: all tasks done

    ContextBuilder --> CoderAgent
    CoderAgent --> UpdateRegistry: writes files
    UpdateRegistry --> ReviewerAgent

    ReviewerAgent --> ExecutorAgent: approved
    ReviewerAgent --> ContextBuilder: rejected, retry < 2
    ReviewerAgent --> SimplifyTask: rejected, retry >= 2

    ExecutorAgent --> SnapshotManager: tests pass
    ExecutorAgent --> DebuggerAgent: tests fail

    DebuggerAgent --> ContextBuilder: fix found
    DebuggerAgent --> HumanEscalation: can't fix

    HumanEscalation --> ContextBuilder: human guides
    HumanEscalation --> SelectTask: human skips
    HumanEscalation --> SimplifyTask: simplify

    SnapshotManager --> SelectTask
    SimplifyTask --> SelectTask
    PhaseVerification --> PatternExtractor
    PatternExtractor --> StateCompactor
    StateCompactor --> SelectTask
    DeploymentVerifier --> [*]: success
    DeploymentVerifier --> DebuggerAgent: deployment fails
```

---

## 4. State Communication Model

```mermaid
graph LR
    subgraph SharedState["SHARED AGENT STATE - LangGraph Annotation.Root"]
        S1[userRequirement]
        S2[clarifiedSpec]
        S3[blueprint]
        S4[taskQueue]
        S5[fileRegistry]
        S6[currentTask]
        S7[reviewResult]
        S8[executionResult]
        S9[tokenUsage]
        S10[projectPatterns]
    end

    subgraph Writers["NODES WRITE"]
        N1[pmAgent]
        N2[architectStep1-5]
        N3[plannerAgent]
        N4[coderAgent]
        N5[reviewerAgent]
        N6[executorAgent]
    end

    subgraph Readers["NODES READ"]
        R1[architectStep1]
        R2[plannerAgent]
        R3[contextBuilder]
        R4[reviewerAgent]
        R5[executorAgent]
    end

    N1 -->|writes| S2
    N2 -->|writes| S3
    N3 -->|writes| S4
    N4 -->|writes| S5
    N4 -->|writes| S6
    N5 -->|writes| S7
    N6 -->|writes| S8

    S2 -->|reads| R1
    S3 -->|reads| R2
    S5 -->|reads| R3
    S6 -->|reads| R4
    S6 -->|reads| R5

    style SharedState fill:#1e293b,stroke:#3b82f6,color:#fff
    style Writers fill:#065f46,color:#fff
    style Readers fill:#7f1d1d,color:#fff
```

---

## 5. Request Flow: From Browser Click to Code Generation

```mermaid
sequenceDiagram
    actor User
    participant UI as React Dashboard
    participant API as Express REST
    participant WS as WebSocket
    participant GR as Graph Runner
    participant LG as LangGraph
    participant GEM as Gemini API
    participant DOC as Docker Sandbox
    participant RED as Redis

    User->>UI: Enter requirement + click LAUNCH
    UI->>API: POST /api/projects
    API->>GR: createProject(requirement)
    GR->>LG: graph.invoke(initialState)
    UI->>WS: Connect ws://localhost:3000/ws

    loop For each agent node
        LG->>GEM: LLM call with prompt
        GEM-->>LG: Generated content
        LG->>RED: Checkpoint state
        LG->>WS: Stream progress event
        WS-->>UI: Real-time update
    end

    LG->>DOC: Write & execute code
    DOC-->>LG: Execution result

    alt Needs human input
        LG->>WS: Request human input
        WS-->>UI: Show input panel
        User->>UI: Provide answer
        UI->>WS: Send response
        WS->>LG: Resume with input
    end

    LG-->>GR: Final state
    GR-->>WS: Project complete
    WS-->>UI: Show generated app
    UI-->>User: Display result
```

---

## 6. Technology Stack

```mermaid
mindmap
  root((AI Dev Team))
    Frontend
      React 18
      Vite
      Zustand State
      WebSocket Client
      CSS Industrial Theme
    Backend
      Node.js ESM
      Express
      WebSocket ws
      CORS
    AI Orchestration
      LangGraph
      State Annotations
      Conditional Edges
      Checkpointing
    LLM
      Google Gemini
      gemini-2.5-flash
      Token Tracking
      Cost Estimation
    Infrastructure
      Docker Sandbox
      Redis Persistence
      File System Registry
      Snapshot Manager
    Patterns
      Multi-Agent System
      State Machine
      Self-Healing Loop
      Human-in-the-Loop
      Event Streaming
```

---

## Key Architectural Decisions

| Decision | Why |
|----------|-----|
| **LangGraph over plain function calls** | State-driven communication scales to 27 nodes without spaghetti code. Conditional edges handle retries/branching cleanly. |
| **Shared state with reducers** | Nodes are pure — they read state, return updates. Reducers handle merging (arrays accumulate, objects merge, scalars overwrite). |
| **Checkpointing after every node** | Long pipelines (sometimes 30+ minutes) can crash. Resume from last good state instead of restarting. |
| **Docker sandbox for code execution** | Generated code is untrusted. Isolation prevents host system damage. |
| **WebSocket for dashboard** | Long-running pipeline needs push-based updates. Polling would miss events and waste resources. |
| **Token budget enforcement** | LLM calls cost money. Hard limit prevents runaway costs from infinite loops. |
| **Pattern extraction phase** | Without it, each task generates code in a different style. Extracting patterns keeps the codebase consistent. |
