<!--
  Paste everything below this comment into the project's AGENTS.md (OpenAI
  Codex, Cursor, Windsurf, Jules and other tools that read AGENTS.md).
  Source of truth: guides/AGENTS.md in the AI_Proxy_Playxoft repo. Re-copy it
  when that file changes.
-->

## AI provider calls: Playxoft AI proxy

This project calls OpenAI, Anthropic (Claude) and Google Gemini only through
the Playxoft AI proxy, `https://ai-proxy.playxoft.com`. The proxy sends each
call through a Worker pinned to a region the providers serve, retries in the
next region on location errors, 5xx or no answer, holds the provider keys,
and enforces the project's allowed models, rate limit and budget. Code uses
the official SDKs; only the base URL and the key differ.

### Rules

- Never call `api.openai.com`, `api.anthropic.com` or
  `generativelanguage.googleapis.com` directly. Use the base URLs below.
- Server-side code only. The proxy refuses browser calls
  (`403 px_browser_forbidden`, no CORS). Never put an AI key in client code, a
  client component, a `NEXT_PUBLIC_` variable, a Vite `define` or a mobile
  build.
- The key is the secret `PX_AI_KEY` (`pxk_<project>_…`). Read it from the
  environment (`env.PX_AI_KEY` in a Worker, `process.env.PX_AI_KEY` in Node,
  read when first needed). Never hard-code it, log it, put it on a command
  line, or ask for it in chat. It lives in the project's Xecret environments
  (xecret.playxoft.com). Workers get it with
  `xecret secrets get PX_AI_KEY --plain | npx wrangler secret put PX_AI_KEY`
  (add `--env <name>` per Wrangler environment). Locally:
  `xecret pull --format env -o .dev.vars` for `wrangler dev` (`.dev.vars`
  gitignored), or `xecret run -- npm run dev` for Node dev servers.
- Always pass the key explicitly as `apiKey`. When it is `undefined`, SDKs
  fall back to `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` /
  `GOOGLE_GENERATIVE_AI_API_KEY` and the proxy answers `401`.
- Log the `x-px-request-id` response header with every AI failure and include
  it when reporting a problem.
- No region pinning, custom fetch wrappers or retry-on-400 loops for location
  errors; the proxy handles them.
- New projects, keys, allowed models, budgets and stored prompts are managed by
  the proxy owner in the `AI_Proxy_Playxoft` repo. Ask; don't work around a
  refusal.
- TypeScript only.

### Base URLs

| Client | Base URL |
|---|---|
| OpenAI SDK, `createOpenAI` (`@ai-sdk/openai`) | `https://ai-proxy.playxoft.com/api/openai/v1` |
| Anthropic SDK (`@anthropic-ai/sdk`, appends `/v1` itself) | `https://ai-proxy.playxoft.com/api/anthropic` |
| `createAnthropic` (`@ai-sdk/anthropic`), raw HTTP | `https://ai-proxy.playxoft.com/api/anthropic/v1` |
| `createGoogleGenerativeAI` (`@ai-sdk/google`), raw HTTP | `https://ai-proxy.playxoft.com/api/gemini/v1beta` |
| `@google/genai` | `httpOptions.baseUrl: 'https://ai-proxy.playxoft.com/api/gemini'` (no version; the SDK appends `/v1beta`) |
| OpenAI-compatible client, any provider | `https://ai-proxy.playxoft.com/api/v1`, model ids `openai/…`, `anthropic/…`, `google/…` |

Key headers are the SDKs' usual ones (`Authorization: Bearer`, `x-api-key`,
`x-goog-api-key`). For raw Gemini calls, put the key in the `x-goog-api-key`
header, never `?key=` in the URL (request URLs are logged). Request bodies
are limited to 25 MB.

```ts
import Anthropic from '@anthropic-ai/sdk';
import { createGoogleGenerativeAI } from '@ai-sdk/google';
import { GoogleGenAI } from '@google/genai';
import OpenAI from 'openai';

const PX = 'https://ai-proxy.playxoft.com';

export function aiClients(pxAiKey: string) {
  return {
    openai: new OpenAI({ baseURL: `${PX}/api/openai/v1`, apiKey: pxAiKey }),
    anthropic: new Anthropic({ baseURL: `${PX}/api/anthropic`, apiKey: pxAiKey }),
    gemini: new GoogleGenAI({ apiKey: pxAiKey, httpOptions: { baseUrl: `${PX}/api/gemini`, timeout: 90_000 } }),
    aiGoogle: createGoogleGenerativeAI({ baseURL: `${PX}/api/gemini/v1beta`, apiKey: pxAiKey }),
  };
}

export async function hello(pxAiKey: string): Promise<string> {
  const { data, response } = await aiClients(pxAiKey)
    .openai.chat.completions.create({ model: 'gpt-5-mini', messages: [{ role: 'user', content: 'Say hi' }] })
    .withResponse();
  console.log(response.headers.get('x-px-request-id'), response.headers.get('x-px-region-used'));
  return data.choices[0]?.message.content ?? '';
}
```

