# QWEN OAuth API Flow (`authType: "qwen-oauth"`)

This document describes only the `qwen-oauth` path in this codebase.

## 1) How auth works

Main entry point:

- `packages/core/src/qwen/qwenOAuth2.ts` -> `getQwenOAuthClient(...)`

High-level behavior:

1. Try to load/reuse valid cached credentials via `SharedTokenManager.getValidCredentials(...)`.
2. If valid token exists, use it immediately.
3. If not, run OAuth device flow (`authWithQwenDeviceFlow(...)`).
4. Cache credentials to `~/.qwen/oauth_creds.json`.

Token cache + refresh coordination:

- `packages/core/src/qwen/sharedTokenManager.ts`
- Handles cross-process locking and refresh to avoid token races.

OAuth endpoints:

- Device code: `https://chat.qwen.ai/api/v1/oauth2/device/code`
- Token (poll + refresh): `https://chat.qwen.ai/api/v1/oauth2/token`

## 2) How device code is generated

In `authWithQwenDeviceFlow(...)`:

1. Local PKCE values are generated:
   - `generateCodeVerifier()` -> random verifier
   - `generateCodeChallenge(verifier)` -> SHA-256 challenge
2. Client requests device authorization with:
   - `client_id`
   - `scope` (`openid profile email model.completion`)
   - `code_challenge`
   - `code_challenge_method: S256`
3. Server returns device auth payload (`DeviceAuthorizationData`) containing:
   - `device_code`
   - `user_code`
   - `verification_uri`
   - `verification_uri_complete`
   - `expires_in`

Important: `device_code` is issued by the OAuth server; the client generates PKCE verifier/challenge used in the same flow.

## 3) How authorization is confirmed

After device auth response:

1. Emits `QwenOAuth2Event.AuthUri` for UI integration.
2. Opens browser to `verification_uri_complete` (unless suppressed), or prints fallback URL box.
3. Polls token endpoint with:
   - `grant_type = urn:ietf:params:oauth:grant-type:device_code`
   - `device_code`
   - `code_verifier`
4. Poll responses handled as:
   - `authorization_pending` -> keep polling
   - `slow_down` -> increase poll interval
   - success -> store tokens, emit success
   - 401/429/timeout/cancel/error -> emit appropriate failure status

## 4) How a message is sent to the API

Routing:

- `packages/core/src/core/contentGenerator.ts`
  - `AuthType.QWEN_OAUTH` creates `QwenContentGenerator`.

Message send path:

1. `QwenContentGenerator.generateContent(...)` or `.generateContentStream(...)`
2. `executeWithCredentialManagement(...)` gets valid token + endpoint (from shared credentials).
3. It sets dynamic client auth:
   - `pipeline.client.apiKey = access_token`
   - `pipeline.client.baseURL = normalized(resource_url or default DashScope URL)`
4. Delegates to OpenAI-compatible pipeline:
   - `packages/core/src/core/openaiContentGenerator/pipeline.ts`
   - calls `client.chat.completions.create(...)` (non-stream or stream)

So inference calls are OpenAI-compatible chat-completions over DashScope-style endpoint, authenticated with OAuth access token.

## 5) How message is received

Response path:

1. OpenAI-compatible response/chunks come back from `chat.completions.create(...)`.
2. Converted by `OpenAIContentConverter`:
   - `packages/core/src/core/openaiContentGenerator/converter.ts`
   - `convertOpenAIResponseToGemini(...)`
   - `convertOpenAIChunkToGemini(...)`
3. Returned to upper layers as `GenerateContentResponse` (Gemini-style internal format).

## 6) What is received

### OAuth token response (auth stage)

On success, token payload includes fields such as:

- `access_token`
- `refresh_token` (if provided)
- `token_type`
- `expires_in`
- `resource_url` (used to build API base URL dynamically)

### Model generation response (inference stage)

Converted `GenerateContentResponse` includes:

- `candidates[].content.parts[]` (text, thought, functionCall/tool-call parts)
- `finishReason`
- `usageMetadata` (token counts, cached tokens, thinking tokens when available)
- `responseId`, `modelVersion`, `createTime`

## 7) What models it can use

Hard-coded Qwen OAuth models are in:

- `packages/core/src/models/constants.ts` (`QWEN_OAUTH_MODELS`)

Allowed model IDs:

- `coder-model` (default)
- `vision-model`

These are always registered for `qwen-oauth` and are not overridden by user modelProviders config:

- `packages/core/src/models/modelRegistry.ts`

## 8) What features it has (vision, web search, tools)

### Vision

- `vision-model` has `capabilities: { vision: true }`.
- DashScope provider has vision handling and enables `vl_high_resolution_images: true` for vision models:
  - `packages/core/src/core/openaiContentGenerator/provider/dashscope.ts`

### Web search

- Tool name exists as `web_search`:
  - `packages/core/src/tools/tool-names.ts`
- Tool is conditionally registered when web search config exists:
  - `packages/core/src/config/config.ts` (`getWebSearchConfig()` check)
- DashScope web-search provider is available for `qwen-oauth` and uses OAuth credentials (`resource_url` + bearer token):
  - `packages/core/src/tools/web-search/providers/dashscope-provider.ts`

### Tool/function calling

- OpenAI-compatible tool calls are supported in request/response conversion:
  - `packages/core/src/core/openaiContentGenerator/converter.ts`
