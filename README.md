# Playxoft Claude Code plugins

A [Claude Code](https://claude.com/claude-code) plugin marketplace, plus guides
for other AI tools.

| Plugin | What it does |
|---|---|
| [`playxoft-ai-proxy`](plugins/playxoft-ai-proxy) | Teaches Claude how to call OpenAI, Anthropic and Gemini through the Playxoft AI proxy (`ai-proxy.playxoft.com`): base URLs, the `PX_AI_KEY` proxy key, bring-your-own provider keys, the unified endpoint, stored prompts, headers and error codes |

## Install

Inside a Claude Code session:

```text
/plugin marketplace add playxoft/claude-plugins
/plugin install playxoft-ai-proxy@playxoft
```

Or from your shell:

```sh
claude plugin marketplace add playxoft/claude-plugins
claude plugin install playxoft-ai-proxy@playxoft
```

The repo is public, so this works on any machine with Claude Code. No GitHub
account or access is needed.

- `--scope user` (the default) turns the plugin on in every project on your
  machine. `--scope project` turns it on for one repo by writing it into that
  repo's `.claude/settings.json`; commit that file.
- Get updates with `/plugin marketplace update playxoft`.
- Check it loaded: `claude plugin list`, or type `/` in a session and look for
  `/playxoft-ai-proxy:playxoft-ai-proxy`. The skill also loads by itself when
  Claude works on AI calls.

### Turn it on for a whole team

Commit this as `.claude/settings.json` in a project. Anyone who opens the
project in Claude Code and trusts the folder is offered the marketplace and
the plugin:

```json
{
  "extraKnownMarketplaces": {
    "playxoft": { "source": { "source": "github", "repo": "playxoft/claude-plugins" } }
  },
  "enabledPlugins": { "playxoft-ai-proxy@playxoft": true }
}
```

## What you still need

The plugin holds no keys. To make calls through the proxy you need a project
key (`PX_AI_KEY`, `pxk_…`) from the proxy owner, kept in your secret manager
and never in code. A read-only docs token (`PX_DOCS_TOKEN`, `pxd_…`) is
optional: with it, Claude reads the live docs instead of the copy in the
plugin.

## Other AI tools

[`guides/`](guides) has the same guide for tools that don't load Claude Code
plugins:

| File | For |
|---|---|
| [`guides/AGENTS.md`](guides/AGENTS.md) | OpenAI Codex, Cursor and other tools that read `AGENTS.md`: paste the section into your project's `AGENTS.md` |
| [`guides/GEMINI.md`](guides/GEMINI.md) | Gemini CLI: paste the section into your project's `GEMINI.md` |
| [`guides/llms.txt`](guides/llms.txt) | ChatGPT, Claude.ai and Gemini on the web: upload it to a Project, Custom GPT or Gem |

## Contributing

This repo is generated from the Playxoft AI proxy's source repo and published
with its `npm run plugin:publish`, so changes made here directly are
overwritten. Open an issue for fixes or ideas.

## License

[MIT](LICENSE)
