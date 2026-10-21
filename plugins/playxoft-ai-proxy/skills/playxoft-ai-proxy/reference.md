# Playxoft AI proxy: reference

Full details behind `SKILL.md`. Source of truth: `guides/` in the
`AI_Proxy_Playxoft` repo. The team docs at `https://ai-proxy.playxoft.com/docs`
say the same, but sit behind a team sign-in (email and password) that AI
tools can't pass; with a docs access token they can be read through the docs
API instead (section 13).

## Contents

1. Base URLs and keys
2. Code per SDK
3. Reading the trace headers
4. Bring your own key (BYOK)
5. Unified endpoint
6. Stored prompts
7. Limits, budgets and usage
8. Regions and failover
9. Headers
10. Error codes
11. Troubleshooting
12. Moving a project onto the proxy
13. Live docs (`PX_DOCS_TOKEN`)

## 1. Base URLs and keys

| Provider | Base URL | Key header the SDK sends |
|---|---|---|
| OpenAI | `https://ai-proxy.playxoft.com/api/openai/v1` | `Authorization: Bearer` |
| Anthropic (Anthropic SDK) | `https://ai-proxy.playxoft.com/api/anthropic` (the SDK appends `/v1/messages`) | `x-api-key` |
| Anthropic (Vercel AI SDK, raw HTTP) | `https://ai-proxy.playxoft.com/api/anthropic/v1` | `x-api-key` |
| Gemini (Vercel AI SDK, raw HTTP) | `https://ai-proxy.playxoft.com/api/gemini/v1beta` | `x-goog-api-key` |
| Gemini (`@google/genai`) | `httpOptions.baseUrl: 'https://ai-proxy.playxoft.com/api/gemini'` (the SDK appends `/v1beta`) | `x-goog-api-key` |
| Unified, OpenAI format | `https://ai-proxy.playxoft.com/api/v1` | `Authorization: Bearer` |

- The key is the project's proxy key, `pxk_<project>_…`, stored as the secret
  `PX_AI_KEY`. The proxy swaps in the real provider key.
- Gemini: the key goes in the `x-goog-api-key` header, never `?key=` in the
  URL (request URLs are logged, so the key would end up in the logs).
- Paths forward unchanged: `/api/openai/v1/*` → `https://api.openai.com/v1/*`,
  `/api/anthropic/v1/*` → `https://api.anthropic.com/v1/*`,
  `/api/gemini/{v1beta,v1,v1alpha}/*` → `https://generativelanguage.googleapis.com/…`.
  Every model, streaming (SSE), tools, structured output, image input and
  image generation work.
- Where the key lives: in Xecret (xecret.playxoft.com, the team's secret
  manager), in the project's environments, e.g. project `spendchat`,
  environments `production`, `staging`, `development`. The proxy owner saves
  it there (`create-project.ts --save-to-xecret <project>/<environment>`).
  Never paste it into a chat or put it on a command line.
- Cloudflare Worker: push it from the project's repo with
  `xecret secrets get PX_AI_KEY --plain | npx wrangler secret put PX_AI_KEY`
  (each Wrangler environment separately, e.g. add `--env staging`). A Worker
  can't read Xecret at runtime, so this push is needed.
- Local `wrangler dev`: `xecret pull --format env -o .dev.vars` (`.dev.vars`
  must be gitignored). Node dev servers: `xecret run -- npm run dev`, which
  puts the key in `process.env`.
- Node servers in production: run them under `xecret run --`, with a service
  token in `XECRET_TOKEN` on servers and in CI.
- Without a `.xecret.yaml` in the repo, add `--project <project>` and
  `--environment <environment>` to the `xecret` commands.
- A new or rotated key takes a minute or two to work everywhere.
- Request bodies are limited to 25 MB (`413 px_payload_too_large`), with or
  without a `content-length`.

## 2. Code per SDK

All samples are server-side. In a Cloudflare Worker read `env.PX_AI_KEY`; in
Node read `process.env.PX_AI_KEY` when the client is first needed, not at
import time.

### OpenAI SDK

