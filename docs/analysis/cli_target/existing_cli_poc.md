# Proof-of-Concept for a Ralph-like Gemini CLI Plugin

## 1. Introduction

This document provides a concrete, proof-of-concept (PoC) implementation for a Ralph-like autonomous agent plugin for the existing Google Gemini CLI. It builds upon the "hybrid" strategy outlined in the analysis, using a combination of the "Extensions" and "Hooks" systems.

The goal of this PoC is to provide a near-complete code example that can serve as a starting point for a real implementation.

## 2. Directory Structure

The plugin, named `gemini-ralph-agent`, would have the following directory structure:

```
gemini-ralph-agent/
├── gemini-extension.json  # The extension manifest
├── hooks/
│   ├── on_command_ralph.js # Hook to start/stop the loop
│   └── on_after_model.js   # The core agentic loop logic
├── lib/
│   ├── analyzer.js         # Response analysis logic
│   └── state.js            # State management (rate limit, etc.)
├── package.json
└── README.md
```

## 3. `gemini-extension.json` (The Manifest)

This file defines the extension and registers the custom command and hooks.

```json
{
  "name": "ralph-agent",
  "version": "0.1.0",
  "description": "An autonomous agent for Gemini CLI, inspired by Ralph.",
  "commands": [
    {
      "name": "ralph",
      "description": "Control the Ralph autonomous agent.",
      "subcommands": [
        {
          "name": "start",
          "description": "Start the autonomous development loop.",
          "prompt": "Starting the Ralph agent. The first task is to review the project goals."
        },
        {
          "name": "stop",
          "description": "Stop the autonomous development loop.",
          "prompt": "Stopping the Ralph agent."
        }
      ]
    }
  ],
  "hooks": {
    "AfterCommand": [
      {
        "matcher": "ralph",
        "hooks": [
          {
            "name": "ralph-controller",
            "type": "command",
            "command": "node ${extensionPath}${/}hooks${/}on_command_ralph.js",
            "description": "Manages the state of the Ralph agent loop."
          }
        ]
      }
    ],
    "AfterModel": [
      {
        "matcher": "*",
        "hooks": [
          {
            "name": "ralph-loop-handler",
            "type": "command",
            "command": "node ${extensionPath}${/}hooks${/}on_after_model.js",
            "description": "The core logic of the Ralph agentic loop."
          }
        ]
      }
    ]
  }
}
```

## 4. `hooks/on_command_ralph.js` (Loop Controller)

This hook is responsible for starting and stopping the loop. It does this by writing to a state file.

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const { getState, saveState } = require('../lib/state');

async function main() {
  const input = JSON.parse(await readStdin());
  const command = input.command_name; // e.g., "ralph:start"

  let state = getState();

  if (command === 'ralph:start') {
    state.isRunning = true;
    state.loopCount = 0;
    saveState(state);
    // The initial prompt is handled by the manifest
  } else if (command === 'ralph:stop') {
    state.isRunning = false;
    saveState(state);
  }

  // This hook doesn't modify the output, so we return an empty object.
  console.log(JSON.stringify({}));
}

function readStdin() {
  return new Promise((resolve) => {
    const chunks = [];
    process.stdin.on('data', (chunk) => chunks.push(chunk));
    process.stdin.on('end', () => resolve(Buffer.concat(chunks).toString()));
  });
}

main().catch(console.error);
```

## 5. `hooks/on_after_model.js` (The Agentic Loop)

This is the core of the plugin. It runs after every model response, checks if the Ralph agent is active, and if so, analyzes the response and programmatically triggers the next prompt.

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const { getState, saveState } = require('../lib/state');
const { analyzeResponse } = require('../lib/analyzer');

async function main() {
  let state = getState();

  if (!state.isRunning) {
    // If Ralph is not running, do nothing.
    console.log(JSON.stringify({}));
    return;
  }

  const input = JSON.parse(await readStdin());
  const modelResponse = input.llm_response.candidates[0]?.content?.parts[0]?.text || '';

  // 1. Analyze the response
  const analysis = analyzeResponse(modelResponse, state);

  // 2. Update the state
  state.loopCount++;
  // ... update rate limiter, circuit breaker, etc.
  saveState(state);

  // 3. Decide on the next action
  if (analysis.isComplete) {
    state.isRunning = false;
    saveState(state);
    console.log(JSON.stringify({
      systemMessage: "✅ Ralph has completed the task."
    }));
  } else {
    // This is the key to creating the loop: we return a `reprompt` action.
    // The Gemini CLI needs to support a mechanism like this for a seamless loop.
    console.log(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: "AfterModel",
        // Hypothetical feature: instruct the CLI to immediately reprompt.
        nextAction: "REPROMPT",
        nextPrompt: analysis.nextPrompt
      },
      systemMessage: `🔁 Ralph Loop #${state.loopCount}: ${analysis.summary}`
    }));
  }
}

function readStdin() {
  // ... (same as in on_command_ralph.js)
}

main().catch(console.error);
```

## 6. `lib/state.js` and `lib/analyzer.js`

These files would contain the business logic, ported from the original Ralph shell scripts to JavaScript.

-   **`state.js`:** Would manage reading and writing the state from a JSON file (e.g., `.gemini/ralph_state.json`). It would include functions for managing the rate limiter, circuit breaker state, etc.
-   **`analyzer.js`:** Would contain the logic for parsing the model's response, looking for completion signals, errors, and test-only loops.

## 7. Key Assumption and Required CLI Feature

This PoC relies on a key assumption: **the Gemini CLI must provide a mechanism for a hook to programmatically trigger a new prompt.**

In the `on_after_model.js` example, this is represented by the hypothetical `nextAction: "REPROMPT"` field. If the Gemini CLI does not support such a feature, a less elegant workaround would be required, such as having the hook script call `gemini ...` in headless mode, which would create a new process for each loop iteration. A first-class "reprompt" action is essential for a clean and efficient implementation.
