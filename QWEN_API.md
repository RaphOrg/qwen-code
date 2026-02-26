# Qwen Code API Reference

Qwen Code uses an **OpenAI-compatible Chat Completions API** as its core interface. All model providers — including DashScope (Alibaba Cloud), OpenAI, DeepSeek, OpenRouter, and ModelScope — are accessed through the standard `/v1/chat/completions` endpoint using the [OpenAI Node.js SDK](https://www.npmjs.com/package/openai).

In short: **yes, it is OpenAI-compatible**. If you can host an OpenAI-compatible server, Qwen Code can talk to it. The sections below document the exact endpoints, request/response formats, provider-specific headers, and the Qwen OAuth flow.

---

## Table of Contents

- [Chat Completions](#chat-completions)
  - [Endpoint](#endpoint)
  - [Request Format](#request-format)
  - [Response Format](#response-format)
  - [Streaming](#streaming)
  - [Tool / Function Calling](#tool--function-calling)
- [Supported Providers & Base URLs](#supported-providers--base-urls)
- [Authentication](#authentication)
  - [API Key (Bearer Token)](#api-key-bearer-token)
  - [Qwen OAuth 2.0 Device Flow](#qwen-oauth-20-device-flow)
- [Provider-Specific Headers](#provider-specific-headers)
- [DashScope Web Search API](#dashscope-web-search-api)
- [Configuration Reference](#configuration-reference)
- [Differences from Vanilla OpenAI](#differences-from-vanilla-openai)

---

## Chat Completions

### Endpoint

```
POST {baseUrl}/chat/completions
```

All providers use the OpenAI Chat Completions endpoint. The `baseUrl` already includes `/v1` (or equivalent), so the full path is:

| Provider     | Full URL                                                              |
| ------------ | --------------------------------------------------------------------- |
| DashScope CN | `https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions`  |
| DashScope Intl | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` |
| OpenAI       | `https://api.openai.com/v1/chat/completions`                         |
| DeepSeek     | `https://api.deepseek.com/v1/chat/completions`                       |
| OpenRouter   | `https://openrouter.ai/api/v1/chat/completions`                      |
| ModelScope   | `https://api.modelscope.cn/v1/chat/completions`                      |

### Request Format

Standard OpenAI Chat Completions request body:

```jsonc
{
  "model": "qwen3-coder-plus",          // Model identifier
  "messages": [                          // Conversation history
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "Hello"                 // Can also be an array of content parts for vision
    },
    {
      "role": "assistant",
      "content": "Hi there!"
    },
    {
      "role": "tool",                    // Tool result message
      "tool_call_id": "call_abc123",
      "content": "{ \"result\": 42 }"
    }
  ],

  // Optional sampling parameters
  "temperature": 0.7,
  "top_p": 0.9,
  "top_k": 40,                          // Supported by DashScope/Qwen models
  "max_tokens": 4096,
  "presence_penalty": 0.0,
  "frequency_penalty": 0.0,
  "repetition_penalty": 1.0,            // DashScope extension

  // Streaming
  "stream": true,
  "stream_options": { "include_usage": true },

  // Tool / function calling
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": { "type": "string" }
          },
          "required": ["location"]
        }
      }
    }
  ],
  "tool_choice": "auto"                 // "auto" | "required" | {"type":"function","function":{"name":"..."}}
}
```

### Response Format

Standard OpenAI Chat Completions response:

```jsonc
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "qwen3-coder-plus",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?",
        "tool_calls": [                  // Present when the model invokes tools
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\":\"Beijing\"}"
            }
          }
        ]
      },
      "finish_reason": "stop"            // "stop" | "tool_calls" | "length"
    }
  ],
  "usage": {
    "prompt_tokens": 50,
    "completion_tokens": 20,
    "total_tokens": 70,
    "prompt_tokens_details": {
      "cached_tokens": 30               // DashScope prompt caching
    }
  }
}
```

> **Note:** DashScope may also return `cached_tokens` at the top level of `usage` (alongside `prompt_tokens_details`). Qwen Code normalizes both formats.

### Streaming

When `"stream": true`, the response is delivered as Server-Sent Events (SSE). Each chunk follows the OpenAI streaming format:

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"qwen3-coder-plus","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"qwen3-coder-plus","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":50,"completion_tokens":20,"total_tokens":70}}

data: [DONE]
```

**DashScope-specific streaming behavior:**
- Some providers may return errors embedded in the stream as a chunk with `"finish_reason": "error_finish"` and the error message in `delta.content`, rather than an HTTP error status code.

### Tool / Function Calling

Tools are defined in the standard OpenAI format. During streaming, tool call arguments arrive incrementally:

```jsonc
// Streaming chunk with tool call fragment
{
  "choices": [{
    "delta": {
      "tool_calls": [{
        "index": 0,
        "id": "call_abc123",           // Present in first chunk for this tool call
        "function": {
          "name": "get_weather",       // Present in first chunk
          "arguments": "{\"loc"        // Partial JSON, accumulated across chunks
        }
      }]
    }
  }]
}
```

Qwen Code accumulates these fragments and reconstructs the complete tool call when `finish_reason` is received.

---

## Supported Providers & Base URLs

Provider detection is automatic based on the configured `baseUrl`:

| Provider       | Base URL                                                         | Detection Rule                                |
| -------------- | ---------------------------------------------------------------- | --------------------------------------------- |
| **DashScope**  | `https://dashscope.aliyuncs.com/compatible-mode/v1` (default)    | URL contains `dashscope.aliyuncs.com`, or `authType` is `qwen-oauth`, or no `baseUrl` set |
| **DashScope Intl** | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`    | URL contains `dashscope-intl.aliyuncs.com`    |
| **Coding Plan CN** | `https://coding.dashscope.aliyuncs.com/v1`                  | URL contains `dashscope.aliyuncs.com`         |
| **Coding Plan Intl** | `https://coding-intl.dashscope.aliyuncs.com/v1`           | URL contains `dashscope.aliyuncs.com`         |
| **OpenAI**     | `https://api.openai.com/v1`                                      | Fallback (default provider)                   |
| **DeepSeek**   | `https://api.deepseek.com/v1`                                    | URL contains `api.deepseek.com`               |
| **OpenRouter** | `https://openrouter.ai/api/v1`                                   | URL contains `openrouter.ai`                  |
| **ModelScope** | `https://api.modelscope.cn/v1`                                   | URL contains `modelscope.cn`                  |

Any URL that doesn't match a specific provider falls through to the **Default** (generic OpenAI-compatible) provider.

---

## Authentication

### API Key (Bearer Token)

All OpenAI-compatible providers use standard Bearer token authentication, injected automatically by the OpenAI SDK:

```
Authorization: Bearer <apiKey>
```

The API key is resolved from (highest to lowest priority):
1. CLI flags (`--openai-api-key`)
2. System environment variables (`OPENAI_API_KEY`, `DASHSCOPE_API_KEY`, etc.)
3. `.env` file
4. `settings.json` → `env` field

### Qwen OAuth 2.0 Device Flow

For the default Qwen experience (no API key needed), Qwen Code implements [RFC 8628 (OAuth 2.0 Device Authorization Grant)](https://datatracker.ietf.org/doc/html/rfc8628):

**Step 1 — Request device code:**

```
POST https://chat.qwen.ai/api/v1/oauth2/device/code
Content-Type: application/x-www-form-urlencoded
Accept: application/json
```

```
client_id=f0304373b74a44d2b584a3fb70ca9e56
&scope=openid+profile+email+model.completion
&code_challenge=<S256_CHALLENGE>
&code_challenge_method=S256
```

**Response:**
```jsonc
{
  "device_code": "...",
  "user_code": "ABCD-1234",
  "verification_uri": "https://chat.qwen.ai/device",
  "verification_uri_complete": "https://chat.qwen.ai/device?user_code=ABCD-1234",
  "expires_in": 600,
  "interval": 5
}
```

**Step 2 — User authorizes in browser**, then the client polls for a token:

```
POST https://chat.qwen.ai/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded
Accept: application/json
```

```
grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adevice_code
&client_id=f0304373b74a44d2b584a3fb70ca9e56
&device_code=<DEVICE_CODE>
&code_verifier=<PKCE_VERIFIER>
```

**Response (success):**
```jsonc
{
  "access_token": "eyJ...",
  "refresh_token": "...",
  "id_token": "eyJ...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

**Polling errors:** `authorization_pending` (HTTP 400) and `slow_down` (HTTP 429) per RFC 8628.

**Step 3 — Token refresh:**

```
POST https://chat.qwen.ai/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded
```

```
grant_type=refresh_token
&refresh_token=<REFRESH_TOKEN>
&client_id=f0304373b74a44d2b584a3fb70ca9e56
```

The obtained `access_token` is then used as the Bearer token for DashScope API calls.

---

## Provider-Specific Headers

All providers include a common `User-Agent` header:

```
User-Agent: QwenCode/<version> (<platform>; <arch>)
```

### DashScope (Qwen) Headers

```http
X-DashScope-CacheControl: enable              # Enables prompt caching
X-DashScope-UserAgent: QwenCode/<version>
X-DashScope-AuthType: qwen-oauth               # or "openai" for API key auth
```

### OpenRouter Headers

```http
HTTP-Referer: https://github.com/QwenLM/qwen-code.git
X-OpenRouter-Title: Qwen Code
```

### DeepSeek

No special headers. Note: DeepSeek only supports text content — image/vision parts are stripped from messages.

### ModelScope

No special headers. Note: `stream_options` is removed from non-streaming requests.

---

## DashScope Web Search API

Qwen Code also calls a DashScope web search endpoint for the web search tool:

```
POST {resource_url}/api/v1/indices/plugin/web_search
Authorization: Bearer <accessToken>
Content-Type: application/json
```

```json
{
  "uq": "search query text",
  "page": 1,
  "rows": 10
}
```

**Response:**
```jsonc
{
  "data": {
    "docs": [
      {
        "title": "Page Title",
        "url": "https://example.com",
        "snippet": "Relevant text excerpt...",
        "timestamp_format": "2025-01-15",
        "_score": 0.95
      }
    ]
  }
}
```

---

## Configuration Reference

The content generator is configured via `settings.json` or CLI arguments. Key fields:

```jsonc
{
  "model": "qwen3-coder-plus",        // Model ID sent to the API
  "apiKey": "sk-...",                  // API key (or use envKey for env var reference)
  "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
  "authType": "openai",               // "openai" | "qwen-oauth" | "anthropic" | "gemini" | "vertex-ai"
  "timeout": 120000,                   // Request timeout in ms (default: 120000)
  "maxRetries": 3,                     // Retry attempts (default: 3)
  "enableCacheControl": true,          // Enable DashScope prompt caching
  "contextWindowSize": 128000,         // Model context window size
  "customHeaders": {},                 // Additional HTTP headers
  "extra_body": {},                    // Extra fields merged into the request body
  "schemaCompliance": "auto",          // "auto" | "openapi_30"
  "samplingParams": {
    "temperature": 0.7,
    "top_p": 0.9,
    "top_k": 40,
    "max_tokens": 4096,
    "presence_penalty": 0.0,
    "frequency_penalty": 0.0,
    "repetition_penalty": 1.0
  }
}
```

### Default Models

| Auth Type    | Default Model         |
| ------------ | --------------------- |
| `openai`     | `qwen3-coder-plus`    |
| `qwen-oauth` | `coder-model`        |

### Qwen OAuth Built-in Models

| Model ID       | Description                                                        | Vision |
| -------------- | ------------------------------------------------------------------ | ------ |
| `coder-model`  | Qwen 3.5 Plus — efficient hybrid model with leading coding perf.  | No     |
| `vision-model` | The latest Qwen Vision model from Alibaba Cloud ModelStudio        | Yes    |

---

## Differences from Vanilla OpenAI

The API is **fully OpenAI-compatible** with these provider-specific extensions:

| Feature | Standard OpenAI | Qwen Code Extensions |
| ------- | --------------- | -------------------- |
| Endpoint format | `/v1/chat/completions` | ✅ Same |
| Request body | Standard fields | Adds `top_k`, `repetition_penalty`, `extra_body` |
| Response body | Standard fields | ✅ Same (plus `cached_tokens` at usage level for DashScope) |
| Streaming | SSE with `data:` lines | ✅ Same (DashScope may use `error_finish` instead of HTTP errors) |
| Tool calling | OpenAI function calling format | ✅ Same |
| Auth | `Authorization: Bearer` | ✅ Same (plus Qwen OAuth device flow as an alternative) |
| Headers | Standard | DashScope adds `X-DashScope-*`; OpenRouter adds `HTTP-Referer`, `X-OpenRouter-Title` |
| Vision/multipart | Content parts array | DeepSeek strips non-text parts; others support it |

**Bottom line:** Any OpenAI-compatible server or client library will work with Qwen Code. The DashScope-specific headers and OAuth flow are only used when connecting to Alibaba Cloud's DashScope service.