```ts
import OpenAI from 'openai';

export function openaiClient(pxAiKey: string): OpenAI {
  return new OpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/openai/v1', apiKey: pxAiKey });
}

export async function hello(pxAiKey: string): Promise<string> {
  const { data, response } = await openaiClient(pxAiKey)
    .responses.create({ model: 'gpt-5-mini', input: 'Say hi' })
    .withResponse();
  console.log(response.headers.get('x-px-request-id'), response.headers.get('x-px-region-used'));
  return data.output_text;
}
```

### Anthropic SDK

```ts
import Anthropic from '@anthropic-ai/sdk';

export async function hello(pxAiKey: string): Promise<string> {
  const anthropic = new Anthropic({ baseURL: 'https://ai-proxy.playxoft.com/api/anthropic', apiKey: pxAiKey });
  const { data: message, response } = await anthropic.messages
    .create({ model: 'claude-opus-5', max_tokens: 1024, messages: [{ role: 'user', content: 'Say hi' }] })
    .withResponse();
  console.log(response.headers.get('x-px-request-id'));
  return message.content.flatMap((block) => (block.type === 'text' ? [block.text] : [])).join('');
}
```

### Vercel AI SDK

```ts
import { createAnthropic } from '@ai-sdk/anthropic';
import { createGoogleGenerativeAI } from '@ai-sdk/google';
import { createOpenAI } from '@ai-sdk/openai';
import { generateText } from 'ai';

const PX = 'https://ai-proxy.playxoft.com';

export function providers(pxAiKey: string) {
  return {
    openai: createOpenAI({ baseURL: `${PX}/api/openai/v1`, apiKey: pxAiKey }),
    anthropic: createAnthropic({ baseURL: `${PX}/api/anthropic/v1`, apiKey: pxAiKey }),
    google: createGoogleGenerativeAI({ baseURL: `${PX}/api/gemini/v1beta`, apiKey: pxAiKey }),
  };
}

export async function hello(pxAiKey: string): Promise<string> {
  const result = await generateText({ model: providers(pxAiKey).google('gemini-2.5-flash'), prompt: 'Say hi' });
  console.log(result.response.headers?.['x-px-request-id']);
  return result.text;
}
```

Always pass `apiKey`. When it is `undefined`, the AI SDK reads
`OPENAI_API_KEY`, `ANTHROPIC_API_KEY` or `GOOGLE_GENERATIVE_AI_API_KEY` and
sends a real provider key, which the proxy refuses (`401`).

### Google Gen AI SDK (`@google/genai`)

```ts
import { GoogleGenAI } from '@google/genai';

export async function hello(pxAiKey: string): Promise<string | undefined> {
  const ai = new GoogleGenAI({
    apiKey: pxAiKey,
    httpOptions: { baseUrl: 'https://ai-proxy.playxoft.com/api/gemini', timeout: 90_000 },
  });
  const response = await ai.models.generateContent({ model: 'gemini-2.5-flash', contents: 'Say hi' });
  console.log(response.sdkHttpResponse?.headers?.['x-px-request-id']);
  return response.text;
}
```

- `baseUrl` has no version: the SDK appends its `apiVersion` (default
  `v1beta`; the proxy also forwards `v1` and `v1alpha`). With `/v1beta` in
  `baseUrl` the path doubles and Google answers `404`.
- Not supported through the proxy: file uploads (`ai.files.upload`, File
  Search uploads; their `/upload/…` path gets `404 px_not_found`) and the Live
  API (WebSockets). Send media inline (base64).
- `ApiError` has no headers; the proxy's `[px_…]` code is in its `message`.

### Plain fetch

```ts
export async function geminiHello(pxAiKey: string): Promise<unknown> {
  const res = await fetch('https://ai-proxy.playxoft.com/api/gemini/v1beta/models/gemini-2.5-flash:generateContent', {
    method: 'POST',
    headers: { 'x-goog-api-key': pxAiKey, 'content-type': 'application/json' },
    body: JSON.stringify({ contents: [{ parts: [{ text: 'Say hi' }] }] }),
  });
  if (!res.ok) {
    throw new Error(`AI proxy ${res.status} ${res.headers.get('x-px-error') ?? ''} (request ${res.headers.get('x-px-request-id')})`);
  }
  return res.json();
}
```

## 3. Reading the trace headers

Every response has `x-px-request-id`. Log it with every AI failure and quote
it in bug reports.

