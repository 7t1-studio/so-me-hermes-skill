# so-me.studio Hermes Agent skill (registry tap)

The published skill bundle for [Hermes Agent](https://hermes-agent.nousresearch.com/) — drives the [@so-me/cli](https://www.npmjs.com/package/@so-me/cli) binary against your so-me.studio workspace to schedule posts, manage your inbox, generate AI captions/images/videos, and query analytics across 12 social platforms.

## Install

End users register this repo as a Hermes Agent skill source ("tap") and install the skill:

```bash
hermes skills tap add 7t1-studio/so-me-hermes-skill
hermes skills install so-me-studio
```

Hermes will prompt for `SOMESTUDIO_API_KEY` during install (generate one at https://app.so-me.studio/settings/api-keys).

You also need the `@so-me/cli` binary on PATH — install via the [npm package page](https://www.npmjs.com/package/@so-me/cli).

## Repo layout

This repo follows the Hermes Agent tap convention:

```
skills/
└── so-me-studio/
    ├── SKILL.md          hand-authored skill body
    ├── tools.md          auto-generated 143-command catalogue
    └── examples/         3 worked transcripts
```

## Source

Mirrored from the so-me.studio monorepo: https://github.com/7t1-studio/schedular-app/tree/main/apps/agent-kits/hermes-skill

The hand-authored `SKILL.md` and `examples/` are the same body shipped in our [OpenClaw bundle](https://clawhub.ai/Yasin047/so-me-studio); the `tools.md` is auto-generated from the so-me.studio MCP catalogue and stays in sync via the regen script in the source monorepo's [`PUBLISHING.md`](https://github.com/7t1-studio/schedular-app/blob/main/apps/agent-kits/hermes-skill/PUBLISHING.md).

## Other ways to integrate so-me.studio with agents

| Integration | Best for |
|---|---|
| `npm i -g @so-me/hermes-agent` | Standalone CLI runtime for Hermes-4 models via OpenRouter / Together / vLLM / Ollama |
| `mcp_servers:` block in `~/.hermes/config.yaml` | Hermes Agent users who prefer MCP-over-HTTP (no CLI binary needed) |
| **`hermes skills install` (this repo)** | Hermes Agent users who want a registry-installable skill |
| `clawhub install yasin047/so-me-studio` | Claude Desktop / Cursor users (via OpenClaw) |

All four share the same so-me.studio backend and `SOMESTUDIO_API_KEY` auth.

## License

MIT
