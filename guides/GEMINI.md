<!--
  Paste everything below this comment into the project's GEMINI.md (Gemini CLI
  context file). Source of truth: guides/GEMINI.md in the AI_Proxy_Playxoft
  repo. Re-copy it when that file changes.
-->

## AI provider calls: Playxoft AI proxy

In this project, every call to Google Gemini, OpenAI or Anthropic (Claude)
goes through the Playxoft AI proxy at `https://ai-proxy.playxoft.com`, never
straight to the provider. The proxy calls the provider from a region it
serves (US, EU, Mumbai, Singapore, Sydney), retries in the next region when
one is refused or fails, holds the provider keys, and enforces the project's
allowed models, rate limit and budget. Code uses the official SDKs; only the
base URL and the key differ.

### Rules

- Never call `generativelanguage.googleapis.com`, `api.openai.com` or
  `api.anthropic.com` directly. Use the base URLs below.
- Server-side code only. The proxy refuses browser calls
  (`403 px_browser_forbidden`). Never put an AI key in client code, a
  `NEXT_PUBLIC_` variable, a Vite `define` or a mobile build
  (`--dart-define`).
- The key is the secret `PX_AI_KEY` (`pxk_<project>_…`). Read it from the
  environment; never hard-code it, log it, put it on a command line, or ask
  for it in chat. It lives in the project's Xecret environments
  (xecret.playxoft.com). Workers get it with
  `xecret secrets get PX_AI_KEY --plain | npx wrangler secret put PX_AI_KEY`
  (add `--env <name>` per Wrangler environment). Locally:
  `xecret pull --format env -o .dev.vars` for `wrangler dev` (`.dev.vars`
  gitignored), or `xecret run -- npm run dev` for Node dev servers.
- Always pass the key explicitly. Without it, SDKs fall back to
  `GEMINI_API_KEY` / `GOOGLE_GENERATIVE_AI_API_KEY` / `OPENAI_API_KEY` /
  `ANTHROPIC_API_KEY` and the proxy answers `401`.
- Log the `x-px-request-id` response header with every AI failure and quote
  it when reporting a problem.
- No region pinning or retry loops for "User location is not supported"; the
  proxy handles it.
- Projects, keys, allowed models, budgets and stored prompts are managed by
  the proxy owner in the `AI_Proxy_Playxoft` repo.
- TypeScript only.

### Gemini

`@google/genai`: set `httpOptions.baseUrl` to the proxy's `/api/gemini` prefix
**without** a version. The SDK appends its API version itself (default
`v1beta`; the proxy also forwards `v1` and `v1alpha`) and sends the key in
`x-goog-api-key`.

```ts
import { GoogleGenAI } from '@google/genai';

interface Env {
  PX_AI_KEY: string;
}

export async function hello(env: Env): Promise<string | undefined> {
  const ai = new GoogleGenAI({
    apiKey: env.PX_AI_KEY,
    httpOptions: { baseUrl: 'https://ai-proxy.playxoft.com/api/gemini', timeout: 90_000 },
  });
  const response = await ai.models.generateContent({ model: 'gemini-2.5-flash', contents: 'Say hi' });
  console.log(response.sdkHttpResponse?.headers?.['x-px-request-id']);
  return response.text;
}
```

- `baseUrl` ending in `/v1beta` doubles the version and Google answers `404`.
- File uploads (`ai.files.upload`) and the Live API don't work through the
  proxy. Send images and audio inline (request bodies up to 25 MB).
- `ApiError` has no headers; the proxy's `[px_…]` code is in its `message`.

Vercel AI SDK: `createGoogleGenerativeAI({ baseURL: 'https://ai-proxy.playxoft.com/api/gemini/v1beta', apiKey: env.PX_AI_KEY })`.
Raw HTTP: `POST https://ai-proxy.playxoft.com/api/gemini/v1beta/models/<model>:generateContent`
with `x-goog-api-key: <PX_AI_KEY>`. Never `?key=` in the URL: request URLs
are logged, so the key would end up in the logs.

### OpenAI and Anthropic