```ts
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';
import { APICallError } from 'ai';

/** Reads a header from an SDK error, whether it gives a Headers object or a plain record. */
function headerOf(headers: unknown, name: string): string | undefined {
  if (headers instanceof Headers) return headers.get(name) ?? undefined;
  if (headers && typeof headers === 'object') {
    const value = (headers as Record<string, unknown>)[name];
    return typeof value === 'string' ? value : undefined;
  }
  return undefined;
}

export function describeAiError(err: unknown): string {
  if (err instanceof OpenAI.APIError || err instanceof Anthropic.APIError) {
    return `${err.status} ${headerOf(err.headers, 'x-px-error') ?? 'provider error'} (request ${headerOf(err.headers, 'x-px-request-id') ?? 'n/a'}): ${err.message}`;
  }
  if (APICallError.isInstance(err)) {
    return `${err.statusCode} (request ${err.responseHeaders?.['x-px-request-id'] ?? 'n/a'}): ${err.message}`;
  }
  return err instanceof Error ? err.message : String(err);
}
```

## 4. Bring your own key (BYOK)

- Your own provider key in the SDK's normal key slot, your proxy key in
  `x-px-key`. The project must have `allowByok` (else
  `403 px_byok_not_allowed`).
- A provider key without `x-px-key`: `401 px_unauthorized`. The proxy is
  never an open relay.
- BYOK calls don't count toward the monthly budget and are never blocked by
  it; they do count toward the rate limit and still obey `allowedModels`.
- Unified endpoint: `Authorization: Bearer <key of the model's vendor>` plus
  `x-px-key`. A key that plainly belongs to another vendor (`sk-ant-…`,
  `AIza…`) gets `400 px_bad_request`.

```ts
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';
import { GoogleGenAI } from '@google/genai';
import { createOpenAI } from '@ai-sdk/openai';

interface Env {
  PX_AI_KEY: string;
  OPENAI_API_KEY: string;
  ANTHROPIC_API_KEY: string;
  GEMINI_API_KEY: string;
}

export function byokClients(env: Env) {
  const pxHeader = { 'x-px-key': env.PX_AI_KEY };
  return {
    openai: new OpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/openai/v1', apiKey: env.OPENAI_API_KEY, defaultHeaders: pxHeader }),
    anthropic: new Anthropic({ baseURL: 'https://ai-proxy.playxoft.com/api/anthropic', apiKey: env.ANTHROPIC_API_KEY, defaultHeaders: pxHeader }),
    gemini: new GoogleGenAI({ apiKey: env.GEMINI_API_KEY, httpOptions: { baseUrl: 'https://ai-proxy.playxoft.com/api/gemini', headers: pxHeader } }),
    aiSdkOpenai: createOpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/openai/v1', apiKey: env.OPENAI_API_KEY, headers: pxHeader }),
  };
}
```

## 5. Unified endpoint

| Endpoint | Method | Use |
|---|---|---|
| `/api/v1/chat/completions` | `POST` | Text, image input, tools, streaming |
| `/api/v1/images/generations` | `POST` | `openai/…` and `google/…` image models |
| `/api/v1/models` | `GET` | Models the project may use |

- Model ids: `openai/<model>`, `anthropic/<model>`, `google/<model>`
  (`gemini/` is an alias; a leading `models/` is dropped). Anything else:
  `400 px_bad_request`.
- Nothing is translated: the call goes to each provider's own
  OpenAI-compatible endpoint with the prefix stripped (OpenAI
  `/v1/chat/completions`, Anthropic `/v1/chat/completions`, Google
  `/v1beta/openai/chat/completions`).
- `POST` only for chat and images (`405 px_method_not_allowed` with `allow`
  otherwise). The body must be a JSON object. The query string isn't
  forwarded. `openai-organization`/`openai-project` headers are dropped for
  Anthropic and Google models.
- Proxy errors come in OpenAI's shape.
- Anthropic through it (Anthropic's OpenAI SDK compatibility layer, which
  Anthropic doesn't recommend for production): `response_format` ignored (no
  JSON mode or structured output); `strict`, `reasoning_effort`, `logprobs`,
  `seed` ignored; `n` must be 1; on Claude 4.7 and later a `temperature` below
  1, a non-default `top_p`, or both set returns `400`; forced tool choice
  errors on Claude Opus 5.5, Sonnet 5.5, Fable 5.1 and Mythos 5.1; audio and
  PDF parts ignored; no prompt caching; thinking text not returned; no image
  generation (`400 px_not_supported`). **For production Claude use the native
  `/api/anthropic` path with the Anthropic SDK.**