Request ids elsewhere: Anthropic SDK `.withResponse()`; Vercel AI SDK
`result.response.headers?.['x-px-request-id']`, errors
`APICallError.responseHeaders`; `@google/genai`
`response.sdkHttpResponse?.headers?.['x-px-request-id']` (its errors carry no
headers; the `[px_…]` code is in the message). `@google/genai` file uploads
and the Live API don't work through the proxy; send media inline.

### Bring your own key

If the project keeps its own provider key: the provider key goes in the SDK's
key slot, the proxy key in an `x-px-key` header (`defaultHeaders` for the
OpenAI and Anthropic SDKs, `headers` for the Vercel AI SDK,
`httpOptions.headers` for `@google/genai`). The project needs `allowByok`;
otherwise `403 px_byok_not_allowed`. A provider key without `x-px-key` gets
`401`. BYOK calls don't count toward the budget.

### Unified endpoint

`POST /api/v1/chat/completions`, `POST /api/v1/images/generations`, `GET /api/v1/models`,
with `vendor/model` ids (`gemini/` works as `google/`). Each call goes to the
provider's own OpenAI-compatible endpoint. Anthropic's compatibility layer
ignores `response_format`, `strict`, `reasoning_effort`, `logprobs` and
`seed`, has no image generation, and isn't meant for production: for Claude
in production use the native `/api/anthropic` path. `GET /api/v1/models` lists the
project's allowed models.

### Stored prompts

`x-px-prompt-id: <id>` (active version) or `<id>@<version>` adds the
project's stored system prompt before any system prompt in the request.
`{{name}}` values come from `x-px-prompt-vars` (a JSON object;
`encodeURIComponent` it if not ASCII) or the body field `px_prompt_vars`
(removed before forwarding).

### Headers

- Request: `x-px-key` (BYOK), `x-px-region: us|eu|in|sg|au` (force a region,
  no failover; rarely needed), `x-px-prompt-id`, `x-px-prompt-vars`.
- Response: `x-px-request-id` (always), `x-px-region-used`, `x-px-attempts`
  (`region:outcome[:status]`), `x-px-error` (proxy errors only),
  `retry-after` (rate limit), `x-should-retry: false` (budget).

### Errors

Provider errors pass through unchanged. Proxy errors have `x-px-error` and
the SDK's own error shape (OpenAI: `error.code`; Anthropic and Google: the
message starts with `[px_…]`).

| Code | Status | Meaning |
|---|---|---|
| `px_bad_request` | 400 | Bad `x-px-region`, unified body or model id, prompt header or vars |
| `px_unauthorized` | 401 | Missing or invalid proxy key; provider key without `x-px-key` |
| `px_project_disabled` | 403 | Project switched off |
| `px_browser_forbidden` | 403 | Called from a browser |
| `px_byok_not_allowed` | 403 | Own key sent, BYOK not allowed |
| `px_provider_key_missing` | 403 | No stored key for this provider |
| `px_model_not_allowed` | 403 | A model the call names (nested realtime, tool and batch models too) isn't in the project's allowlist, or the proxy can't tell which model the call uses (for example a call that names no model and would run the provider's default, a field sent twice, or an image edit or transcription upload over 8 MB). Name the model in every call, once |
| `px_region_not_allowed` | 403 | Forced region not allowed |
| `px_not_found` | 404 | Wrong base URL or unsupported path |
| `px_method_not_allowed` | 405 | Wrong method on a unified endpoint |
| `px_payload_too_large` | 413 | Body over 25 MB |
| `px_rate_limited` | 429 | Project rate limit; wait `retry-after` |
| `px_budget_exceeded` | 429 | Monthly budget reached; don't retry |
| `px_prompt_not_found` | 404 | No such stored prompt or version |
| `px_prompt_variables` | 400 | Prompt placeholders without values |
| `px_not_supported` | 400 | Stored prompt on an unsupported endpoint; image generation with `anthropic/` |
| `px_upstream_unreachable` | 502 | Region couldn't reach the provider |
| `px_all_regions_failed` | 502 | Every region failed; retry later |
| `px_internal` | 500 | Proxy problem; report the request id |

Wrong base URLs: `/api/openai` without `/v1` gives `404 px_not_found`; the
Anthropic SDK with `…/api/anthropic/v1` gives Anthropic's `404` on
`/v1/v1/messages`; `@google/genai` with `…/api/gemini/v1beta` gives Google's
`404`.

### Live docs

If `PX_DOCS_TOKEN` is set (a read-only docs access token, `pxd_…`, from
the proxy owner: dashboard → Settings → Access tokens, kept in Xecret), read
the current docs with curl and a bearer header. WebFetch and other URL
fetchers can't send headers.

```sh
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/llms.txt     # the whole guide
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/pages        # every page: slug, title, url, markdownUrl
curl -sS -H "Authorization: Bearer $PX_DOCS_TOKEN" https://ai-proxy.playxoft.com/api/docs/pages/quickstart.md
```

Slugs can contain `/` (`pages/migrations/playxoft-com.md`). Never print or
paste the token. `401`: missing, wrong, revoked or expired token. No token:
this guide is enough.
