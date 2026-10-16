# DATALYR agent skills

Skills and plugin files for agents that use DATALYR through MCP (`https://api.datalyr.com/mcp`).

| Skill | What it teaches |
|---|---|
| [`datalyr-ads`](skills/datalyr-ads/SKILL.md) | Plan and propose Meta, TikTok and Google Ads ads onto one review page the person uploads from: files, paused campaigns / ad sets / ads, every-placement previews, turning on within spend limits, stopping |

## Install

**Claude Code**

```
/plugin marketplace add datalyr/agent-skills
/plugin install datalyr@datalyr
```

This adds the DATALYR MCP server and the skill. Sign in when Claude Code asks.

**Codex**

```
codex mcp add datalyr --url https://api.datalyr.com/mcp
codex mcp login datalyr
```

Then copy `skills/datalyr-ads` into `~/.codex/skills/`.

**Claude.ai / ChatGPT**

Add the connector `https://api.datalyr.com/mcp` (Claude: Customize → Connectors; ChatGPT: Settings → Apps). The server sends the same rules to every client; the skill adds the creative playbook.

## More

- Setup and tools: https://docs.datalyr.com/datalyr-mcp
- Ads with agents: https://docs.datalyr.com/agent-ads
