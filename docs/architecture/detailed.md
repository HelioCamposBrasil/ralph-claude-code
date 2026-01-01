# Detailed Architecture of Ralph for Claude Code

## 1. Introduction

This document provides a detailed, code-level examination of the Ralph for Claude Code tool. It builds on the high-level architecture, diving into specific scripts, functions, and implementation details.

## 2. Core Scripts and Execution Flow

The primary logic of Ralph is contained within a set of shell scripts. The execution flow is orchestrated by the main `ralph` command, which is a wrapper that ultimately calls `ralph_loop.sh`.

### 2.1. `install.sh`

This script is responsible for the one-time installation of Ralph.
-   **Creates Directories:** It creates `~/.ralph` and `~/.local/bin`.
-   **Copies Files:** It copies the core scripts and the `lib/` directory into `~/.ralph`.
-   **Creates Symlinks:** It creates symlinks in `~/.local/bin` to make the `ralph`, `ralph-setup`, and `ralph-monitor` commands globally available.
-   **Dependency Checks:** It verifies that `git`, `jq`, and `tmux` are installed.

### 2.2. `ralph` (Wrapper Script)

This is the main entry point for the user.
-   **Argument Parsing:** It parses command-line arguments like `--monitor`, `--calls`, `--prompt`, etc.
-   **tmux Session Management:** If the `--monitor` flag is used, it calls `setup_tmux_session` to create a new `tmux` session.
-   **Launches Core Loop:** It executes `ralph_loop.sh`, passing along any parsed arguments.

### 2.3. `ralph_loop.sh` (Core Loop)

This is the engine of the autonomous development cycle.

#### Key Functions:

-   `main()`: The main execution block. It initializes the environment, then enters a `while` loop that continues until an exit condition is met.
-   `init_call_tracking()`: Initializes or resets the API call counter and timestamp.
-   `can_make_call()`: Checks if the rate limit has been exceeded.
-   `wait_for_reset()`: If the rate limit is hit, this function pauses execution until the next hour.
-   `execute_claude_code()`: Constructs and executes the command to run the Claude Code CLI.
-   `update_status()`: Writes the current status of the loop (loop count, API calls, etc.) to `logs/status.json`.
-   `should_exit_gracefully()`: Checks for exit conditions and determines if the loop should terminate.

#### Sequence Diagram of a Single Loop Iteration:

```mermaid
sequenceDiagram
    participant User
    participant ralph
    participant ralph_loop.sh
    participant ClaudeCodeCLI
    participant ResponseAnalyzer
    participant CircuitBreaker

    User->>ralph: executes `ralph --monitor`
    ralph->>ralph_loop.sh: starts the main loop
    loop until exit
        ralph_loop.sh->>ralph_loop.sh: init_call_tracking()
        ralph_loop.sh->>ralph_loop.sh: can_make_call()
        alt Rate limit exceeded
            ralph_loop.sh->>ralph_loop.sh: wait_for_reset()
        end
        ralph_loop.sh->>CircuitBreaker: should_halt_execution()
        alt Circuit breaker is open
            ralph_loop.sh->>ralph_loop.sh: Halts execution
        end
        ralph_loop.sh->>ClaudeCodeCLI: execute_claude_code()
        ClaudeCodeCLI-->>ralph_loop.sh: returns AI output
        ralph_loop.sh->>ResponseAnalyzer: analyze_response(output)
        ResponseAnalyzer-->>ralph_loop.sh: returns analysis
        ralph_loop.sh->>CircuitBreaker: record_loop_result(analysis)
        ralph_loop.sh->>ralph_loop.sh: update_status()
        ralph_loop.sh->>ralph_loop.sh: should_exit_gracefully()
        alt Exit conditions met
            ralph_loop.sh->>ralph_loop.sh: Exits loop
        end
    end
```

## 3. The `lib/` Directory

The `lib/` directory contains modularized shell scripts that provide specific functionalities to the core loop.

### 3.1. `response_analyzer.sh`

This library is responsible for interpreting the output from the Claude Code CLI.

-   `analyze_response()`: The main function. It takes the AI's output as input and performs a series of checks.
-   `detect_completion_signals()`: Looks for keywords and phrases that indicate the AI believes the task is complete.
-   `detect_errors()`: Uses a two-stage process to identify errors. It first looks for common error patterns and then filters out false positives (e.g., the string "error" inside a JSON field).
-   `detect_stuck_loop()`: Compares the errors from the current loop to previous loops to see if the AI is stuck on the same problem.
-   `update_exit_signals()`: Records the findings of the analysis in the `.exit_signals` file.

### 3.2. `circuit_breaker.sh`

This library implements the Circuit Breaker pattern to prevent runaway loops.

-   **States:** The circuit breaker can be in one of three states: `CLOSED` (normal operation), `OPEN` (execution is halted), or `HALF_OPEN` (a trial run is allowed to see if the issue is resolved).
-   `init_circuit_breaker()`: Initializes the state of the circuit breaker.
-   `record_loop_result()`: Takes the analysis from the `response_analyzer` and updates the circuit breaker's internal state. It increments counters for consecutive errors or loops with no progress.
-   `should_halt_execution()`: Checks the current state and determines if the main loop should be halted.

#### State Diagram of the Circuit Breaker:

```mermaid
stateDiagram-v2
    [*] --> CLOSED: Initial State
    CLOSED --> OPEN: on consecutive_errors >= 5 OR no_progress_loops >= 3
    OPEN --> HALF_OPEN: after timeout
    HALF_OPEN --> CLOSED: on successful trial run
    HALF_OPEN --> OPEN: on failed trial run
```

### 3.3. `date_utils.sh`

This is a utility library that provides cross-platform compatibility for date and time operations, handling the differences between GNU and BSD (macOS) `date` commands.

## 4. Data and State Management

Ralph uses a combination of files in the project directory and the `~/.ralph` directory to manage state.

-   `~/.ralph/.ralph_call_count`: Stores the number of API calls made in the current hour.
-   `~/.ralph/.ralph_timestamp`: Stores the timestamp of the last rate limit reset.
-   `logs/status.json`: A JSON file that contains the current state of the loop. This is read by `ralph-monitor`.
-   `logs/ralph.log`: A detailed log of all operations.
-   `.exit_signals`: A JSON file that stores the signals (completion, errors, etc.) detected by the `response_analyzer`.
-   `@fix_plan.md`: The user-defined list of tasks. The `should_exit_gracefully` function checks this file for completion.

## 5. `ralph-setup` and `ralph-import`

These are project initialization scripts.

-   `ralph-setup`: Creates a new, blank Ralph project with the standard directory structure and template files.
-   `ralph-import`: Takes an existing document (e.g., a PRD in Markdown) and uses Claude Code to convert it into the `PROMPT.md` and `@fix_plan.md` files for a new project. This script is a good example of how Ralph uses AI to bootstrap its own configuration.
