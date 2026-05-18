# OpenACA — Beta Tester Guide

Thanks for trying OpenACA. This is a closed beta on the `0.1.0b2`
pre-release. Goal: surface the highest-friction gaps before a wider
release.

## Prerequisites — Python 3.11+

OpenACA requires Python 3.11 or newer. Check what you have:

```bash
python3 --version
```

If your `python3` is 3.10 or older (macOS ships 3.9 as the system
Python), install a newer one:

- **Homebrew** (macOS, Linuxbrew): `brew install python@3.13`, then
  use `python3.13`.
- **uv** (cross-platform; installs Python without touching your
  system one): `curl -LsSf https://astral.sh/uv/install.sh | sh`,
  then `uv python install 3.13`.
- **pyenv** (any platform): `pyenv install 3.13`.
- **Direct installer**: https://www.python.org/downloads/ for an
  official package.

Any of these works; pick whichever fits your existing setup.

## Install

```bash
python3.13 -m pip install openaca==0.1.0b2   # or python3.11 / 3.12, etc.
openaca --version
# expected: openaca, version 0.1.0b2
```

If `python3` already resolves to 3.11+, plain `pip install
openaca==0.1.0b2` works.

## First scan

Point it at one endpoint or repo you already maintain — your own
Claude Code install, an agent project, an MCP server you author. The
scanner doesn't phone home; everything runs locally.

Run scans with `-v` for verbose output — per-finding component /
source / container context is much more useful than the default
compact view for first impressions.

**Endpoint mode** — scans your installed Claude Code endpoint
(`~/.claude` or `$CLAUDE_CONFIG_DIR`). Usually the fastest "is this
useful?" check, since most testers already have Claude Code running:

```bash
openaca scan endpoint -v
```

**Repo mode** — scans declared manifests in a code repo (`.mcp.json`,
`.claude-plugin/plugin.json`, `.claude/settings.json`, marketplace
registries, etc.):

```bash
openaca scan repo --target /path/to/your/repo -v
```

Both modes accept `--sarif results.sarif`, `--format json`, and
`--include-posture` (configuration-hygiene rules — off by default,
worth turning on if you want first-scan signal even when the corpus
finds no vulnerabilities).

Want to verify the install before pointing it at a real target? The
[fixtures in this repo](./README.md) — `sample-mcp/`, `clean-scan/`,
`posture-checks/` — each exercise one scan flavor end-to-end. Clone
the repo and `openaca scan repo --target <fixture-dir> -v`.

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

## Known limitations in V0

- **Agent-host coverage is narrow.**
  - **Endpoint mode** scans Claude Code only — it expects the
    `~/.claude` (or `$CLAUDE_CONFIG_DIR`) layout. Cursor, Aider,
    Continue, Cody, Claude Desktop, etc. aren't endpoint-supported
    yet.
  - **Repo mode** is anchored on Claude Code's declared manifests
    (`.claude-plugin/plugin.json`, `.claude/settings.json`, plus the
    host-agnostic `.mcp.json` / `mcp.json` that most MCP-aware hosts
    use). Other agent hosts are inventory-supported only via their
    `mcp.json` if they emit one — host-specific config formats
    aren't parsed.
- **Declared manifests only.** V0 reads what's in those config
  files. It does **not** extract MCP servers defined SDK-inline
  (`query({ mcpServers: { ... } })`), tools registered
  programmatically, or anything from source-code parsing. Those are
  V1 scope.
- **Corpus focused on agent-stack threats.** The overlay corpus
  prioritizes malicious-package records for MCP servers, agent
  framework packages, and AI infra. It's not a substitute for a
  general-purpose SCA tool on your whole dependency tree — use both.
- **Posture findings off by default.** The configuration-hygiene
  rules (mutable install references, insecure transport, missing
  remote auth) are opt-in via `--include-posture`. They're separate
  from vulnerability findings and never fail CI by default.
- **In-flight work.** I'll call out specific known gaps in beta
  updates as they ship. If you find yourself wishing for something
  the scanner doesn't do yet, that's exactly the feedback I want
  — say so.

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

## Pinning a version

Since you're testing `0.1.0b2`, your bug report links to a specific
build. If I push fixes and you want to re-test the same scenario:

```bash
pip install --force-reinstall openaca==0.1.0b2
```

For the next pre-release (e.g., `0.1.0b3`), I'll send a note.

— Vinod
