# Diagrams for the Core Autonomous Loop

This document provides a set of diagrams to visualize the Core Autonomous Loop of the Ralph tool from different perspectives.

## 1. Activity Diagram

This diagram shows the flow of activities within a single iteration of the loop.

```mermaid
graph TD
    A[Start Loop Iteration] --> B{Rate Limit OK?};
    B -- Yes --> C{Circuit Breaker Closed?};
    B -- No --> D[Wait for Reset];
    D --> A;
    C -- Yes --> E[Execute Claude Code];
    C -- No --> F[Halt Execution];
    F --> G[End];
    E --> H[Analyze Response];
    H --> I[Update State & Logs];
    I --> J{Exit Conditions Met?};
    J -- Yes --> K[Graceful Exit];
    K --> G;
    J -- No --> A;
```

## 2. Sequence Diagram

This diagram illustrates the interactions between the different components of the system over time during a loop iteration.

```mermaid
sequenceDiagram
    participant Loop as Core Loop
    participant RateLimiter
    participant CircuitBreaker
    participant ClaudeCLI as Claude Code CLI
    participant Analyzer as Response Analyzer

    Loop->>RateLimiter: can_make_call()
    RateLimiter-->>Loop: true
    Loop->>CircuitBreaker: should_halt_execution()
    CircuitBreaker-->>Loop: false
    Loop->>ClaudeCLI: execute()
    ClaudeCLI-->>Loop: AI Output
    Loop->>Analyzer: analyze_response(AI Output)
    Analyzer-->>Loop: Analysis Results
    Loop->>Loop: update_status(Analysis Results)
    Loop->>Loop: check_exit_conditions()
```

## 3. State Diagram

This diagram represents the different states that the Core Loop can be in.

```mermaid
stateDiagram-v2
    [*] --> Initializing
    Initializing --> Running: On successful setup
    Running --> Paused: On rate limit exceeded
    Paused --> Running: On rate limit reset
    Running --> Halted: On circuit breaker open
    Running --> Exiting: On exit conditions met
    Halted --> [*]
    Exiting --> [*]
```
