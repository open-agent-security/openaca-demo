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
# Include the current directory as project context.
uvx openaca scan endpoint --project . -v

# Or point at a specific project directory.
uvx openaca scan endpoint --project /path/to/your/agent-project -v
```

The scanner prints a one-line reminder on every endpoint scan that
omits `--project`, so you don't need to memorize the flag — it'll
tell you. The reminder is suppressed once you pass `--project`.

### Repo mode

Scans declared manifests in a code repo (`.mcp.json`,
`.claude-plugin/plugin.json`, `.claude/settings.json`, marketplace
registries, lockfiles, etc.) without touching your user-level Claude
install:

```bash
uvx openaca scan repo --target /path/to/your/repo -v
```

### Common flags (both modes)

- `--sarif results.sarif` — SARIF output for CI integration.
- `--format github` — GitHub Actions annotations.
- `--format json` — JSON to stdout (informational lines go to stderr,
  so the JSON stays clean for piping).
- `--fail-on high|medium|low|none` — exit-code policy (default: `any`).
- `--include-posture` — opt-in to configuration-hygiene rules (see
  the "Posture rules" section below). Off by default.

## What "no findings" means

"No findings" means **no matching OpenACA overlay or OSV advisory for
the components OpenACA could identify** — not that your environment
is safe. The non-zero inventory line ("Scanned N manifests, M
components") is the success signal that the scanner saw something;
zero inventory usually means the scanner didn't find what you
expected, which is worth reporting back.

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

Terms that show up in scan output and aren't always self-explanatory:

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
you report back, your scan output may contain internal package
names, paths, or component IDs you don't want public:

- **Redact freely.** Replace internal names with `<redacted>` or
  generic placeholders. The shape of the output is more useful than
  the literal contents.
- **SARIF is sometimes easier to redact than text** — it's structured
  JSON, you can drop or rename specific entries cleanly. Run with
  `--sarif results.sarif --format json` and edit before sending.
- **For sensitive output, DM beats filing in public** (see the
  "How to report" section below for the DM path).

## How to report

DM me — email vinodkone@gmail.com, or whatever channel we already
use. The repo is private during this beta, so a GitHub issue path
isn't available to external testers yet.

One paragraph is plenty. The fields I find most useful:

- **Feedback type** — ergonomics / coverage gap / workflow fit
- **Command run** — full invocation, e.g. `uvx openaca scan endpoint
  --include-posture`
- **Version** — output of `uvx openaca --version`
- **Expected vs actual** — one line each
- **Output** — redacted as needed (see the privacy note above)
- **Inventory mismatch** — if the scanner missed something it should
  have inventoried, name what + where

One filed observation is the bar. The friction signal compounds across
the cohort.

— Vinod
