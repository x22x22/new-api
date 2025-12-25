# API 格式转换指南

## 概述

本文档说明 New API Gateway 如何支持 OpenAI Chat Completions API 和 Responses API 格式之间的自动转换。

## 背景

### 问题描述

OpenAI 引入了两种不同的 API 格式：

1. **Chat Completions API** (`/v1/chat/completions`) - 传统的、广泛使用的格式
2. **Responses API** (`/v1/responses`) - 新的统一 API，具有增强功能

这带来了兼容性挑战：
- **旧的大模型服务提供商**仅支持 Chat Completions API
- **新的客户端应用**可能仅支持 Responses API
- 项目需要一种方法来桥接这两种格式

### 解决方案

New API Gateway 现在支持**自动格式转换**，允许：
- 使用 Responses API 的客户端与仅支持 Chat Completions API 的提供商通信
- （未来）使用 Chat Completions API 的客户端访问使用 Responses API 的提供商

## API 格式差异

### 请求格式对比

| 功能 | Chat Completions | Responses |
|------|-----------------|-----------|
| **端点** | `/v1/chat/completions` | `/v1/responses` |
| **消息** | `messages` 数组 | `input` JSON（可包含消息） |
| **系统消息** | `messages` 数组中 role="system" | `instructions` 字段 |
| **最大令牌数** | `max_tokens` 或 `max_completion_tokens` | `max_output_tokens` |
| **推理** | `reasoning_effort` 字符串 | `reasoning` 对象，包含 `effort` |
| **内置工具** | 不支持 | `tools` 支持内置类型 |

### 响应格式对比

| 功能 | Chat Completions | Responses |
|------|-----------------|-----------|
| **结果** | `choices` 数组 | `output` 数组 |
| **使用量** | `prompt_tokens`, `completion_tokens` | `input_tokens`, `output_tokens` |
| **状态** | 隐式（HTTP 状态） | 显式 `status` 字段 |

## 转换函数

以下转换函数可在 `service/convert.go` 中使用：

### 请求转换

#### 1. `ChatCompletionsToResponsesRequest()`
将 Chat Completions 请求转换为 Responses API 格式。

```go
func ChatCompletionsToResponsesRequest(chatRequest *dto.GeneralOpenAIRequest) (*dto.OpenAIResponsesRequest, error)
```

**转换细节：**
- `messages` → `input`（仅非系统消息）
- 系统消息 → `instructions`
- `max_tokens`/`max_completion_tokens` → `max_output_tokens`
- `reasoning_effort` → `reasoning.effort`
- 工具、温度、top_p 直接复制

#### 2. `ResponsesToChatCompletionsRequest()`
将 Responses API 请求转换为 Chat Completions 格式。

```go
func ResponsesToChatCompletionsRequest(responsesReq *dto.OpenAIResponsesRequest) (*dto.GeneralOpenAIRequest, error)
```

**转换细节：**
- `instructions` → `messages` 数组中的系统消息
- `input` → 消息数组（解包）
- `max_output_tokens` → `max_completion_tokens`
- `reasoning.effort` → `reasoning_effort`

### 响应转换

#### 3. `ChatCompletionsToResponsesResponse()`
将 Chat Completions 响应转换为 Responses API 格式。

```go
func ChatCompletionsToResponsesResponse(chatResp *dto.OpenAITextResponse) (*dto.OpenAIResponsesResponse, error)
```

**转换细节：**
- `choices` → `output` 数组
- `prompt_tokens`/`completion_tokens` → `input_tokens`/`output_tokens`
- 添加 `status: "completed"` 字段

#### 4. `ResponsesToChatCompletionsResponse()`
将 Responses API 响应转换为 Chat Completions 格式。

```go
func ResponsesToChatCompletionsResponse(responsesResp *dto.OpenAIResponsesResponse) (*dto.OpenAITextResponse, error)
```

**转换细节：**
- `output` → `choices` 数组
- `input_tokens`/`output_tokens` → `prompt_tokens`/`completion_tokens`
- 设置 `finish_reason: "stop"`

## 使用示例

### 示例 1：客户端使用 Responses API，提供商使用 Chat Completions

**场景：**新应用程序使用 OpenAI 的 Responses API，但您的后端渠道仅支持 Chat Completions 格式。

**客户端请求**（Responses API 格式）：
```json
{
  "model": "gpt-4",
  "instructions": "你是一个有帮助的助手。",
  "input": [
    {"role": "user", "content": "你好，你好吗？"}
  ],
  "max_output_tokens": 100,
  "temperature": 0.7
}
```

