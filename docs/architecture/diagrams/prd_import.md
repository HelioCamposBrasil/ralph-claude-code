# Diagrams for PRD Import Feature

This document provides a set of diagrams to visualize the PRD (Product Requirement Document) Import feature of the Ralph tool.

## 1. Activity Diagram

This diagram shows the flow of activities when the `ralph-import` command is executed.

```mermaid
graph TD
    A[User runs `ralph-import prd.md my-project`] --> B{Validate input file exists};
    B -- Yes --> C[Create project directory `my-project`];
    C --> D[Construct a prompt to convert PRD];
    D --> E[Execute Claude Code with conversion prompt];
    E --> F{Conversion successful?};
    F -- Yes --> G[Save AI output to PROMPT.md and @fix_plan.md];
    G --> H[Create standard Ralph project structure];
    H --> I[End];
    F -- No --> J[Report error to user];
    J --> I;
```

## 2. Sequence Diagram

This diagram illustrates the interactions between the user, the import script, and the Claude Code CLI.

```mermaid
sequenceDiagram
    participant User
    participant ImportScript as ralph-import
    participant ClaudeCLI as Claude Code CLI
    participant FileSystem

    User->>ImportScript: executes `ralph-import prd.md`
    ImportScript->>FileSystem: Read `prd.md`
    FileSystem-->>ImportScript: PRD content
    ImportScript->>ImportScript: Construct conversion prompt
    ImportScript->>ClaudeCLI: execute(conversion_prompt)
    ClaudeCLI-->>ImportScript: Converted content
    ImportScript->>FileSystem: WRITE PROMPT.md
    ImportScript->>FileSystem: WRITE @fix_plan.md
    ImportScript->>FileSystem: Create project directories (specs/, src/, etc.)
```

## 3. Data Flow Diagram

This diagram focuses on how the data is transformed from the initial PRD to the final Ralph project files.

```mermaid
graph TD
    A[Input PRD (.md, .txt, etc.)] --> B[ralph-import script];
    B -- "sends as context in a prompt" --> C[Claude Code CLI];
    C -- "returns structured text" --> D{AI-Generated Content};
    D -- "instructions part" --> E["PROMPT.md"];
    D -- "task list part" --> F["@fix_plan.md"];
    B -- "creates" --> G["Standard Directories (src/, specs/, etc.)"];
```
