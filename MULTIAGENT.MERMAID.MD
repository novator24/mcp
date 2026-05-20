# Multiagent HLD (based on `openai/codex`)

Ниже HLD взаимодействий внутри мультиагентной системы Codex (по открытой структуре `codex-rs`, `app-server`, MCP/tooling слоям).

```mermaid
flowchart TD
    U[User / IDE Client] --> AS[Codex App Server\nJSON-RPC transport layer]
    AS --> TM[Thread Manager\nthread/start resume fork]
    TM --> ORCH[Turn Orchestrator\nturn/start steer interrupt]

    ORCH --> AG[Agent Runtime\nCodexThread + session context]
    AG --> SK[Skills Manager\nimplicit/explicit skill injection]
    AG --> PM[Plugin Manager\nmarketplace + plugin skills/hooks/apps]

    AG --> MCP[MCP Manager\nserver discovery + tool exposure]
    MCP --> MCPS[MCP Servers\nexternal/internal tools]

    AG --> TP[Tool Planner\nfunction/tool call routing]
    TP --> SH[Shell Executor\ncommand/exec + thread shellCommand]
    TP --> FS[Filesystem Ops\nfs/read write watch remove]
    TP --> WB[Web Search / Fetch]
    TP --> APPS[App Connectors\napp/list + auth endpoints]

    SH --> SBX[Sandbox & Exec Policy\nworkspace-write/full-access gates]
    FS --> SBX

    AG --> MODEL[Model Client Session\nprovider capabilities/model selection]
    MODEL --> LLM[LLM Provider APIs]

    AG --> EVT[Event Stream Mapper\nitem started/completed deltas]
    EVT --> AS
    AS --> U

    AG --> ROLLOUT[Rollout Recorder / State DB\nthreads turns items metadata]
    ROLLOUT --> TM

    SEC[Safety & Guardrails\napproval policies redaction checks] --> TP
    SEC --> SBX
    SEC --> EVT
```

```mermaid
sequenceDiagram
    participant C as Client (IDE/CLI)
    participant S as App Server
    participant T as Thread Manager
    participant O as Orchestrator/Agent
    participant M as MCP Manager
    participant X as External MCP Server
    participant E as Exec/Sandbox
    participant L as LLM Provider

    C->>S: initialize + initialized
    C->>S: thread/start
    S->>T: create/load thread context
    T-->>S: thread/started
    C->>S: turn/start (user intent)
    S->>O: start turn
    O->>M: discover/select tools
    M->>X: tools/call (if needed)
    X-->>M: tool result
    O->>E: command/exec or fs ops (policy-checked)
    E-->>O: execution result
    O->>L: model request (context + tool outputs)
    L-->>O: streamed tokens / function calls
    O-->>S: item/* notifications + final agent message
    S-->>C: turn/completed + usage/metadata
```

## Ключевые архитектурные акценты

- Оркестрация идет через `thread`/`turn` примитивы (сессии и события первого класса).
- Мультиагентность реализуется как кооперация подсистем: planner, skills/plugins, MCP manager, exec/sandbox, model client.
- Инструменты и внешние агенты подключаются через MCP и app-server API, а не через hardcoded интеграции.
- Безопасность вшита в execution path: approvals, sandbox policy, redaction и контроль событий.
