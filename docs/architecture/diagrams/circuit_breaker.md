# Diagrams for the Circuit Breaker

This document provides a set of diagrams to visualize the Circuit Breaker feature of the Ralph tool.

## 1. Activity Diagram

This diagram shows the process of recording a loop result and determining whether to halt execution.

```mermaid
graph TD
    A[Loop completes] --> B[record_loop_result()];
    B --> C{Analysis shows error?};
    C -- Yes --> D{Analysis shows no progress?};
    C -- No --> E[Reset failure counters];
    E --> F{Halt execution?};
    D -- Yes --> G[Increment consecutive error and no progress counters];
    D -- No --> H[Increment consecutive error counter];
    G --> F;
    H --> F;
    F -- No --> I[Continue];
    I --> J[End];
    F -- Yes --> K[Halt];
    K --> J;
```

## 2. Sequence Diagram

This diagram shows the interactions that lead to the circuit breaker opening.

```mermaid
sequenceDiagram
    participant CoreLoop
    participant CircuitBreaker
    participant ResponseAnalyzer

    CoreLoop->>ResponseAnalyzer: analyze_response()
    ResponseAnalyzer-->>CoreLoop: Analysis with errors
    CoreLoop->>CircuitBreaker: record_loop_result(analysis)
    CircuitBreaker->>CircuitBreaker: Update internal failure counts
    CoreLoop->>CircuitBreaker: should_halt_execution()
    alt Failure threshold not reached
        CircuitBreaker-->>CoreLoop: false
    else Failure threshold reached
        CircuitBreaker-->>CoreLoop: true (Open the circuit)
    end
```

## 3. State Diagram

This diagram illustrates the three states of the Circuit Breaker and the transitions between them.

```mermaid
stateDiagram-v2
    [*] --> CLOSED: Initial State
    CLOSED --> OPEN: consecutive_errors >= 5 OR no_progress_loops >= 3
    OPEN --> HALF_OPEN: after timeout (e.g., 5 minutes)
    HALF_OPEN --> CLOSED: on successful trial run
    HALF_OPEN --> OPEN: on failed trial run
```
