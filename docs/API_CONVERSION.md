# API Format Conversion Guide

## Overview

This document explains how the New API Gateway supports automatic conversion between OpenAI's Chat Completions API and Responses API formats.

## Background

### The Problem

OpenAI has introduced two different API formats:

1. **Chat Completions API** (`/v1/chat/completions`) - The traditional, widely-used format
2. **Responses API** (`/v1/responses`) - A newer unified API with enhanced capabilities

This creates a compatibility challenge:
- **Old LLM providers** only support Chat Completions API
- **New client applications** may only support Responses API
- Projects need a way to bridge these two formats

### The Solution

New API Gateway now supports **automatic format conversion**, allowing:
- Clients using Responses API to communicate with providers that only support Chat Completions API
- (Future) Clients using Chat Completions API to access providers using Responses API

## API Format Differences

### Request Format Comparison

| Feature | Chat Completions | Responses |
|---------|-----------------|-----------|
| **Endpoint** | `/v1/chat/completions` | `/v1/responses` |
| **Messages** | `messages` array | `input` JSON (can contain messages) |
| **System Message** | In `messages` array with role="system" | `instructions` field |
| **Max Tokens** | `max_tokens` or `max_completion_tokens` | `max_output_tokens` |
| **Reasoning** | `reasoning_effort` string | `reasoning` object with `effort` |
| **Built-in Tools** | Not supported | `tools` with built-in types |

### Response Format Comparison

| Feature | Chat Completions | Responses |
|---------|-----------------|-----------|
| **Result** | `choices` array | `output` array |
| **Usage** | `prompt_tokens`, `completion_tokens` | `input_tokens`, `output_tokens` |
| **Status** | Implicit (HTTP status) | Explicit `status` field |

## Conversion Functions

The following conversion functions are available in `service/convert.go`:

### Request Conversions

#### 1. `ChatCompletionsToResponsesRequest()`
Converts a Chat Completions request to Responses API format.

```go
func ChatCompletionsToResponsesRequest(chatRequest *dto.GeneralOpenAIRequest) (*dto.OpenAIResponsesRequest, error)
```

**Conversion Details:**
- `messages` → `input` (non-system messages only)
- System messages → `instructions`
- `max_tokens`/`max_completion_tokens` → `max_output_tokens`
- `reasoning_effort` → `reasoning.effort`
- Tools, temperature, top_p copied directly

#### 2. `ResponsesToChatCompletionsRequest()`
Converts a Responses API request to Chat Completions format.

```go
func ResponsesToChatCompletionsRequest(responsesReq *dto.OpenAIResponsesRequest) (*dto.GeneralOpenAIRequest, error)
```

**Conversion Details:**
- `instructions` → System message in `messages` array
- `input` → Messages array (unwrapped)
- `max_output_tokens` → `max_completion_tokens`
- `reasoning.effort` → `reasoning_effort`

### Response Conversions

#### 3. `ChatCompletionsToResponsesResponse()`
Converts a Chat Completions response to Responses API format.

```go
func ChatCompletionsToResponsesResponse(chatResp *dto.OpenAITextResponse) (*dto.OpenAIResponsesResponse, error)
```

**Conversion Details:**
- `choices` → `output` array
- `prompt_tokens`/`completion_tokens` → `input_tokens`/`output_tokens`
- Adds `status: "completed"` field

#### 4. `ResponsesToChatCompletionsResponse()`
Converts a Responses API response to Chat Completions format.

```go
func ResponsesToChatCompletionsResponse(responsesResp *dto.OpenAIResponsesResponse) (*dto.OpenAITextResponse, error)
```

**Conversion Details:**
- `output` → `choices` array
- `input_tokens`/`output_tokens` → `prompt_tokens`/`completion_tokens`
- Sets `finish_reason: "stop"`

## Usage Examples

### Example 1: Client Uses Responses API, Provider Uses Chat Completions

**Scenario:** A new application uses OpenAI's Responses API, but your backend channel only supports the Chat Completions format.

**Client Request** (Responses API format):
```json
{
  "model": "gpt-4",
  "instructions": "You are a helpful assistant.",
  "input": [
    {"role": "user", "content": "Hello, how are you?"}
  ],
  "max_output_tokens": 100,
  "temperature": 0.7
}
```

**After Conversion** (Chat Completions format sent to backend):
```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello, how are you?"}
  ],
  "max_completion_tokens": 100,
  "temperature": 0.7
}
```

**Backend Response** (Chat Completions format):
```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "model": "gpt-4",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "I'm doing well, thank you!"
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 10,
    "total_tokens": 30
  }
}
```

**After Conversion** (Responses API format returned to client):
```json
{
  "id": "chatcmpl-123",
  "object": "response",
  "model": "gpt-4",
  "status": "completed",
  "output": [{
    "index": 0,
    "type": "message",
    "content": "I'm doing well, thank you!"
  }],
  "usage": {
    "input_tokens": 20,
    "output_tokens": 10,
    "total_tokens": 30
  }
}
```

## Implementation Status

### ✅ Completed
- [x] Request conversion functions (both directions)
- [x] Response conversion functions (both directions)
- [x] Documentation

### 🚧 In Progress / Future Work
- [ ] Integration with relay handlers (automatic conversion)
- [ ] Channel configuration flag for format support
- [ ] Stream response conversion
- [ ] Built-in tools handling (Responses → Chat Completions)
- [ ] Unit tests
- [ ] Integration tests

## Limitations

### Current Limitations
1. **Streaming**: Stream response conversion not yet implemented
2. **Built-in Tools**: Responses API built-in tools (web search, file search) cannot be converted to Chat Completions format
3. **Advanced Features**: Some Responses API advanced features may be lost in conversion

### Future Enhancements
1. Add automatic detection and conversion in relay handlers
2. Support streaming conversion
3. Add channel setting to declare native API format
4. Provide fallback behavior for unsupported features

## Testing

To test the conversion functions:

```go
import (
    "github.com/QuantumNous/new-api/dto"
    "github.com/QuantumNous/new-api/service"
)

// Test Chat Completions → Responses conversion
chatReq := &dto.GeneralOpenAIRequest{
    Model: "gpt-4",
    Messages: []dto.Message{
        {Role: "system", Content: "You are helpful"},
        {Role: "user", Content: "Hello"},
    },
    MaxCompletionTokens: 100,
}

responsesReq, err := service.ChatCompletionsToResponsesRequest(chatReq)
// responsesReq now contains the converted request

// Test Responses → Chat Completions conversion
responsesReq2 := &dto.OpenAIResponsesRequest{
    Model: "gpt-4",
    Instructions: json.RawMessage(`"You are helpful"`),
    Input: json.RawMessage(`[{"role":"user","content":"Hello"}]`),
    MaxOutputTokens: 100,
}

chatReq2, err := service.ResponsesToChatCompletionsRequest(responsesReq2)
// chatReq2 now contains the converted request
```

## Contributing

If you'd like to contribute to improving the API conversion functionality:

1. Add support for more field mappings
2. Implement streaming conversion
3. Add comprehensive tests
4. Improve error handling for edge cases
5. Document additional use cases

## Related Documentation

- [OpenAI Chat Completions API Documentation](https://platform.openai.com/docs/api-reference/chat)
- [OpenAI Responses API Documentation](https://platform.openai.com/docs/api-reference/responses)
- [Channel Configuration Guide](./channel/other_setting.md)