- Google through it: still labelled beta by Google; `reasoning_effort` takes
  `low`, `medium`, `high`, `none`; image models such as
  `google/gemini-2.5-flash-image` and `google/gemini-3-pro-image-preview` work
  on `/api/v1/images/generations` with `response_format: 'b64_json'`.
- `GET /api/v1/models`: live lists from each provider the project has a stored
  key for (fetched through a region), ids prefixed `vendor/`, filtered by the
  allowlist, sorted. A provider whose list fails is skipped and reported in a
  non-standard `px_errors` array (`{provider, status, code, message}`). All
  fail and nothing to list: `502 px_upstream_unreachable`. With BYOK, the
  caller's key is never used to list; providers without a stored key add only
  their exact (non-glob) allowlist entries.
- Streamed chat completions: the proxy adds `stream_options.include_usage`
  to log tokens, and removes that usage chunk from your stream if you didn't
  ask for it.

```ts
import OpenAI from 'openai';

export async function ask(pxAiKey: string, model: string, question: string): Promise<string> {
  const client = new OpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/v1', apiKey: pxAiKey });
  const completion = await client.chat.completions.create({ model, messages: [{ role: 'user', content: question }] });
  return completion.choices[0]?.message.content ?? '';
}
// ask(key, 'google/gemini-2.5-flash', '…'), ask(key, 'openai/gpt-5-mini', '…')
```

## 6. Stored prompts

- `x-px-prompt-id: <id>` uses the active version; `<id>@<version>` pins one.
  Ids: 1–64 lowercase letters, digits, `-`, `_`.
