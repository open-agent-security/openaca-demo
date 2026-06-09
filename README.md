# openaca-demo

Sample manifests for trying out OpenACA. The examples below use
`openaca` as a shell function for the latest published OpenACA pre-release.

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
openaca() { uvx --from openaca openaca "$@"; }
openaca --version
```

The function fetches the latest published OpenACA pre-release on demand;
no separate install step is needed.

## Fixtures

Each subdirectory is a self-contained agent/MCP project. `cd` into one
and run `openaca scan repo --target .` to see what the scanner does
on that scenario.

### `sample-mcp/` — vulnerability finding

A single MCP server pinned to a vulnerable version. Scanning this
should produce a HIGH-severity finding against `GHSA-3q26-f695-pp76`.

```bash
cd sample-mcp
openaca scan repo --target . --fail-on none
```

Expected output:

```
Target
  host surface: repository
  path: .

Inventory

repo .
└── direct components/
    └── MCPs/ (1)
        └── @cyanheads/git-mcp-server@1.1.0 (stdio via npx) (from mcp.json)  [! GHSA-3q26-f695-pp76]

Findings

Found 1 vulnerability in 1 package.

@cyanheads/git-mcp-server 1.1.0
  location: mcp.json
  fix:      upgrade to >=2.1.5

  HIGH  GHSA-3q26-f695-pp76  fixed in 2.1.5  @cyanheads/git-mcp-server vulnerable to command injection in several tools  [osv.dev]

Summary
  Scanned 1 manifest, 1 component · advisories: 1 · posture: skipped
  sources: osv.dev

Next
  emit Agent BOM: openaca bom repo --target . --output openaca-bom.json
```

### `clean-scan/` — no findings (same package, fixed version)

The same MCP server, pinned to the fixed version. Demonstrates what a
clean scan looks like — and that bumping a single version closes the
finding from the previous fixture.

```bash
cd clean-scan
openaca scan repo --target . --fail-on none
```

Expected output:

```
Target
  host surface: repository
  path: .

Inventory

repo .
└── direct components/
    └── MCPs/ (1)
        └── @cyanheads/git-mcp-server@2.1.5 (stdio via npx) (from mcp.json)

Findings

  No advisories matched the scanned components.
  OpenACA scans agent composition; for general software dependency scans,
  use a general-purpose SCA scanner.

Summary
  Scanned 1 manifest, 1 component · advisories: 0 · posture: skipped
  sources: (none)

Next
  emit Agent BOM: openaca bom repo --target . --output openaca-bom.json
```

### `posture-checks/` — configuration hygiene findings

A manifest with unpinned install references and an insecure
(`http://`) endpoint. Posture findings are off by default; opt in
with `--include-posture` to see them.

```bash
cd posture-checks
openaca scan repo --target . --include-posture --fail-on none
```

