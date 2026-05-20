# openaca-demo

Sample manifests for trying out OpenACA. Scan them with
`uvx openaca scan repo --target <fixture>` — `uvx` fetches OpenACA
on the fly, no install step needed.

If you're a closed-beta tester, read the
[**beta-tester guide**](./BETA-TESTER-GUIDE.md) first — it covers
install, first scan, what feedback I'm looking for, and how to
report. This repo gives you a few stable targets to point the
scanner at while you're getting your bearings.

## Setup

You need [uv](https://docs.astral.sh/uv/getting-started/installation/).
Then:

```bash
git clone https://github.com/open-agent-security/openaca-demo.git
cd openaca-demo
```

`uvx openaca ...` fetches OpenACA on the fly; no separate install
step.

## Fixtures

Each subdirectory is a self-contained MCP project. `cd` into one and
run `uvx openaca scan repo --target .` to see what the scanner does on
that scenario.

### `sample-mcp/` — vulnerability finding

A single MCP server pinned to a vulnerable version. Scanning this
should produce a HIGH-severity finding against `GHSA-3q26-f695-pp76`.

```bash
cd sample-mcp
uvx openaca scan repo --target .
```

Expected output:

```
Found 1 vulnerability in 1 package.

@cyanheads/git-mcp-server 1.1.0
  location: mcp.json
  fix:      upgrade to >=2.1.5

  HIGH  GHSA-3q26-f695-pp76  fixed in 2.1.5  @cyanheads/git-mcp-server vulnerable to command injection in several tools  [osv.dev]

Scanned 1 manifest, 1 component. Sources: osv.dev.
```

### `clean-scan/` — no findings (same package, fixed version)

The same MCP server, pinned to the fixed version. Demonstrates what a
clean scan looks like — and that bumping a single version closes the
finding from the previous fixture.

```bash
cd clean-scan
uvx openaca scan repo --target .
```

Expected output:

```
Scanned 1 manifest, 1 component — no findings.
OpenACA scans agent composition; for general software dependency scans, use a general-purpose SCA scanner.
```

### `posture-checks/` — configuration hygiene findings

A manifest with unpinned install references and an insecure
(`http://`) endpoint. Posture findings are off by default; opt in
with `--include-posture` to see them.

```bash
cd posture-checks
uvx openaca scan repo --target . --include-posture --fail-on none
```

Expected output — no vulnerability findings, but three posture
findings on the configuration hygiene side:

```
Scanned 1 manifest, 2 components — no findings.

Posture findings (configuration hygiene):

  MEDIUM  openaca-posture-insecure-transport  mcp-server/http-endpoint @ http://example.com/mcp
       fix:      Configure the MCP endpoint over https://. Plain http:// exposes prompts, tool calls, and any returned data to network observers and tampering.

  LOW  openaca-posture-mutable-install-reference  mcp-stdio/npx-unpinned:@modelcontextprotocol/server-filesystem
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest.

  LOW  openaca-posture-mutable-install-reference  mcp-stdio/uvx-unpinned:some-mcp-server
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest.
```

(Exact wording may shift across pre-release builds; the rule IDs are
stable.)

## Full-repo demo

Scan the whole repo with verbose output and `--include-posture` to
exercise four V0 capabilities in one run: vulnerability matching, OSV
federation, source-less remote MCP inventory, and posture findings.

```bash
uvx openaca scan repo --target . -v --include-posture
```

Expected output:

```
loaded 107 OpenACA overlay(s)
scanned 3 manifest(s), 5 component(s):
repo openaca-demo
└── direct components/
    └── MCPs/ (5)
        ├── @cyanheads/git-mcp-server@1.1.0 (stdio via npx)  [! GHSA-3q26-f695-pp76]
        ├── @cyanheads/git-mcp-server@2.1.5 (stdio via npx)
        ├── http://example.com/mcp (SSE)
        ├── npx (stdio, args hidden)
        └── uvx (stdio, args hidden)
federation: queried 2 PURL(s) on osv.dev; fetched 1 advisory record(s)
federation: skipped 3 ref(s) without queryable PURL (mcp_server=3)
matched 1 finding(s):
  pkg:npm/%40cyanheads/git-mcp-server@1.1.0 → GHSA-3q26-f695-pp76 (high)
Found 1 vulnerability in 1 package.

@cyanheads/git-mcp-server 1.1.0
  location: sample-mcp/mcp.json
  fix:      upgrade to >=2.1.5

  HIGH  GHSA-3q26-f695-pp76  fixed in 2.1.5  @cyanheads/git-mcp-server vulnerable to command injection in several tools  [osv.dev]
        taxonomies: owasp_agentic_top10=asi02,asi05
        evidence_level: confirmed
        confidence: high
        Component: mcp_server git
        Source: pkg:npm/%40cyanheads/git-mcp-server@1.1.0
        Active in: claude-code
        Declared by: sample-mcp/mcp.json

Scanned 3 manifests, 5 components. Sources: osv.dev.

Posture findings (configuration hygiene):

  MEDIUM  openaca-posture-insecure-transport  mcp-server/http-endpoint @ http://example.com/mcp
       location: posture-checks/mcp.json
       fix:      Configure the MCP endpoint over https://. Plain http:// exposes prompts, tool calls, and any returned data to network observers and tampering.
       standards: A02:2021, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  mcp-stdio/npx-unpinned:@modelcontextprotocol/server-filesystem (npx @modelcontextprotocol/server-filesystem)
       location: posture-checks/mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  mcp-stdio/uvx-unpinned:some-mcp-server (uvx some-mcp-server)
       location: posture-checks/mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025
```

### What's worth noticing

- **One vulnerability matched** against the OpenACA + OSV corpus. The
  same package at `@2.1.5` is clean — one version bump closes it.
- **The SSE endpoint** at `example.com/mcp` is inventoried but not
  federation-queryable. Remote MCPs don't have queryable PURLs by
  design; they match only against OpenACA-curated overlays that
  target them by component identity.
- **Two stdio MCPs with unpinned install references** surface only
  as posture findings (LOW severity), never as vulnerabilities.
  Without `--include-posture` these are invisible.
- **The federation line** reports that 2 PURLs were queried on OSV
  and 3 refs were skipped — honest signal about coverage scope.
  Advisory federation works for package-pinned stdio MCPs; remote
  and unpinned MCPs rely on OpenACA's overlay corpus.

## Why these three

Three small fixtures cover the three things a beta tester wants to
see early:

1. **The scanner finds real vulnerabilities** (sample-mcp).
2. **A clean scan still gives confidence the tool ran** (clean-scan —
   inventory line shows "Scanned 1 manifest, 1 component" rather than
   reading like a broken scan).
3. **The configuration-hygiene rules work and are opt-in**
   (posture-checks).

After running these, point OpenACA at one of your own repos or your
Claude Code install (`uvx openaca scan endpoint`) and send feedback to
the maintainer (vinodkone@gmail.com, or whatever channel you already
use). The openaca repo is private during the closed beta, so GitHub
issues aren't open to external testers yet — DM is the path.

## License

Apache-2.0. Same as openaca itself. The manifests are intentionally
minimal — feel free to copy them into your own test repos.
