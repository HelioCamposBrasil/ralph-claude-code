# Diagrams for the Rate Limiter

This document provides a set of diagrams to visualize the Rate Limiter feature of the Ralph tool.

## 1. Activity Diagram

This diagram shows the logic flow when a request to make an API call is handled.

```mermaid
graph TD
    A[Request to make API call] --> B{Call tracking initialized?};
    B -- No --> C[Initialize call count and timestamp];
    C --> D{Is current hour > last reset hour?};
    B -- Yes --> D;
    D -- Yes --> E[Reset call count and update timestamp];
    D -- No --> F{Call count < limit?};
    E --> F;
    F -- Yes --> G[Allow API call and increment count];
    F -- No --> H[Deny API call];
    H --> I[Wait for next hour];
    I --> D;
    G --> J[End];
```

## 2. Sequence Diagram

This diagram shows the interactions when the core loop checks the rate limiter.

```mermaid
sequenceDiagram
    participant CoreLoop
    participant RateLimiter
    participant FileSystem as ~/.ralph/

    CoreLoop->>RateLimiter: can_make_call()
    RateLimiter->>FileSystem: Read .ralph_timestamp
    FileSystem-->>RateLimiter: timestamp
    RateLimiter->>RateLimiter: Compare with current time
    alt Time window has reset
        RateLimiter->>FileSystem: Write new timestamp
        RateLimiter->>FileSystem: Reset .ralph_call_count to 1
        FileSystem-->>RateLimiter: success
        RateLimiter-->>CoreLoop: true
    else Call count < limit
        RateLimiter->>FileSystem: Read .ralph_call_count
        FileSystem-->>RateLimiter: count
        RateLimiter->>FileSystem: Increment .ralph_call_count
        FileSystem-->>RateLimiter: success
        RateLimiter-->>CoreLoop: true
    else Call count >= limit
        RateLimiter-->>CoreLoop: false
    end
```

## 3. State Diagram

This diagram represents the states of the Rate Limiter within an hourly window.

```mermaid
stateDiagram-v2
    [*] --> OK: Start of hour
    OK --> OK: On API call (count < limit)
    OK --> Throttled: On API call (count == limit)
    Throttled --> OK: On hour reset
```