**转换后**（发送到后端的 Chat Completions 格式）：
```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "你是一个有帮助的助手。"},
    {"role": "user", "content": "你好，你好吗？"}
  ],
  "max_completion_tokens": 100,
  "temperature": 0.7
}
```

**后端响应**（Chat Completions 格式）：
```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "model": "gpt-4",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "我很好，谢谢你！"
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

**转换后**（返回给客户端的 Responses API 格式）：
```json
{
  "id": "chatcmpl-123",
  "object": "response",
  "model": "gpt-4",
  "status": "completed",
  "output": [{
    "index": 0,
    "type": "message",
    "content": "我很好，谢谢你！"
  }],
  "usage": {
    "input_tokens": 20,
    "output_tokens": 10,
    "total_tokens": 30
  }
}
```

## 实现状态

### ✅ 已完成
- [x] 请求转换函数（双向）
- [x] 响应转换函数（双向）
- [x] 文档

### 🚧 进行中 / 未来工作
- [ ] 与中继处理程序集成（自动转换）
- [ ] 渠道配置标志，用于格式支持
- [ ] 流式响应转换
- [ ] 内置工具处理（Responses → Chat Completions）
- [ ] 单元测试
- [ ] 集成测试

## 限制

### 当前限制
1. **流式传输**：尚未实现流式响应转换
2. **内置工具**：Responses API 内置工具（网页搜索、文件搜索）无法转换为 Chat Completions 格式
3. **高级功能**：某些 Responses API 高级功能在转换中可能会丢失

### 未来增强
1. 在中继处理程序中添加自动检测和转换
2. 支持流式转换
3. 添加渠道设置以声明原生 API 格式
4. 为不支持的功能提供回退行为

## 测试

要测试转换函数：

```go
import (
    "github.com/QuantumNous/new-api/dto"
    "github.com/QuantumNous/new-api/service"
)

// 测试 Chat Completions → Responses 转换
chatReq := &dto.GeneralOpenAIRequest{
    Model: "gpt-4",
    Messages: []dto.Message{
        {Role: "system", Content: "你很有帮助"},
        {Role: "user", Content: "你好"},
    },
    MaxCompletionTokens: 100,
}

responsesReq, err := service.ChatCompletionsToResponsesRequest(chatReq)
// responsesReq 现在包含转换后的请求

// 测试 Responses → Chat Completions 转换
responsesReq2 := &dto.OpenAIResponsesRequest{
    Model: "gpt-4",
    Instructions: json.RawMessage(`"你很有帮助"`),
    Input: json.RawMessage(`[{"role":"user","content":"你好"}]`),
    MaxOutputTokens: 100,
}

chatReq2, err := service.ResponsesToChatCompletionsRequest(responsesReq2)
// chatReq2 现在包含转换后的请求
```

## 贡献

如果您想为改进 API 转换功能做出贡献：

1. 添加对更多字段映射的支持
2. 实现流式转换
3. 添加全面的测试
4. 改进边缘情况的错误处理
5. 记录其他用例

## 相关文档

- [OpenAI Chat Completions API 文档](https://platform.openai.com/docs/api-reference/chat)
- [OpenAI Responses API 文档](https://platform.openai.com/docs/api-reference/responses)
- [渠道配置指南](./channel/other_setting.md)

## 使用说明

### 当前实现

目前，转换函数已经实现并可供使用。要在您的代码中使用这些函数：

```go
// 导入必要的包
import (
    "github.com/QuantumNous/new-api/dto"
    "github.com/QuantumNous/new-api/service"
)

// 使用转换函数
responsesReq, err := service.ChatCompletionsToResponsesRequest(chatRequest)
if err != nil {
    // 处理错误
}
```

### 未来集成

未来版本将自动在中继层进行转换，无需手动调用这些函数。渠道管理员将能够：

1. 在渠道设置中指定原生 API 格式
2. 系统自动检测格式不匹配
3. 透明地进行转换，无需客户端或提供商感知

## 总结

本功能为 New API Gateway 提供了在 OpenAI Chat Completions API 和 Responses API 之间转换的能力，使得：

- **旧服务提供商**可以通过新的 Responses API 对外暴露
- **新客户端**可以访问只支持旧 Chat Completions API 的提供商
- **灵活性**：支持双向转换，适应不同的使用场景

这是一个重要的兼容性层，帮助用户在两种 API 格式之间无缝切换。