Expected output — the unpinned `@modelcontextprotocol/server-filesystem`
reference matches two known advisories (the scanner can't confirm the
installed version, so treat them as "the package you're tracking has
known advisories"), plus three posture findings on the configuration
hygiene side:

```
Target
  host surface: repository
  path: .

Inventory

repo .
└── direct components/
    └── MCPs/ (3)
        ├── @modelcontextprotocol/server-filesystem (stdio via npx, unpinned) (from mcp.json)  [! GHSA-hc55-p739-j48w, GHSA-q66q-fx2p-7w4m]
        ├── http://example.com/mcp (SSE) (from mcp.json)
        └── some-mcp-server (stdio via uvx, unpinned) (from mcp.json)

Findings

Found 2 vulnerabilities in 1 package.

@modelcontextprotocol/server-filesystem
  location: mcp.json
  fix:      upgrade to >=2025.7.1

  HIGH  GHSA-hc55-p739-j48w  fixed in 2025.7.1  @modelcontextprotocol/server-filesystem vulnerability allows for path validation bypass via colliding path prefix  [osv.dev]
  HIGH  GHSA-q66q-fx2p-7w4m  fixed in 2025.7.1  @modelcontextprotocol/server-filesystem allows for path validation bypass via prefix matching and symlink handling  [osv.dev]

Posture findings (configuration hygiene):

  MEDIUM  openaca-posture-insecure-transport  mcp-server/http-endpoint @ http://example.com/mcp
       location: mcp.json
       fix:      Configure the MCP endpoint over https://. Plain http:// exposes prompts, tool calls, and any returned data to network observers and tampering.
       standards: A02:2021, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  PyPI/some-mcp-server (uvx some-mcp-server)
       location: mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  npm/@modelcontextprotocol/server-filesystem (npx @modelcontextprotocol/server-filesystem)
       location: mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025

Summary
  Scanned 1 manifest, 3 components · advisories: 2 · posture: 3
  sources: osv.dev

Next
  emit Agent BOM: openaca bom repo --target . --output openaca-bom.json
```

(Exact wording may shift across pre-release builds; the rule IDs are
stable.)

### `playwright-plugin/` — plugin attribution graph

A Claude Code plugin-shaped fixture that bundles the Playwright MCP
server pinned to a vulnerable historical version. This mirrors the
official Playwright plugin shape but pins the MCP runtime package so
the advisory match is deterministic.

```bash
cd playwright-plugin
openaca scan repo --target . -v --fail-on none
```

Expected inventory shape:

```
repo .
└── plugin/playwright  [! bundles: GHSA-6fg3-hvw7-2fwq]
    └── MCPs/ (1)
        └── @playwright/mcp@0.0.39 (stdio via npx) (from .mcp.json)  [! GHSA-6fg3-hvw7-2fwq]
```

Expected finding:

```
Found 1 vulnerability in 1 package.

@playwright/mcp 0.0.39
  path:     plugin/playwright -> @playwright/mcp
  via:      plugin/playwright
  fix:      upgrade or remove plugin/playwright

  HIGH  GHSA-6fg3-hvw7-2fwq  fixed in 0.0.40  Microsoft Playwright MCP Server vulnerable to DNS Rebinding Attack; Allows Attackers Access to All Server Tools  [osv.dev]
```

For contrast, package-manifest-only scanners will not find package
sources in this fixture because the vulnerable runtime is declared in
agent configuration, not in `package.json` or a lockfile:

```bash
trivy filesystem playwright-plugin --scanners vuln
```

Expected summary:

```
Supported files for scanner(s) not found.
```

## Full-repo demo

Scan the whole repo with verbose output and `--include-posture` to
exercise five V0 capabilities in one run: vulnerability matching, OSV
federation, plugin attribution, source-less remote MCP inventory, and
posture findings.

```bash
openaca scan repo --target . -v --include-posture --fail-on none
```

Expected output:

```
loaded 107 OpenACA overlay(s)
federation: queried 5 target(s) on osv.dev; fetched 4 advisory record(s)
  pkg:npm/%40cyanheads/git-mcp-server@2.1.5
  pkg:npm/%40playwright/mcp@0.0.39
  npm:@modelcontextprotocol/server-filesystem (unpinned)
  PyPI:some-mcp-server (unpinned)
  pkg:npm/%40cyanheads/git-mcp-server@1.1.0
federation: skipped 2 ref(s) without supported OSV.dev query (mcp_server=1, plugin=1)
matched 4 finding(s):
  pkg:npm/%40playwright/mcp@0.0.39 → GHSA-6fg3-hvw7-2fwq (high) via plugin/playwright
  pkg:npm/%40modelcontextprotocol/server-filesystem → GHSA-hc55-p739-j48w (unknown)
  pkg:npm/%40modelcontextprotocol/server-filesystem → GHSA-q66q-fx2p-7w4m (unknown)
  pkg:npm/%40cyanheads/git-mcp-server@1.1.0 → GHSA-3q26-f695-pp76 (high)

Target
  host surface: repository
  path: .

Inventory

repo .
├── plugin/playwright  [! bundles: GHSA-6fg3-hvw7-2fwq]
│   └── MCPs/ (1)
│       └── @playwright/mcp@0.0.39 (stdio via npx) (from playwright-plugin/.mcp.json)  [! GHSA-6fg3-hvw7-2fwq]
└── direct components/
    └── MCPs/ (5)
        ├── @cyanheads/git-mcp-server@1.1.0 (stdio via npx) (from sample-mcp/mcp.json)  [! GHSA-3q26-f695-pp76]
        ├── @cyanheads/git-mcp-server@2.1.5 (stdio via npx) (from clean-scan/mcp.json)
        ├── @modelcontextprotocol/server-filesystem (stdio via npx, unpinned) (from posture-checks/mcp.json)  [! GHSA-hc55-p739-j48w, GHSA-q66q-fx2p-7w4m]
        ├── http://example.com/mcp (SSE) (from posture-checks/mcp.json)
        └── some-mcp-server (stdio via uvx, unpinned) (from posture-checks/mcp.json)

Findings

Found 4 vulnerabilities in 3 packages.

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

@modelcontextprotocol/server-filesystem
  location: posture-checks/mcp.json
  fix:      upgrade to >=2025.7.1

  HIGH  GHSA-hc55-p739-j48w  fixed in 2025.7.1  @modelcontextprotocol/server-filesystem vulnerability allows for path validation bypass via colliding path prefix  [osv.dev]
        confidence: unknown
        Component: mcp_server unpinned-npx
        Source: pkg:npm/%40modelcontextprotocol/server-filesystem
        Active in: claude-code
        Declared by: posture-checks/mcp.json
  HIGH  GHSA-q66q-fx2p-7w4m  fixed in 2025.7.1  @modelcontextprotocol/server-filesystem allows for path validation bypass via prefix matching and symlink handling  [osv.dev]
        confidence: unknown
        Component: mcp_server unpinned-npx
        Source: pkg:npm/%40modelcontextprotocol/server-filesystem
        Active in: claude-code
        Declared by: posture-checks/mcp.json

@playwright/mcp 0.0.39
  location: <demo-repo>/playwright-plugin/.mcp.json
  path:     plugin/playwright -> @playwright/mcp
  via:      plugin/playwright
  fix:      upgrade or remove plugin/playwright

  HIGH  GHSA-6fg3-hvw7-2fwq  fixed in 0.0.40  Microsoft Playwright MCP Server vulnerable to DNS Rebinding Attack; Allows Attackers Access to All Server Tools  [osv.dev]
        confidence: high
        Component: mcp_server playwright
        Source: pkg:npm/%40playwright/mcp@0.0.39
        Declared by: <demo-repo>/playwright-plugin/.mcp.json

Posture findings (configuration hygiene):

  MEDIUM  openaca-posture-insecure-transport  mcp-server/http-endpoint @ http://example.com/mcp
       location: posture-checks/mcp.json
       fix:      Configure the MCP endpoint over https://. Plain http:// exposes prompts, tool calls, and any returned data to network observers and tampering.
       standards: A02:2021, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  PyPI/some-mcp-server (uvx some-mcp-server)
       location: posture-checks/mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025

  LOW  openaca-posture-mutable-install-reference  npm/@modelcontextprotocol/server-filesystem (npx @modelcontextprotocol/server-filesystem)
       location: posture-checks/mcp.json
       fix:      Pin the install reference to an exact version, commit SHA, or Docker digest. Mutable refs (no version, @latest, branch refs, missing digest) can roll forward to unexpected code at any time.
       standards: CWE-1357, Pinned-Dependencies, immutable-references, asi04, mcp04:2025

Summary
  Scanned 5 manifests, 7 components · advisories: 4 · posture: 3
  sources: osv.dev

Next
  emit Agent BOM: openaca bom repo --target . --output openaca-bom.json
```

(`<demo-repo>` stands for your clone's absolute path — the
plugin-rooted manifest currently prints its full path there.)

### What's worth noticing

- **Four vulnerabilities matched** against the OpenACA + OSV corpus —
  two at high confidence (version-pinned packages) and two at unknown
  confidence (an unpinned package whose name has known advisories). The
  same `@cyanheads/git-mcp-server` at `@2.1.5` is clean — one version
  bump closes it.
- **The Playwright finding rolls up through the plugin graph.** The
  plugin header says it bundles `GHSA-6fg3-hvw7-2fwq`, and the MCP leaf
  shows the direct package match.
- **The SSE endpoint** at `example.com/mcp` is inventoried but not
  federation-queryable. Remote MCPs don't have queryable PURLs by
  design; they match only against OpenACA-curated overlays that
  target them by component identity.
- **Unpinned stdio MCPs are matched two ways.** The scanner queries OSV
  for the *package name* (the `(unpinned)` federation lines) and reports
  matches at `unknown` confidence — it can't confirm which version is
  actually installed — and `--include-posture` additionally flags the
  mutable install reference itself (LOW severity).
- **The federation line** reports that 5 targets were queried on OSV
  and 2 refs were skipped — honest signal about coverage scope.
  Advisory federation covers package-pinned stdio MCPs, lockfile
  dependencies, and unpinned package launches (at reduced confidence);
  remote MCPs and plugins rely on OpenACA's overlay corpus or posture
  checks.

## Why these fixtures

These small fixtures cover the things a beta tester wants to see early:

1. **The scanner finds real vulnerabilities** (sample-mcp).
2. **A clean scan still gives confidence the tool ran** (clean-scan —
   inventory line shows "Scanned 1 manifest, 1 component" rather than
   reading like a broken scan).
3. **The configuration-hygiene rules work and are opt-in**
   (posture-checks).
4. **Vulnerable MCP runtime packages are attributed to the plugin that
   bundles them** (playwright-plugin).

After running these, point OpenACA at one of your own repos or your
Claude Code install (`openaca scan endpoint`) and send feedback to the maintainer (vinodkone@gmail.com,
or whatever channel you already use). The openaca repo is private
during the closed beta, so GitHub issues aren't open to external
testers yet — DM is the path.

## License

Apache-2.0. Same as openaca itself. The manifests are intentionally
minimal — feel free to copy them into your own test repos.
