# Qwen Code API Surface

This repository supports multiple model backends.  
The **default API protocol is OpenAI-compatible** when `authType` is `openai`.

## How provider selection works

- `AuthType` is defined in `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/contentGenerator.ts`:
  - `openai`
  - `qwen-oauth`
  - `gemini`
  - `vertex-ai`
  - `anthropic`
- Runtime routing happens in `createContentGenerator(...)` in the same file:
  - `openai` -> OpenAI-compatible generator
  - `qwen-oauth` -> Qwen OAuth flow + Qwen content generator
  - `gemini` / `vertex-ai` -> Gemini generator
  - `anthropic` -> Anthropic generator

## OpenAI-compatible path (yes, this is the main/default style)

OpenAI-compatible routing lives in:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/openaiContentGenerator/index.ts`
- `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/openaiContentGenerator/provider/default.ts`

It uses the OpenAI SDK client with:

- `apiKey`
- `baseURL` (from config)
- standard chat-completions style request building

Default/recognized OpenAI-compatible endpoints are in:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/openaiContentGenerator/constants.ts`
  - `https://api.openai.com/v1`
  - `https://dashscope.aliyuncs.com/compatible-mode/v1`
  - `https://api.deepseek.com/v1`
  - `https://openrouter.ai/api/v1`

Provider specializations are still OpenAI-compatible (same core protocol) with small header/request tweaks:

- DashScope: `/provider/dashscope.ts`
- DeepSeek: `/provider/deepseek.ts`
- OpenRouter: `/provider/openrouter.ts`
- ModelScope: `/provider/modelscope.ts`

## What is **not** purely OpenAI-compatible

### 1) `qwen-oauth` authentication flow

Qwen OAuth token acquisition is **not OpenAI API format**. It uses OAuth device-code endpoints:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/qwen/qwenOAuth2.ts`
  - `https://chat.qwen.ai/api/v1/oauth2/device/code`
  - `https://chat.qwen.ai/api/v1/oauth2/token`

After obtaining/refreshing tokens, generation requests are sent through an OpenAI-compatible DashScope provider in:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/qwen/qwenContentGenerator.ts`

So `qwen-oauth` = **OAuth login flow + OpenAI-compatible inference calls**.

### 2) Gemini / Vertex AI

Not OpenAI-compatible protocol. Uses Google GenAI SDK:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/geminiContentGenerator/geminiContentGenerator.ts`

### 3) Anthropic

Not OpenAI-compatible protocol. Uses Anthropic SDK:

- `/home/runner/work/qwen-code/qwen-code/packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.ts`

## Bottom line

- If configured with `authType: "openai"` (or DashScope/DeepSeek/OpenRouter/ModelScope OpenAI-style base URLs), this codebase is OpenAI-compatible.
- If configured with `qwen-oauth`, only the login/token flow is custom OAuth; model calls still go through an OpenAI-compatible provider.
- `gemini`/`vertex-ai` and `anthropic` are provider-native, not OpenAI-compatible.