- Templates use `{{name}}` placeholders. Values come from the
  `x-px-prompt-vars` header (a JSON object; percent-encode it with
  `encodeURIComponent` when it isn't ASCII) and/or the body field
  `px_prompt_vars` (wins over the header; always removed before forwarding).
  Values are strings, numbers or booleans, inserted once as plain text.
- The prompt goes before any system prompt the caller sent: OpenAI chat → a
  first `system` message; OpenAI Responses → `instructions`; Anthropic
  Messages → `system`; Gemini `generateContent`/`streamGenerateContent` →
  `systemInstruction`; Gemini `countTokens` → wrapped in
  `generateContentRequest`; OpenAI-compatible chat (Anthropic, Google,
  unified) → a first `system` message. Token-count endpoints get it too.
- Errors: `404 px_prompt_not_found` (no such prompt or version),
  `400 px_prompt_variables` (placeholders without values; the message names
  them), `400 px_not_supported` (endpoint doesn't take stored prompts),
  `400 px_bad_request` (bad header, bad vars, non-JSON body).
- Managed by the proxy owner: `node scripts/prompt-put.ts --project <p>
  --prompt <id> --file <path> [--note …] [--no-activate]`, `prompt-list.ts`
  (`--prompt <id> [--show]`), `prompt-activate.ts --version <n>`,
  `prompt-delete.ts`. Templates up to 64 KB; versions never reused; the first
  version of a new prompt is always active; up to 50 versions kept.
- Replaces playxoft.com's `SYSTEM_PROMPT_1…20` environment variables.

```ts
import OpenAI from 'openai';

export async function supportReply(pxAiKey: string, question: string, shop: string): Promise<string> {
  const openai = new OpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/openai/v1', apiKey: pxAiKey });
  const completion = await openai.chat.completions.create(
    { model: 'gpt-5-mini', messages: [{ role: 'user', content: question }] },
    { headers: { 'x-px-prompt-id': 'support-bot', 'x-px-prompt-vars': encodeURIComponent(JSON.stringify({ shop })) } },
  );
  return completion.choices[0]?.message.content ?? '';
}
```

## 7. Limits, budgets and usage

- **`allowedModels`:** `vendor/model` patterns; `*` matches any run of
  characters (including `/`); the whole id must match; case-sensitive;
  `gemini/` reads as `google/`, a leading `models/` is ignored. Empty: every
  model. Every model a call names is checked: the body `model` (a JSON body
  is read whatever its `content-type`; Gemini's too), Gemini's URL model, the
  multipart `model` field, OpenAI realtime session and transcription models
  (`realtime/client_secrets`, `sessions`, `transcription_sessions`, `calls`,
  `calls/{id}/accept`) and `?model=` on `realtime`, `tools[].model` (OpenAI
  Responses, Anthropic), and every request of an Anthropic message batch.
  With a list the check fails closed: a `POST`, `PUT` or `PATCH` that names
  no model the proxy can read is refused ("Can't tell which model this call
  uses"). So name the model in every call: a video, a moderation or an
  Assistants run with no `model`, or a Responses `image_generation` tool with
  none (it would run the default image model), is refused. Only calls that run no
  model may name none. OpenAI: files; uploads and their parts, complete and
  cancel; batches and batch cancel; vector stores, their files and file
  batches; fine-tuning job cancel, pause and resume; `responses/{id}/cancel`;
  conversations and their items; containers and their files; realtime call
  hangup, refer and reject. Anthropic: files; `messages/batches/{id}/cancel`.
  Gemini: files; `cachedContents`; batch and operation cancels;
  `openai/files`; `openai/batches` and their cancel. A model these calls do
  name is still checked. `GET`, `HEAD` and `DELETE` aren't refused for naming
  none. Also refused: a JSON body that repeats a top-level field, a multipart
  body with more than one `model` or `session` field, and multipart uploads
  over 8 MB (their `model` field isn't read). Known gaps: models inside
  OpenAI batch input files, Gemini inline batch requests and OpenAI
  fine-tuning graders; repeated keys below the top level. Refused:
  `403 px_model_not_allowed`.
- **`rateLimit.perMinute`:** sliding window over every call of the project,
  BYOK included. Over: `429 px_rate_limited` with `retry-after` (seconds).
  Refused calls aren't counted. If the limiter doesn't answer within 1 s the
  call goes through.
- **`monthlyBudgetUsd`:** UTC calendar months; only spend on the project's
  stored keys counts (BYOK never does). Cost comes from reported tokens and the
  proxy's price table (a little low, never high; unpriced models count as $0).
  A call counts toward the month it started in. Alerts at 80 % and 100 %.
  At 100 %: `429 px_budget_exceeded` with `x-should-retry: false` (SDKs
  don't retry) until the 1st (UTC) or a raise.
  Soft cap: may overshoot by about a minute of traffic. Budgets are optional:
  without one the project has no spend limit (the default). Budget `0` blocks
  every stored-key call; it never means unlimited (`--budget none` removes a
  budget). Raising the budget unblocks once the change reaches the router (a
  block written for a lower budget is ignored).
- **Usage log (D1):** one row per call: request id, project, provider, model,
  endpoint, region, attempts, status, error code, stream, BYOK, tokens
  (input includes cache reads and writes; output includes reasoning), cost,
  time to first byte, total time, stored prompt used. No prompt or response
  text unless the project sets `logPrompts` (then up to 64 KB each, in a
  separate table). A caller that disconnects cancels the provider call; the
  row gets `client_closed`. Replies of any size record tokens and cost: a
  large JSON reply (for example base64 images) isn't held whole; the proxy
  reads its top-level `usage` (Gemini: `usageMetadata`) as it streams by.

## 8. Regions and failover

| Id | Location | Pinned to | Serves callers in |
|---|---|---|---|
| `us` | Virginia | `aws:us-east-1` | Americas, unknown locations |
| `eu` | Belgium | `gcp:europe-west1` | Europe, Africa, Middle East, West and Central Asia |
| `in` | Mumbai | `gcp:asia-south1` | IN, PK, BD, LK, NP, BT, MV, AF |
| `sg` | Singapore | `aws:ap-southeast-1` | Rest of Asia, incl. Hong Kong, China, Japan |
| `au` | Sydney | `aws:ap-southeast-2` | Oceania |

- Failover order: `us` → eu, au, sg, in; `eu` → us, in, sg, au; `in` → sg,
  eu, au, us; `sg` → in, au, eu, us; `au` → sg, us, in, eu. At most two
  regions are tried (the first two allowed).
- Retried in the next region: provider location errors, `5xx`, unreachable
  region, and streams with no response headers within 20 s. Normal `4xx` are
  returned as they are. Nothing is retried once the reply has started.
- The 20 s limit applies to every streamed call. Gemini models that think for
  a long time may send headers late: `timeout` in `x-px-attempts`, then
  failover or `502 px_all_regions_failed`. Report it with the request id;
  the proxy owner can raise the limit.
- `x-px-region: us|eu|in|sg|au` forces one region, with **no failover**. A
  bad value: `400`; a region outside the project's `allowedRegions`: `403
  px_region_not_allowed`.
- Projects always call the router, never a region hostname.
- Non-streamed calls have no proxy timeout; set one in the SDK if needed.

## 9. Headers

Request (all `x-px-*` headers are removed before forwarding):

| Header | Purpose |
|---|---|
| `x-px-key` | Proxy key when the key slot holds your own provider key (BYOK); also works alone |
| `x-px-region` | Force a region, no failover |
| `x-px-prompt-id` | Stored prompt, `id` or `id@version` |
| `x-px-prompt-vars` | JSON object of prompt values (percent-encoded if not ASCII) |

Response:

| Header | Meaning |
|---|---|
| `x-px-request-id` | Trace id, on every response |
| `x-px-region-used` | Region that answered (comma-separated list on `/api/v1/models`) |
| `x-px-attempts` | `region:outcome[:status]`, comma-separated; outcomes `ok`, `location_error`, `upstream_5xx`, `region_error`, `timeout`, `network_error` |
| `x-px-error` | The `px_*` code, on proxy errors only |
| `x-px-upstream-ms` | Region → provider response headers, in ms |
| `retry-after` | On `429 px_rate_limited` |
| `x-should-retry: false` | On `429 px_budget_exceeded` |

No CORS headers are ever sent; `access-control-*` and `set-cookie` from
providers are removed.

## 10. Error codes

Provider errors pass through unchanged (no `x-px-error`). Proxy errors carry
`x-px-error` and use the caller's SDK shape: OpenAI shape for `/api/openai` and
`/api/v1` (`error.code` is the `px_*` code), Anthropic shape for `/api/anthropic` and
Google shape for `/api/gemini` (message starts with `[px_…]`), plain
`{ "error": { "code", "message" } }` elsewhere.

| Code | Status | Meaning |
|---|---|---|
| `px_bad_request` | 400 | Malformed for the proxy (bad `x-px-region`, unified body or model id, wrong-vendor BYOK key, bad prompt header or vars) |
| `px_unauthorized` | 401 | Missing, invalid or revoked proxy key; or a provider key without `x-px-key` |
| `px_unauthorized` | 401 | `/api/docs/*` or the dashboard API without the team sign-in; on `/api/docs/*`, a missing, wrong, revoked or expired docs token |
| `px_forbidden` | 403 | A dashboard change by a viewer, or without the dashboard's own checks |
| `px_project_disabled` | 403 | Project switched off |
| `px_browser_forbidden` | 403 | Browser call (`Origin`, `Sec-Fetch-Site`, `Sec-Fetch-Dest` or `OPTIONS`) |
| `px_byok_not_allowed` | 403 | Own provider key sent, project lacks `allowByok` |
| `px_provider_key_missing` | 403 | No stored key for this provider |
| `px_model_not_allowed` | 403 | Model not in `allowedModels`, or the project has a list and the proxy can't tell which model the call uses |
| `px_region_not_allowed` | 403 | Forced region not allowed, or no usable region |
| `px_not_found` | 404 | Unknown path (wrong base URL, Gemini `/upload/…`, unknown API version) |
| `px_method_not_allowed` | 405 | Wrong method on a unified endpoint |
| `px_payload_too_large` | 413 | Body over 25 MB |
| `px_rate_limited` | 429 | Project rate limit; see `retry-after` |
| `px_budget_exceeded` | 429 | Monthly budget reached; don't retry |
| `px_prompt_not_found` | 404 | No such stored prompt or version |
| `px_prompt_variables` | 400 | Stored prompt placeholders without values |
| `px_not_supported` | 400 | Stored prompt on an unsupported endpoint; image generation with `anthropic/` |
| `px_upstream_unreachable` | 502 | A region couldn't reach the provider; or no model list could be fetched |
| `px_all_regions_failed` | 502 | Every region tried failed without a provider answer |
| `px_region_unreachable` | 502 | Defined, not returned by the current code |
| `px_internal` | 500 | Proxy misconfiguration or bug; report with the request id |

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401` "Send your proxy key in x-px-key to use your own provider key" | The SDK sent a real provider key, usually because `apiKey` was `undefined` and it fell back to `OPENAI_API_KEY` and friends | Pass `PX_AI_KEY` explicitly; check the secret is set in this environment |
| `401` "Invalid proxy key" | Wrong, rotated or revoked key; or a brand new key not yet everywhere | Check the secret; wait a minute or two after creation; ask the owner |
| `403 px_browser_forbidden` | Called from browser code (or a client component) | Move the call to a server route |
| Google `404` on a path with `v1beta/v1beta` | `@google/genai` `baseUrl` included the version | Use `https://ai-proxy.playxoft.com/api/gemini` |
| Anthropic `404 not_found_error` on `/v1/v1/messages` | Anthropic SDK `baseURL` ended in `/v1` | Use `https://ai-proxy.playxoft.com/api/anthropic` |
| `404 px_not_found` | Base URL missing `/api/` or the version (`/api/openai` instead of `/api/openai/v1`), or an unsupported path | Use the base URLs in section 1 |
| `403 px_model_not_allowed` | Model not in the project's allowlist (nested realtime, tool and batch models count); or "Can't tell which model this call uses" (no readable model, e.g. a call left to the provider's default model, a field sent twice, a multipart upload over 8 MB or a batch request without `params.model`) | Use an allowed model or ask the owner (`GET /api/v1/models` lists them); name the model in every call, once; send a smaller upload |
| `429 px_budget_exceeded` | Monthly budget reached | Ask the owner; don't retry |
| `502 px_all_regions_failed` | Provider or region outage | Retry later; report the request id |
| "User location is not supported" from a provider | The call didn't go through the proxy, or a region was forced with `x-px-region` | Use the proxy base URL; drop `x-px-region` |

## 12. Moving a project onto the proxy

1. Find every AI call: grep for `openai`, `@anthropic-ai/sdk`, `@ai-sdk/`,
   `@google/genai`, `@google/generative-ai`, `generativelanguage`,
   `api.openai.com`, `api.anthropic.com`, `OPENAI_API_KEY`,
   `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`.
2. Browser or mobile calls must first move behind a server endpoint.
3. Ask the proxy owner for a project (stored keys or BYOK, allowed models,
   budget, rate limit, regions) and its key.
4. Keep `PX_AI_KEY` in the project's Xecret environments and push it to
   every deployed environment (section 1).
5. Change only the base URL and the key. Pass the key explicitly. Log
   `x-px-request-id` on failures. Remove location workarounds (custom fetch
   wrappers, region pinning, retry-on-400 loops).
6. Tests mock `fetch` and assert the proxy base URL; they never call the
   proxy for real.
7. Staging first, then production. Keep the old provider key secret until it
   has run cleanly for a while, then remove it.

## 13. Live docs (`PX_DOCS_TOKEN`)

The proxy owner makes read-only docs access tokens (`pxd_` and 40 characters)
in the dashboard under Settings → Access tokens; keep one in Xecret as
`PX_DOCS_TOKEN` and give it to the agent as an environment variable. With it,
fetch the current docs with curl and a bearer header (WebFetch and other URL
fetchers can't send headers):

```sh
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/llms.txt
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/pages
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/pages/migrations/playxoft-com.md
```

| Path | Returns |
|---|---|
| `/api/docs/llms.txt` | The self-contained guide (`guides/llms.txt`), `text/plain` |
| `/api/docs/pages` | JSON `[{ slug, title, section, url, markdownUrl }]`, in the docs' order |
| `/api/docs/pages/<slug>.md` | One page's Markdown, `text/markdown`; slugs can contain `/` |
| `/api/docs/status` | Live health of each region (JSON) |
| `/api/docs/projects` | Each project's providers, allowed models, regions, rate limit and budget (JSON, never keys) |

- Read-only: no dashboard API, no AI calls, no changes.
- `401 px_unauthorized`: missing, wrong, revoked or expired token; ask the
  owner for a new one. `429 px_rate_limited`: too many wrong tokens from your
  network in a minute; wait `retry-after`. `503 px_unavailable`: try again
  shortly.
- Never print, log or paste the token, or put it in a file in a repo.
