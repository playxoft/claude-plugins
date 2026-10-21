---
name: playxoft-ai-proxy
description: How Playxoft projects call OpenAI, Anthropic (Claude) and Google Gemini through the Playxoft AI proxy at ai-proxy.playxoft.com. Use when writing, reviewing or debugging code that calls an AI provider in a Playxoft project (spendchat, archivizz, xrbeezz, playxoft.com, clik.so or a new one), when adding an AI feature, when moving a project's AI calls onto the proxy, or when an AI call fails with a px_* error code, an x-px-* header, PX_AI_KEY, or "User location is not supported". Covers base URLs per SDK, the PX_AI_KEY secret, bring-your-own-key, the unified /api/v1 endpoint, stored prompts, limits, headers and error codes, and reading the live team docs with a PX_DOCS_TOKEN.
---

# Playxoft AI proxy

Every Playxoft project calls OpenAI, Anthropic and Gemini through one proxy,
`https://ai-proxy.playxoft.com`, instead of calling the providers directly.
The router sends each call through a Worker pinned to a region the providers
serve (US, EU, Mumbai, Singapore, Sydney) and retries in the next region on a
location error, a 5xx or no answer. Calls made from a Cloudflare data center
the providers block (Hong Kong) work. The proxy also holds the provider keys
and enforces each project's models, rate limit and budget.

Projects keep the official SDKs and change two things: the base URL and the
key. The key is the project's proxy key, `pxk_<project>_…`, stored as the
secret `PX_AI_KEY`.

## Rules

1. **Never call a provider directly** from a Playxoft project
   (`api.openai.com`, `api.anthropic.com`,
   `generativelanguage.googleapis.com`). Use the proxy base URLs below.
2. **Server-side only.** The proxy refuses browser calls
   (`403 px_browser_forbidden`, no CORS). Never put `PX_AI_KEY` or any AI key
   in client code, a client component, a `NEXT_PUBLIC_` variable, a Vite
   `define`, or a mobile app build. Browsers and apps call the project's own
   server, which calls the proxy.
3. **Never hard-code a key, never put it on a command line, and never ask
   the user to paste one into the chat.** Read `PX_AI_KEY` from the
   environment. It lives in the project's Xecret environments
   (xecret.playxoft.com). Workers get it with
   `xecret secrets get PX_AI_KEY --plain | npx wrangler secret put PX_AI_KEY`
   (add `--env <name>` per Wrangler environment). Locally:
   `xecret pull --format env -o .dev.vars` for `wrangler dev` (check
   `.dev.vars` is gitignored), or `xecret run -- npm run dev` for Node dev
   servers.
4. **Pass the key explicitly** to every SDK. If `apiKey` is `undefined`, SDKs
   fall back to `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` /
   `GOOGLE_GENERATIVE_AI_API_KEY` and the proxy refuses the real key (`401`).
5. **Log `x-px-request-id`** with every AI failure, and quote it when
   reporting a problem. It is on every response.
6. Don't add region pinning, custom fetch wrappers or retry-on-400 loops for
   location errors: the proxy handles them.
7. Creating projects, issuing keys, changing allowed models, budgets and
   stored prompts is the proxy owner's job (scripts in the
   `AI_Proxy_Playxoft` repo). Say what to ask for; don't work around a
   refusal.
8. Don't invent proxy features. Check [reference.md](reference.md).

## Base URLs

| SDK or client | Base URL |
|---|---|
| OpenAI SDK, `createOpenAI` | `https://ai-proxy.playxoft.com/api/openai/v1` |
| Anthropic SDK (`@anthropic-ai/sdk`) | `https://ai-proxy.playxoft.com/api/anthropic` (the SDK appends `/v1`) |
| `createAnthropic` (Vercel AI SDK), raw HTTP | `https://ai-proxy.playxoft.com/api/anthropic/v1` |
| `createGoogleGenerativeAI` (Vercel AI SDK), raw HTTP | `https://ai-proxy.playxoft.com/api/gemini/v1beta` |
| `@google/genai` | `httpOptions.baseUrl: 'https://ai-proxy.playxoft.com/api/gemini'` (no version; the SDK appends `/v1beta`) |
| Any OpenAI-compatible client, all providers | `https://ai-proxy.playxoft.com/api/v1` with `vendor/model` ids (unified endpoint) |

The provider's own API passes through unchanged: every model, streaming,
tools, structured output, image input and image generation.

Raw Gemini calls: the key goes in the `x-goog-api-key` header, never `?key=`
in the URL (request URLs are logged, so the key would end up in the logs).

## Code

