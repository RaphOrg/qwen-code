# Qwen OAuth API Reference

This document covers the API used by Qwen Code when running with the **qwen-oauth** auth type — the default, free authentication method that requires only a `qwen.ai` account.

---

## Table of Contents

- [How Authentication Works](#how-authentication-works)
  - [Overview](#overview)
  - [Step 1: Generate a Device Code](#step-1-generate-a-device-code)
  - [Step 2: Confirm the Device Code](#step-2-confirm-the-device-code)
  - [Step 3: Token Refresh](#step-3-token-refresh)
- [How Messages Are Sent](#how-messages-are-sent)
  - [Endpoint](#endpoint)
  - [Request Headers](#request-headers)
  - [Request Body](#request-body)
- [How Messages Are Received](#how-messages-are-received)
  - [Non-Streaming Response](#non-streaming-response)
  - [Streaming Response](#streaming-response)
  - [Streaming Errors](#streaming-errors)
- [Available Models](#available-models)
- [Features](#features)
  - [Vision (Image Understanding)](#vision-image-understanding)
  - [Web Search](#web-search)
  - [Tool / Function Calling](#tool--function-calling)
  - [Prompt Caching](#prompt-caching)

---

## How Authentication Works

### Overview

Qwen OAuth uses the **OAuth 2.0 Device Authorization Grant** ([RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628)) with **PKCE** ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)). The flow is:

1. Client generates a PKCE pair (`code_verifier` + `code_challenge`).
2. Client requests a **device code** from `chat.qwen.ai`.
3. User opens a browser link and authorizes.
4. Client polls for an **access token**.
5. Access token is used as a `Bearer` token for all DashScope API calls.
6. When the token expires, the client uses the **refresh token** to get a new one.

**OAuth constants used by Qwen Code:**

| Constant | Value |
| -------- | ----- |
| OAuth base URL | `https://chat.qwen.ai` |
| Client ID | `f0304373b74a44d2b584a3fb70ca9e56` |
| Scope | `openid profile email model.completion` |
| Device code grant type | `urn:ietf:params:oauth:grant-type:device_code` |

### Step 1: Generate a Device Code

Before requesting a device code, the client generates a **PKCE pair**:

- **`code_verifier`**: 32 random bytes, base64url-encoded.
- **`code_challenge`**: SHA-256 hash of the `code_verifier`, base64url-encoded.

The client stores the `code_verifier` locally (never sent to the server until token exchange).

Then the client sends:

```
POST https://chat.qwen.ai/api/v1/oauth2/device/code
Content-Type: application/x-www-form-urlencoded
Accept: application/json
x-request-id: <random UUID>
```

**Request body** (form-encoded):

```
client_id=f0304373b74a44d2b584a3fb70ca9e56
&scope=openid+profile+email+model.completion
&code_challenge=<BASE64URL_SHA256_HASH>
&code_challenge_method=S256
```

**Response** (success):

```json
{
  "device_code": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "user_code": "ABCD-1234",
  "verification_uri": "https://chat.qwen.ai/device",
  "verification_uri_complete": "https://chat.qwen.ai/device?user_code=ABCD-1234",
  "expires_in": 600
}
```

| Field | Description |
| ----- | ----------- |
| `device_code` | Opaque code used to poll for the token |
| `user_code` | Short code the user enters or sees on the authorization page |
| `verification_uri` | URL the user opens in their browser |
| `verification_uri_complete` | Full URL with the `user_code` pre-filled |
| `expires_in` | Seconds before the device code expires (typically 600) |

Qwen Code automatically opens `verification_uri_complete` in the user's browser (or displays it in the terminal for headless environments).

### Step 2: Confirm the Device Code

After the user authorizes in their browser, the client polls for a token. Polling starts at a 2-second interval and the client makes up to `expires_in / interval` attempts.

```
POST https://chat.qwen.ai/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded
Accept: application/json
```

**Request body** (form-encoded):

```
grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adevice_code
&client_id=f0304373b74a44d2b584a3fb70ca9e56
&device_code=<DEVICE_CODE>
&code_verifier=<PKCE_CODE_VERIFIER>
```

**While waiting for the user:**

- HTTP 400 with `"error": "authorization_pending"` — the user hasn't authorized yet, keep polling.
- HTTP 429 with `"error": "slow_down"` — polling too fast, the client increases the interval.

**Response** (success — user authorized):

```json
{
  "access_token": "eyJ...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "resource_url": "https://dashscope.aliyuncs.com"
}
```

| Field | Description |
| ----- | ----------- |
| `access_token` | JWT used as `Bearer` token for all API calls |
| `refresh_token` | Used to obtain a new access token when the current one expires |
| `token_type` | Always `"Bearer"` |
| `expires_in` | Access token lifetime in seconds |
| `resource_url` | Base URL for the DashScope API and web search service |

The `access_token` and `refresh_token` are cached locally so the user doesn't need to log in again.

### Step 3: Token Refresh

When the `access_token` expires, the client automatically refreshes it:

```
POST https://chat.qwen.ai/api/v1/oauth2/token
Content-Type: application/x-www-form-urlencoded
Accept: application/json
```

**Request body** (form-encoded):

```
grant_type=refresh_token
&refresh_token=<REFRESH_TOKEN>
&client_id=f0304373b74a44d2b584a3fb70ca9e56
```

**Response** (success):

```json
{
  "access_token": "eyJ...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "resource_url": "https://dashscope.aliyuncs.com"
}
```

If the refresh token is also expired or invalid (HTTP 400), the credentials are cleared and the user must re-authenticate through the device flow again.

---

## How Messages Are Sent

### Endpoint

Messages are sent using the **OpenAI-compatible Chat Completions API** on DashScope:

```
POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions
```

(or `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` for international users.)

The base URL can also come from the `resource_url` returned during OAuth token exchange.

### Request Headers

Every request includes these headers:

```http
Authorization: Bearer <access_token>
User-Agent: QwenCode/<version> (<platform>; <arch>)
X-DashScope-CacheControl: enable
X-DashScope-UserAgent: QwenCode/<version>
X-DashScope-AuthType: qwen-oauth
```

| Header | Purpose |
| ------ | ------- |
| `Authorization` | The OAuth access token obtained during login |
| `X-DashScope-CacheControl` | Enables DashScope prompt caching for repeated content |
| `X-DashScope-UserAgent` | Identifies the client to DashScope |
| `X-DashScope-AuthType` | Tells DashScope this is an OAuth-authenticated request (vs. API key) |

### Request Body

The request body follows the standard **OpenAI Chat Completions** format:

```json
{
  "model": "coder-model",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "Explain how quicksort works"
    }
  ],
  "stream": true,
  "stream_options": { "include_usage": true },
  "temperature": 0.7,
  "top_p": 0.9,
  "max_tokens": 4096
}
```

**Message roles:**

| Role | Description |
| ---- | ----------- |
| `system` | System prompt that sets the assistant's behavior |
| `user` | User's message |
| `assistant` | Previous assistant response (for conversation history) |
| `tool` | Result of a tool/function call (includes `tool_call_id`) |

**Supported sampling parameters:**

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `temperature` | number | Controls randomness (0.0 = deterministic, higher = more random) |
| `top_p` | number | Nucleus sampling threshold |
| `top_k` | number | Top-K sampling |
| `max_tokens` | number | Maximum number of tokens to generate |
| `presence_penalty` | number | Penalizes tokens already present in the conversation |
| `frequency_penalty` | number | Penalizes tokens based on frequency |
| `repetition_penalty` | number | DashScope-specific repetition penalty |

**Streaming:** When `"stream": true` is set, `"stream_options": { "include_usage": true }` is also added so token usage is included in the final streaming chunk.

---

## How Messages Are Received

### Non-Streaming Response

When `stream` is `false` (or omitted), a single JSON response is returned:

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "coder-model",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Quicksort works by selecting a pivot element..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 50,
    "completion_tokens": 200,
    "total_tokens": 250,
    "prompt_tokens_details": {
      "cached_tokens": 30
    }
  }
}
```

| Field | Description |
| ----- | ----------- |
| `choices[].message.content` | The assistant's text response |
| `choices[].message.tool_calls` | Present when the model wants to call a tool (see [Tool / Function Calling](#tool--function-calling)) |
| `choices[].finish_reason` | Why generation stopped: `"stop"` (natural end), `"tool_calls"` (wants to call tools), `"length"` (hit max_tokens) |
| `usage.prompt_tokens` | Number of tokens in the prompt |
| `usage.completion_tokens` | Number of tokens generated |
| `usage.prompt_tokens_details.cached_tokens` | Number of prompt tokens served from DashScope's cache (saves cost/latency) |

### Streaming Response

When `"stream": true`, the response is delivered as **Server-Sent Events (SSE)**. Each event is a line prefixed with `data: `:

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"coder-model","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"coder-model","choices":[{"index":0,"delta":{"content":"Quick"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"coder-model","choices":[{"index":0,"delta":{"content":"sort"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"coder-model","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":50,"completion_tokens":200,"total_tokens":250}}

data: [DONE]
```

The client concatenates all `delta.content` strings to build the full response. Token usage appears on the final chunk (because `include_usage: true` was set).

### Streaming Errors

DashScope may return errors **inside the stream** rather than as HTTP error codes. When a chunk has `"finish_reason": "error_finish"`, the content in `delta.content` is the error message:

```json
{
  "choices": [{
    "index": 0,
    "delta": { "content": "Rate limit exceeded. Please try again later." },
    "finish_reason": "error_finish"
  }]
}
```

Qwen Code detects this and raises a `StreamContentError`.

---

## Available Models

The qwen-oauth auth type provides two built-in models:

| Model ID | Description | Vision Support |
| -------- | ----------- | -------------- |
| `coder-model` | Qwen 3.5 Plus — efficient hybrid model with leading coding performance | No |
| `vision-model` | The latest Qwen Vision model from Alibaba Cloud ModelStudio | Yes |

`coder-model` is the default. Switch between models using the `/model` command inside Qwen Code.

**Rate limits** (Qwen OAuth free tier): **60 requests/minute** and **1,000 requests/day**.

---

## Features

### Vision (Image Understanding)

The `vision-model` can understand images. Images are sent as part of the `messages` array using the OpenAI multi-modal content format:

```json
{
  "model": "vision-model",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "What is in this image?" },
        {
          "type": "image_url",
          "image_url": {
            "url": "data:image/png;base64,iVBORw0KGgo..."
          }
        }
      ]
    }
  ]
}
```

**Supported image formats:**

- **Base64 inline**: `data:<mime>;base64,<data>` (e.g., `data:image/png;base64,...`)
- **Remote URL**: `https://example.com/image.png`

When using `vision-model`, the DashScope provider also sends `"vl_high_resolution_images": true` in the request body for better OCR and detail extraction.

### Web Search

The web search feature uses a **separate DashScope endpoint** (not the Chat Completions API). The `resource_url` from the OAuth token response provides the base URL.

```
POST {resource_url}/api/v1/indices/plugin/web_search
Authorization: Bearer <access_token>
Content-Type: application/json
```

**Request:**

```json
{
  "uq": "latest TypeScript features",
  "page": 1,
  "rows": 10
}
```

| Field | Description |
| ----- | ----------- |
| `uq` | The search query |
| `page` | Page number (starts at 1) |
| `rows` | Number of results to return |

**Response:**

```json
{
  "data": {
    "docs": [
      {
        "title": "TypeScript 5.0 Release Notes",
        "url": "https://devblogs.microsoft.com/typescript/...",
        "snippet": "TypeScript 5.0 introduces decorators, const type parameters...",
        "timestamp_format": "2025-03-01",
        "_score": 0.95
      }
    ]
  }
}
```

Web search results are fed back into the conversation as context for the model.

### Tool / Function Calling

Qwen models support OpenAI-compatible function calling. Tools are declared in the request:

```json
{
  "model": "coder-model",
  "messages": [{ "role": "user", "content": "What's the weather in Beijing?" }],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": { "type": "string", "description": "City name" }
          },
          "required": ["location"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

When the model decides to call a tool, the response includes `tool_calls` instead of `content`:

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [
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
    "finish_reason": "tool_calls"
  }]
}
```

The client executes the tool and sends the result back:

```json
{
  "messages": [
    { "role": "user", "content": "What's the weather in Beijing?" },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [{ "id": "call_abc123", "type": "function", "function": { "name": "get_weather", "arguments": "{\"location\":\"Beijing\"}" } }]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "{\"temperature\": 22, \"condition\": \"sunny\"}"
    }
  ]
}
```

During streaming, tool call arguments arrive incrementally across multiple chunks. Qwen Code accumulates the fragments and reconstructs the full call when `finish_reason` is received.

**`tool_choice` options:**

- `"auto"` — model decides whether to call a tool
- `"required"` — model must call at least one tool
- `{"type": "function", "function": {"name": "..."}}` — force a specific tool

### Prompt Caching

DashScope supports **prompt caching** to speed up repeated requests with shared context (like system prompts). This is enabled by default for qwen-oauth.

**How it works:**

- The `X-DashScope-CacheControl: enable` header activates caching.
- The DashScope provider automatically marks messages with `cache_control: { type: "ephemeral" }` to indicate which parts should be cached:
  - **System message** — always cached.
  - **Last tool definition** — cached during streaming.
  - **Most recent history message** — cached during streaming.
- Cached tokens are reported in the response under `usage.prompt_tokens_details.cached_tokens`.

Caching is automatic and transparent — no user configuration needed.
