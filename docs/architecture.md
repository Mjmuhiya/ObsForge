# ObsForge Architecture

This document captures the conceptual architecture and principal flows of ObsForge. Update component names and interfaces to match the implementation as it evolves.

## 1. High-level system architecture

```mermaid
flowchart LR
    LG[Load Generator]
    APP[Product Services / Clients]
    API[Ingestion API]
    PROC[Ingestion Processing]
    STORE[(Bounded In-Memory Store)]
    QUERY[Query Service]
    SLO[SLO Evaluation]
    CLIENT[Platform Consumers]

    LG --> API
    APP --> API
    API --> PROC
    PROC --> STORE
    STORE --> QUERY
    QUERY --> CLIENT
    STORE --> SLO
    SLO --> CLIENT
```

## 2. Telemetry data flow

```mermaid
flowchart TD
    A[Load Generator] --> B[Telemetry Request]
    B --> C[Ingestion Endpoint]
    C --> D{Valid Request?}
    D -- No --> E[Reject / Return Error]
    D -- Yes --> F[Process Event]
    F --> G{Capacity Available?}
    G -- No --> H[Apply Back-pressure / Reject]
    G -- Yes --> I[Bounded Telemetry Store]
    I --> J[Query Service]
    J --> K[Product Team / Consumer]
    I --> L[SLO Evaluator]
    L --> M[Calculate SLI]
    M --> N[Compare Against SLO]
    N --> O[Error Budget Status]
```

## 3. Ingestion request sequence

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant API as Ingestion API
    participant Processor
    participant Store as Bounded Store

    Client->>API: POST telemetry
    API->>API: Validate request
    alt Invalid request
        API-->>Client: 400 Bad Request
    else Valid request
        API->>Processor: Submit event
        Processor->>Store: Attempt bounded write
        alt Capacity available
            Store-->>Processor: Accepted
            Processor-->>API: Processing result
            API-->>Client: Success response
        else Capacity exhausted
            Store-->>Processor: Capacity unavailable
            Processor-->>API: Back-pressure outcome
            API-->>Client: Overload response
        end
    end
```

## 4. SLO and error-budget evaluation

```mermaid
flowchart TD
    A[Select Service and Evaluation Window]
    B[Retrieve Relevant Telemetry]
    C[Calculate Service Level Indicator]
    D[Compare SLI with SLO Target]
    E[Calculate Allowed Failure Budget]
    F[Calculate Consumed Error Budget]
    G{Budget Exhausted?}
    H[Within Reliability Budget]
    I[Budget Exhausted / Reliability Risk]

    A --> B --> C --> D --> E --> F --> G
    G -- No --> H
    G -- Yes --> I
```

## Architectural principles

- **Explicit resource limits:** bounded storage makes capacity constraints visible.
- **Controlled overload:** back-pressure defines how the system behaves when consumers cannot keep up.
- **Separation of concerns:** ingestion, storage, querying, traffic generation, and SLO evaluation have distinct responsibilities.
- **Stable consumer boundary:** product teams interact through a query-facing API rather than storage internals.
- **Measurable reliability:** SLOs and error budgets turn telemetry into operational signals.

> These diagrams describe the intended conceptual design. Revise them whenever the implementation's module boundaries, request lifecycle, or storage behaviour differ.
