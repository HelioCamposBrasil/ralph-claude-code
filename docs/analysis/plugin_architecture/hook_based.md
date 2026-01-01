# Plugin Architecture Proposal: Hook-Based System

## 1. Introduction

This document proposes a hook-based plugin architecture for a Gemini CLI. This model is heavily inspired by the existing Gemini CLI's hook system but expands upon it to better accommodate autonomous agents like Ralph. The core principle of this architecture is to provide specific, well-defined "interception points" in the CLI's lifecycle where plugins can inject their own logic.

## 2. Core Concepts

-   **Lifecycle Events:** The CLI's execution flow is divided into a series of distinct lifecycle events (e.g., `SessionStart`, `BeforePrompt`, `AfterModel`).
-   **Hooks:** Plugins are essentially collections of scripts or functions (hooks) that are registered to run when these lifecycle events are fired.
-   **Data Pipeline:** Each event is associated with a data object. Hooks can read and modify this data object, and the modified data is then passed to the next hook in the chain and, ultimately, to the next stage of the CLI's lifecycle.

## 3. Architecture Diagram

```mermaid
graph TD
    A[CLI Core] --> B{Fires Event};
    B --> C[Hook Runner];
    C --> D{Find registered hooks for event};
    D --> E[Execute Hook 1];
    E --> F[Execute Hook 2];
    F --> G[...];
    G --> H[Return modified data to Core];
    H --> I[Continue execution];

    subgraph "Plugin A"
        direction LR
        PA1[Hook for Event X]
    end

    subgraph "Plugin B"
        direction LR
        PB1[Hook for Event X]
        PB2[Hook for Event Y]
    end

    C -- reads from --> Plugin A;
    C -- reads from --> Plugin B;
```

## 4. Detailed Implementation

### 4.1. Lifecycle Events

A comprehensive set of lifecycle events is crucial for a powerful hook-based system. The following events would be necessary to support a Ralph-like plugin:

-   `on_session_start`: When the CLI is first launched.
-   `on_command(command_name, args)`: When a custom command is executed. This would be used to trigger the start of the autonomous loop.
-   `on_before_prompt(prompt)`: Before a prompt is sent to the model. Allows for last-minute context injection.
-   `on_after_model(response)`: After the model returns a response. The core of the autonomous loop would reside here.
-   `on_before_tool_call(tool_name, args)`: Before a tool is executed.
-   `on_after_tool_call(tool_name, result)`: After a tool is executed.
-   `on_session_end`: When the CLI is about to exit.

### 4.2. Hook Execution

-   **Chaining:** Hooks for the same event are executed in a predefined order.
-   **Data Modification:** Each hook receives a context object and can return a modified version of it. For example, an `on_before_prompt` hook could add a preamble to the prompt string.
-   **Control Flow:** To enable an autonomous loop, the `on_after_model` hook would need to be able to return a special value that instructs the CLI to re-run the loop with a new prompt instead of waiting for user input.

```json
// Example of the data object for on_after_model
{
  "model_response": "...",
  "next_action": "WAIT_FOR_USER", // Default
  "next_prompt": null
}

// A Ralph plugin's hook could return:
{
  "model_response": "...",
  "next_action": "REPROMPT",
  "next_prompt": "The next step is to..."
}
```

## 5. Pros and Cons

### 5.1. Pros

-   **Simple to Implement:** The logic for a hook-based system is relatively straightforward.
-   **Easy for Plugin Developers:** Writing a hook is as simple as writing a script that accepts and returns JSON.
-   **Decoupled:** Plugins are decoupled from the CLI's core logic, which makes the system more stable.

### 5.2. Cons

-   **Limited Control:** Plugins can only react to predefined events. They cannot fundamentally alter the core execution flow outside of the specific mechanisms provided (like the `next_action` field).
-   **Implicit Looping:** Creating an autonomous loop can feel "hacky," as it relies on a hook repeatedly telling the CLI to re-prompt. The core of the CLI is not a true agentic loop.
-   **State Management:** The CLI would need to provide a separate mechanism for plugins to manage state between hook executions, as hooks themselves are stateless.

## 6. Conclusion

A hook-based architecture is a viable and practical way to enable a Ralph-like plugin. It provides a good balance of power and simplicity. However, it can feel less elegant for autonomous agents, as the core of the CLI is not designed around the concept of a long-running, agentic loop.
