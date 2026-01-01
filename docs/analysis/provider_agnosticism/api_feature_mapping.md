# API Feature Mapping: Claude vs. Gemini

## 1. Introduction

This document provides a high-level mapping of the key features of the Anthropic Claude API and the Google Gemini API. The goal is to identify the similarities and differences that would need to be handled by an abstraction layer (either in a plugin or in a CLI) to achieve provider-agnosticism.

*Note: This is based on publicly available documentation and may not be exhaustive. The exact feature set can change rapidly.*

## 2. Core API Features Comparison

| Feature                 | Anthropic Claude                                | Google Gemini                                       | Abstraction Notes                                                                                             |
| ----------------------- | ----------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Primary Endpoint**    | `v1/messages`                                   | `v1beta/models/{model}:generateContent`             | The core abstraction is straightforward: a function that takes a list of messages and returns a response.     |
| **Message Structure**   | `role` (`user` or `assistant`) and `content`.   | `role` (`user` or `model`) and `parts`.             | The role names (`assistant` vs. `model`) need to be normalized. The `content` vs. `parts` structure is similar. |
| **Tool Use / Function Calling** | Supported. Defined in a `tools` parameter.      | Supported. Defined in a `tools` parameter.      | The structure of the tool definitions is very similar and can be easily mapped. Both use a JSON schema-like format. |
| **Streaming Response**  | Supported.                                      | Supported.                                          | Both APIs provide a streaming interface. The abstraction would need to normalize the format of the streamed chunks. |
| **System Prompt**       | Supported via a `system` parameter at the top level. | Supported by including a message with `role: system`. | The placement of the system prompt is different, but the concept is the same. The abstraction layer would handle this. |
| **Model Naming**        | e.g., `claude-3-opus-20240229`                   | e.g., `gemini-1.5-pro-latest`                         | The abstraction layer would need a mapping of generic model capabilities (e.g., "best," "fastest") to the specific model names. |

## 3. Key Differences to Handle

-   **Tool Call Response Format:**
    -   **Claude:** When the model wants to call a tool, the response message will have a `stop_reason` of `tool_use` and the `content` will be an array of `tool_use` blocks. The developer must then execute the tools and send back a new message with `role: user` and a `content` block of type `tool_result`.
    -   **Gemini:** The model's response will contain a `functionCall` part. The developer must then execute the tool and send back a new message with `role: user` and a `toolResponse` part.
    -   **Abstraction:** The core logic is the same (call tool, return result), but the naming and structure of the JSON objects are different. The abstraction layer would need to handle this mapping.

-   **Error Handling:**
    -   The specific error codes and messages for things like rate limits, content filtering, and invalid requests are different for each API. The abstraction layer would need to catch the provider-specific errors and normalize them into a standard set of error types.

-   **Advanced Features:**
    -   **Gemini:** Has features like grounding and more complex multi-modal capabilities that may not have a direct equivalent in the Claude API.
    -   **Claude:** May have its own unique features and optimizations.
    -   **Abstraction:** The abstraction layer would need to decide whether to support these advanced, provider-specific features. It could either ignore them (a "lowest common denominator" approach) or provide a way for plugins to access them through a provider-specific extension point.

## 4. Conclusion

While the core functionalities of the Claude and Gemini APIs are very similar, there are enough differences in the details (message structure, tool call format, error codes) to make a direct, drop-in replacement impossible.

However, the concepts are so closely aligned that creating an effective abstraction layer is very feasible. The primary task of such a layer would be to normalize the data structures for messages, tool calls, and errors, allowing the core agent logic to operate on a single, standardized data model.