```ts
import Anthropic from '@anthropic-ai/sdk';
import { createAnthropic } from '@ai-sdk/anthropic';
import { createGoogleGenerativeAI } from '@ai-sdk/google';
import { createOpenAI } from '@ai-sdk/openai';
import { GoogleGenAI } from '@google/genai';
import OpenAI from 'openai';

const PX = 'https://ai-proxy.playxoft.com';

// In a Worker pass env.PX_AI_KEY; in Node read process.env.PX_AI_KEY when first needed.
export function aiClients(pxAiKey: string) {
  return {
    openai: new OpenAI({ baseURL: `${PX}/api/openai/v1`, apiKey: pxAiKey }),
    anthropic: new Anthropic({ baseURL: `${PX}/api/anthropic`, apiKey: pxAiKey }),
    gemini: new GoogleGenAI({ apiKey: pxAiKey, httpOptions: { baseUrl: `${PX}/api/gemini`, timeout: 90_000 } }),
    // Vercel AI SDK
    aiOpenai: createOpenAI({ baseURL: `${PX}/api/openai/v1`, apiKey: pxAiKey }),
    aiAnthropic: createAnthropic({ baseURL: `${PX}/api/anthropic/v1`, apiKey: pxAiKey }),
    aiGoogle: createGoogleGenerativeAI({ baseURL: `${PX}/api/gemini/v1beta`, apiKey: pxAiKey }),
  };
}

export async function hello(pxAiKey: string): Promise<string> {
  const { data, response } = await aiClients(pxAiKey)
    .openai.responses.create({ model: 'gpt-5-mini', input: 'Say hi' })
    .withResponse();
  console.log(response.headers.get('x-px-request-id'), response.headers.get('x-px-region-used'));
  return data.output_text;
}
```

Reading the request id elsewhere: Anthropic SDK `.withResponse()` the same
way; Vercel AI SDK `result.response.headers?.['x-px-request-id']` (errors:
`APICallError.responseHeaders`); `@google/genai`
`response.sdkHttpResponse?.headers?.['x-px-request-id']` (its `ApiError` has
no headers; the `[px_…]` code is in `message`). OpenAI and Anthropic SDK
errors: `err.headers` (a `Headers` object or a plain record, depending on
version).

`@google/genai` through the proxy: no file uploads (`ai.files.upload`) and no
Live API; send media inline. Request bodies are limited to 25 MB.

## Bring your own key

When a project keeps its own provider key: put that key in the SDK's normal
key slot and the proxy key in an `x-px-key` header (`defaultHeaders` for the
OpenAI and Anthropic SDKs, `headers` for the Vercel AI SDK,
`httpOptions.headers` for `@google/genai`). The project needs `allowByok`.
BYOK calls don't count toward the budget. Details in
[reference.md](reference.md).

## Other features

- **Unified endpoint:** `POST /api/v1/chat/completions`,
  `POST /api/v1/images/generations`, `GET /api/v1/models` with ids like
  `openai/gpt-5-mini`, `anthropic/claude-opus-5`, `google/gemini-2.5-flash`.
  For production Claude use the native `/api/anthropic` path: Anthropic's
  OpenAI-compatible layer ignores `response_format` and more.
- **Stored prompts:** `x-px-prompt-id: <id>` or `<id>@<version>` adds the
  project's stored system prompt; `{{var}}` values come from
  `x-px-prompt-vars` (JSON, percent-encoded if not ASCII) or the body field
  `px_prompt_vars`.
- **Force a region:** `x-px-region: us|eu|in|sg|au` (no failover). Rarely
  needed.

## Common errors

| Code | Status | Do this |
|---|---|---|
| `px_unauthorized` | 401 | Check `PX_AI_KEY` is set and passed explicitly; for BYOK add `x-px-key` |
| `px_browser_forbidden` | 403 | Move the call to the server |
| `px_model_not_allowed` | 403 | Use an allowed model (`GET /api/v1/models`) or ask the owner. "Can't tell which model this call uses": name the model in every call, once, instead of relying on the provider's default (multipart uploads are read only up to 8 MB) |
| `px_provider_key_missing` | 403 | Ask the owner to store that provider's key, or use BYOK |
| `px_rate_limited` | 429 | Wait `retry-after` seconds (SDKs do this) |
| `px_budget_exceeded` | 429 | Don't retry; ask the owner to raise the budget |
| `px_prompt_variables` | 400 | Send the named prompt values |
| `px_all_regions_failed` | 502 | Retry later; report the request id |

Provider errors pass through unchanged and have no `x-px-error` header. Every
code, plus troubleshooting for wrong base URLs, is in
[reference.md](reference.md).

## Live docs

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

## Adding AI to a project: checklist

1. The call runs on the server.
2. The project has a proxy project, and `PX_AI_KEY` is in its Xecret
   environments and pushed to every deployed environment (ask the owner if
   not).
3. Base URL from the table above; `apiKey: PX_AI_KEY` passed explicitly.
4. The model is allowed for the project.
5. `x-px-request-id` is logged on failure.
6. Tests mock `fetch`; no real calls.
