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

## Install

OpenACA requires Python 3.11+. The cleanest path also takes care of
the Python prereq for you: install [uv](https://docs.astral.sh/uv/),
then `uv tool install` openaca.

```bash
# 1. Install uv if you don't already have it.
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Install openaca as an isolated CLI tool (uv provisions a
#    matching Python automatically; nothing else to set up).
uv tool install openaca

# 3. Verify.
openaca --version
```

`uv tool install` puts the `openaca` binary in `~/.local/bin/` and
gives it its own venv, so it doesn't touch any of your existing
Python setups. While OpenACA has no stable release yet, uv
auto-picks the latest pre-release without any extra flag.

If you need to reproduce a bug against an exact build, pin it:
`uv tool install openaca==<version>`.

## See a real finding first

Before pointing the scanner at your own environment (which might
legitimately produce no findings), validate the install against the
[openaca-demo](https://github.com/open-agent-security/openaca-demo)
fixtures. This takes 30 seconds and shows what a real finding looks
like, so you know what "working" looks like before interpreting your
own scan.

```bash
git clone https://github.com/open-agent-security/openaca-demo.git
cd openaca-demo
openaca scan repo --target sample-mcp
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

The three fixtures (`sample-mcp/`, `clean-scan/`, `posture-checks/`)
exercise the three things worth seeing early: a real vulnerability
finding, a clean inventory line, and configuration-hygiene
(posture) findings. See the
[openaca-demo README](https://github.com/open-agent-security/openaca-demo)
for all three.

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
openaca scan endpoint -v
```

This scans **only your user-level Claude config**. To also include
project-local skills (`.claude/skills/`), MCP configs (`.mcp.json`),
and plugin manifests from a project, pass `--project`:

```bash
# Include the current directory as project context.
openaca scan endpoint --project . -v

# Or point at a specific project directory.
openaca scan endpoint --project /path/to/your/agent-project -v
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
openaca scan repo --target /path/to/your/repo -v
```

### Common flags (both modes)

- `--sarif results.sarif` — SARIF output for CI integration.
- `--format github` — GitHub Actions annotations.
- `--format json` — JSON to stdout (informational lines go to stderr,
  so the JSON stays clean for piping).
- `--fail-on high|medium|low|none` — exit-code policy (default: `any`).
- `--include-posture` — opt-in to configuration-hygiene rules (see
  the "Posture rules" section below). Off by default.

## What "no findings" actually means

A clean scan reports something like:

```
Scanned 1 manifest, 2 components — no findings.
OpenACA scans agent composition; for general software dependency scans, use a general-purpose SCA scanner.
```

This is a specific claim: **no matching OpenACA overlay or OSV
advisory was found for the components OpenACA was able to identify**.
It is **not** a claim that your environment is safe.

In particular:

- **Coverage scope.** OpenACA matches against its own overlay corpus
  (focused on agent-stack threats) plus OSV/GHSA advisories that have
  queryable package identities (PURLs). Skills, plugins, and hook
  scripts often don't yet have such identities, so a skill-heavy
  inventory can produce zero advisory findings even if vulnerabilities
  exist somewhere in those components.
- **V0 mode is inventory-first for endpoint scans.** Treat the
  inventory output ("you have N components, here's what they are") as
  the primary value of endpoint mode today. Advisory matching is
  strongest in repo mode where lockfiles and `mcp.json` give the
  scanner queryable package identities.
- **A non-zero inventory line is the success signal.** "Scanned 1
  manifest, 2 components — no findings" means the scanner saw your
  components and matched zero advisories. "Scanned 0 manifests, 0
  components" usually means the scanner didn't find what you
  expected — file that as feedback.

## Posture rules (--include-posture)

Posture rules cover **configuration hygiene** — settings that aren't
vulnerabilities but make the agent stack riskier to operate. They're
off by default (because they're advisory, not vulnerability-grade)
and excluded from `--fail-on` so they never break CI by accident.

Turn them on with `--include-posture`:

```bash
openaca scan repo --target /path/to/repo --include-posture
```

V0 ships two posture rules:

| Rule ID | Severity | What it flags |
|---|---|---|
| `openaca-posture-insecure-transport` | MEDIUM | MCP endpoints over plain `http://` instead of `https://`. |
| `openaca-posture-mutable-install-reference` | LOW | Install references without a pinned version (e.g. `npx @scope/pkg` with no version, `uvx some-server`, `@latest`, branch refs). |

If `--include-posture` produces no output, it means none of these
rules matched your config. It is not a no-op — see the
`openaca-demo/posture-checks/` fixture for example output showing
both rules firing.

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

## What V0 covers (and what it doesn't)

### Endpoint mode sources

When you run `openaca scan endpoint`, the scanner looks at:

| Source | In V0 scope? |
|---|---|
| `~/.claude/settings.json` and equivalents | Yes |
| `~/.claude/skills/`, `~/.claude/commands/`, `~/.claude/agents/` | Yes |
| Installed Claude Code plugins (manifest discovery) | Yes |
| Current project's `.claude/skills/`, `.mcp.json`, plugin manifest | Yes (via the default CWD project context) |
| `mcpServers` in `~/.claude/settings.json` or project `.mcp.json` | Yes |
| Claude Desktop config | Partial (same JSON shape; not fully validated against Claude Desktop layouts yet) |
| Cursor / Aider / Continue / Cody / other agent-host configs | Not yet |
| MCPs registered programmatically via SDK (`query({ mcpServers: ... })`) | Not yet (V1) |
| Remote/marketplace MCP servers attached at runtime | Not yet |

If the scan reports `0 MCP servers` and you know you have remote or
SDK-attached MCPs, that's an out-of-scope coverage gap — please
report it so the inventory boundary keeps moving.

### What V0 deliberately does NOT do

- **No SDK-inline parsing.** V0 reads declared manifests; it doesn't
  parse Python or TypeScript source to find programmatically
  registered MCP servers, tools, or hooks. Those are V1 scope.
- **Not a general-purpose SCA scanner.** The corpus is focused on
  agent-stack threats (malicious MCP packages, agent framework
  vulnerabilities, AI infra). Use a general-purpose SCA tool for
  your full dependency tree.
- **No artifact scanning.** OpenACA doesn't unpack or analyze the
  contents of installed skills/plugins — it identifies them and
  matches against the advisory corpus.

### What's in-flight

If you find yourself wishing for something that isn't there yet —
that's exactly the feedback I want. Coverage gaps are the most
useful signal right now.

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
- **Command run** — full invocation, e.g. `openaca scan endpoint
  --include-posture`
- **Version** — output of `openaca --version`
- **Expected vs actual** — one line each
- **Output** — redacted as needed (see the privacy note above)
- **Inventory mismatch** — if the scanner missed something it should
  have inventoried, name what + where

One filed observation is the bar. The friction signal compounds across
the cohort.

## Re-testing after a fix

When I ship a new pre-release with a fix you reported, upgrade to
the newest beta:

```bash
uv tool upgrade openaca
```

`openaca --version` will show what you just got. No need to wait for
me to send a note — `uv tool upgrade` always fetches the latest
available release (currently the newest pre-release).

If you want to go back to the exact build you originally tested
against (e.g., to confirm a fix actually changed behavior):

```bash
uv tool install --force openaca==<version>
```

Substitute the version that's in your bug report.

— Vinod
