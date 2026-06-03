# OpenACA — Beta Tester Guide

## What OpenACA is

OpenACA is a security scanner + overlay corpus for **Agent Composition
Analysis (ACA)** — identifying the plugins, MCP servers, skills, and
library dependencies that make up an AI agent, and matching them
against known security records (CVE / GHSA / OSV + agent-context
overlays maintained by this project).

It's the AI-agent analogue of Software Composition Analysis (SCA):
SCA tells you what's in your application's dependency tree; ACA tells
you what's in your agent's composition.

Two scan modes:

- `openaca scan repo --target <path>` answers *"what agent components
  are declared in this repository?"* — CI gate, PR check.
- `openaca scan endpoint` answers *"what agent tools are installed on
  this machine right now?"* — laptop / runner sweep against the Claude
  Code endpoint and the current project.

Full overview, V0 scope, and example output: https://pypi.org/project/openaca/

## About this beta

Thanks for trying it. This is a closed pre-release beta — goal: surface
the highest-friction gaps before a wider release. What I'm looking for
is below in "Feedback I'm looking for."

## Quick start

You need [uv](https://docs.astral.sh/uv/getting-started/installation/).
Then:

```bash
uvx openaca scan endpoint -v
```

This runs OpenACA against your Claude Code endpoint with verbose
output. For configuration-hygiene checks, add `--include-posture`
(see the "Optional: posture findings" section below).

Want to see what a real vulnerability finding looks like first? Run
the demo in the next section.

## See a real finding first

Before scanning your own environment, run OpenACA against the demo
repo so you know what a real finding looks like.

```bash
git clone https://github.com/open-agent-security/openaca-demo.git
cd openaca-demo
uvx openaca scan repo --target sample-mcp
```

Expected output:

```
Found 1 vulnerability in 1 package.

@cyanheads/git-mcp-server 1.1.0
  location: sample-mcp/mcp.json
  fix:      upgrade to >=2.1.5

  HIGH  GHSA-3q26-f695-pp76  fixed in 2.1.5  @cyanheads/git-mcp-server vulnerable to command injection in several tools  [osv.dev]

Scanned 1 manifest, 1 component. Sources: osv.dev.
```

That confirms install + advisory matching are working, and shows the
finding shape (affected component, manifest, severity, advisory ID,
fix). For the fuller demo — remote MCP inventory, OSV federation
details, and posture findings end-to-end — see the
[openaca-demo README](https://github.com/open-agent-security/openaca-demo).

To see plugin attribution as well, run the Playwright plugin fixture:

```bash
uvx openaca scan repo --target playwright-plugin -v
```

That fixture shows a vulnerability on an MCP runtime package rolling up
through the plugin that bundles it:

```text
claude-plugin/playwright  [! bundles: GHSA-6fg3-hvw7-2fwq]
└── MCPs/ (1)
    └── @playwright/mcp@0.0.39  [! GHSA-6fg3-hvw7-2fwq]
```

## Scan your own environment

Once you've seen the demo run, point OpenACA at one of your own
targets. The scanner doesn't phone home; everything runs locally.

Run scans with `-v` for verbose output — per-finding component /
source / container context is much more useful than the default
compact view for first impressions.

### Endpoint mode

Scans your installed Claude Code endpoint (`~/.claude` or
`$CLAUDE_CONFIG_DIR`):

```bash
uvx openaca scan endpoint -v
```

This scans **only your user-level Claude config**. To also include
project-local skills (`.claude/skills/`), MCP configs (`.mcp.json`),
and plugin manifests from a project, pass `--project`:

```bash
# Include a specific directory as project context.
uvx openaca scan endpoint --project /path/to/your/agent-project -v
```

### Repo mode

Scans declared manifests in a code repo (`.mcp.json`,
`.claude-plugin/plugin.json`, `.claude/settings.json`, marketplace
registries, lockfiles, etc.) without touching your user-level Claude
install:

```bash
uvx openaca scan repo --target /path/to/your/repo -v
```

## What "no findings" means

"No findings" means **no matching OpenACA overlay or OSV advisory for
the components OpenACA could identify** — not that your environment
is safe. The non-zero inventory line ("Scanned N manifests, M
components") is the success signal that the scanner saw something.

## Optional: Use OpenACA inside Claude Code

If you use Claude Code, you can install the OpenACA plugin for an
inline experience. The plugin is a thin wrapper around the same
`uvx openaca` CLI commands above — it gives Claude four namespaced
slash commands to invoke scans, generate BOMs, and explain findings
without leaving the chat.

Install (requires `openaca` >= 0.1.0b5, which `uvx openaca` pulls
automatically):

```text
/plugin marketplace add open-agent-security/openaca-claude-plugin
/plugin install openaca@openaca
/reload-plugins
```

Commands available after install:

- `/openaca:scan` — run an endpoint or repo scan
- `/openaca:bom` — generate an Agent BOM
- `/openaca:explain` — explain a finding in conversation
- `/openaca:triage` — guided review after agent config changes

The plugin is explicit-invocation only — no hooks, no background
monitors, no auto-modification of Claude Code settings, no upload
of local config anywhere.

Full plugin docs: https://github.com/open-agent-security/openaca-claude-plugin

## Optional: posture findings

Posture findings are configuration-hygiene checks — risky settings
that aren't vulnerabilities. Off by default, opt in with
`--include-posture`. Never affect `--fail-on` exit codes.

```bash
uvx openaca scan repo --target /path/to/repo --include-posture
```

V0 ships four rules:

| Rule | Severity | Flags |
|---|---|---|
| `openaca-posture-insecure-transport` | MEDIUM | MCP endpoints over plain `http://` |
| `openaca-posture-mutable-install-reference` | LOW | Unpinned install refs (`@latest`, no version, branch refs) |
| `openaca-posture-api-endpoint-override` | MEDIUM (HIGH with hardcoded token or model override) | Claude API endpoint overridden in `.claude/settings.json` |
| `openaca-posture-mcp-auto-approve` | MEDIUM | MCP server with auto-approval / consent-bypass enabled |

See [openaca-demo](https://github.com/open-agent-security/openaca-demo)
for example output with both rules firing.

## Inventory glossary

- **Active plugin** — an enabled Claude Code plugin (installed via
  marketplace, manifest, etc.) and the skills, commands, hooks, or
  MCPs it bundles. Bundled components are attributed back to the
  parent plugin in the tree output.
- **Direct component** — a skill, command, hook, or MCP config found
  in your user or project config that isn't attributable to an
  installed plugin (e.g., a hand-authored skill under
  `~/.claude/skills/` or `<repo>/.claude/skills/`).
- **Manifest** — a config file OpenACA parsed to discover components:
  `.mcp.json`, `.claude-plugin/plugin.json`, `.claude/settings.json`,
  `package.json`, `pyproject.toml`, `package-lock.json`, `uv.lock`,
  marketplace registries, etc.

## What V0 covers

| Source | In scope? |
|---|---|
| `~/.claude/settings.json`, skills, commands, agents | Yes |
| Installed Claude Code plugins | Yes |
| Current project's `.claude/` + `.mcp.json` | Yes, with `--project` |
| `mcpServers` (stdio + HTTP/SSE transports) | Yes |
| Claude Desktop config | Partial |
| Cursor / Aider / Continue / Cody / other hosts | Not yet |
| SDK-registered MCPs (`query({ mcpServers })`) | Not yet (V1) |
| claude.ai-managed MCPs (`claude.ai *` in `claude mcp list`) | Not yet — runtime state from your account, not on-disk |

V0 reads declared manifests only — no SDK-inline parsing, no artifact
content scanning. It's not a general-purpose SCA tool; pair with one
for your full dependency tree. Coverage gaps you hit are exactly the
feedback I want.

## Feedback I'm looking for

Three buckets — please tag your report with the closest fit:

1. **Scanner ergonomics** — does install → first-scan → output read
   right? Anywhere you bounced, anything you had to guess at, anything
   the CLI made you do twice.
2. **Coverage gaps** — V0 reads declared manifests only (no SDK-inline
   parsing). What did your environment contain that the scanner
   should have inventoried and didn't? What did it inventory that
   surprised you?
3. **Workflow fit** — where would this slot into your existing
   security tooling? CI gate? Pre-deploy check? Endpoint sweep on a
   schedule? Something else? And what's missing for that fit?

One filed observation is plenty. The friction signal is more valuable
than the polish.

## Privacy & redaction

OpenACA runs locally and doesn't send scan data anywhere. But when
you report back, feel free to redact any sensitive / private data

## How to report

DM me on whatever channel we already use. 

One paragraph is plenty. The fields I find most useful:

- **Feedback type** — ergonomics / coverage gap / workflow fit
- **Command run** — full invocation, e.g. `uvx openaca scan endpoint
  --include-posture`
- **Version** — output of `uvx openaca --version`
- **Expected vs actual** — one line each
- **Output** — redacted as needed (see the privacy note above)
- **Inventory mismatch** — if the scanner missed something it should
  have inventoried, name what + where
