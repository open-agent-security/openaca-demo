# openaca-demo

Sample manifests for trying out [OpenACA](https://github.com/open-agent-security/openaca).

If you're a beta tester, the
[beta-tester guide](https://github.com/open-agent-security/openaca/blob/main/docs/beta-tester-guide.md)
has the full path. This repo gives you a few stable targets to point
the scanner at while you're getting your bearings.

## Setup

```bash
pip install openaca==0.1.0b1
git clone https://github.com/open-agent-security/openaca-demo.git
cd openaca-demo
```

## Fixtures

Each subdirectory is a self-contained MCP project. `cd` into one and
run `openaca scan repo --target .` to see what the scanner does on
that scenario.

### `sample-mcp/` — vulnerability finding

A single MCP server pinned to a vulnerable version. Scanning this
should produce a HIGH-severity finding against `GHSA-3q26-f695-pp76`.

```bash
cd sample-mcp
openaca scan repo --target .
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
openaca scan repo --target .
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
openaca scan repo --target . --include-posture --fail-on none
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
Claude Code install (`openaca scan endpoint`) and send feedback to
the maintainer (vinodkone@gmail.com, or whatever channel you already
use). The openaca repo is private during the closed beta, so GitHub
issues aren't open to external testers yet — DM is the path.

## License

Apache-2.0. Same as openaca itself. The manifests are intentionally
minimal — feel free to copy them into your own test repos.