| Client | Base URL |
|---|---|
| OpenAI SDK, `createOpenAI` | `https://ai-proxy.playxoft.com/api/openai/v1` |
| Anthropic SDK (appends `/v1` itself) | `https://ai-proxy.playxoft.com/api/anthropic` |
| `createAnthropic`, raw HTTP | `https://ai-proxy.playxoft.com/api/anthropic/v1` |
| Any OpenAI-compatible client, all providers | `https://ai-proxy.playxoft.com/api/v1` with `openai/…`, `anthropic/…`, `google/…` model ids |

```ts
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';

export function clients(pxAiKey: string) {
  return {
    openai: new OpenAI({ baseURL: 'https://ai-proxy.playxoft.com/api/openai/v1', apiKey: pxAiKey }),
    anthropic: new Anthropic({ baseURL: 'https://ai-proxy.playxoft.com/api/anthropic', apiKey: pxAiKey }),
  };
}
```

Request ids: `.withResponse()` on OpenAI and Anthropic SDK calls gives the raw
`Response`; Vercel AI SDK `result.response.headers?.['x-px-request-id']`.

### Bring your own key

If the project keeps its own provider key: that key in the SDK's key slot,
the proxy key in an `x-px-key` header (`httpOptions.headers` for
`@google/genai`, `headers` for the Vercel AI SDK, `defaultHeaders` for the
OpenAI and Anthropic SDKs). The project needs `allowByok`. BYOK calls don't
count toward the budget.

### Unified endpoint and stored prompts

- `POST /api/v1/chat/completions`, `POST /api/v1/images/generations` (for example
  `google/gemini-2.5-flash-image` with `response_format: 'b64_json'`),
  `GET /api/v1/models` (the project's allowed models). For Claude in production,
  use the native `/api/anthropic` path, not the unified endpoint.
- `x-px-prompt-id: <id>` or `<id>@<version>` adds a stored system prompt
  (Gemini: as the first `systemInstruction` part). `{{name}}` values come from
  `x-px-prompt-vars` (JSON, `encodeURIComponent` if not ASCII) or the body
  field `px_prompt_vars`.

### Headers and errors

- Request: `x-px-key`, `x-px-region: us|eu|in|sg|au` (no failover; rarely
  needed), `x-px-prompt-id`, `x-px-prompt-vars`.
- Response: `x-px-request-id`, `x-px-region-used`, `x-px-attempts`,
  `x-px-error` (proxy errors only), `retry-after`, `x-should-retry`.
- Provider errors pass through unchanged. Proxy errors use the provider's
  error shape with the code first in the message (Gemini:
  `{ "error": { "code": 403, "message": "[px_model_not_allowed] …", "status": "PERMISSION_DENIED" } }`).

| Code | Status | Meaning |
|---|---|---|
| `px_unauthorized` | 401 | Missing or invalid proxy key; provider key without `x-px-key` |
| `px_browser_forbidden` | 403 | Called from a browser |
| `px_byok_not_allowed` | 403 | Own key sent, BYOK not allowed |
| `px_provider_key_missing` | 403 | No stored key for this provider |
| `px_model_not_allowed` | 403 | A model the call names isn't allowed for the project (Gemini: the path model and any body `model`, for example on `cachedContents` or `interactions`), or the proxy can't tell which model the call uses (a `POST`, `PUT` or `PATCH` that names no model, other than files, `cachedContents`, OpenAI-compatible batches and cancels; or a field sent twice). Name the model in every call, once |
| `px_not_found` | 404 | Wrong base URL or unsupported path (Gemini `/upload/…`) |
| `px_payload_too_large` | 413 | Body over 25 MB |
| `px_rate_limited` | 429 | Project rate limit; wait `retry-after` |
| `px_budget_exceeded` | 429 | Monthly budget reached; don't retry |
| `px_prompt_not_found` | 404 | No such stored prompt or version |
| `px_prompt_variables` | 400 | Prompt placeholders without values |
| `px_bad_request` | 400 | Bad proxy header, vars or unified model id |
| `px_upstream_unreachable` | 502 | Region couldn't reach the provider |
| `px_all_regions_failed` | 502 | Every region failed; retry later |
| `px_internal` | 500 | Proxy problem; report the request id |

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
