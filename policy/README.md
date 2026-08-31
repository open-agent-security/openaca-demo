# Claude Code policy verification for testers

These files help a tester manually verify OpenACA's Claude Code policy
compiler against a real endpoint. A managed-settings artifact is only proven
when Claude Code selects it and the relevant component is unavailable in a
fresh session.

The commands below use the macOS managed-settings location. On Linux, replace
`/Library/Application Support/ClaudeCode` with `/etc/claude-code`.

All tests replace the same drop-in. Recompile and install your normal policy
afterward to restore it.

## Prerequisites

From the root of this repository, define a location for the generated
artifact:

```bash
artifact=/tmp/50-openaca-policy.json
managed_dir="/Library/Application Support/ClaudeCode"
```

Compile a policy and install its artifact with:

```bash
openaca policy compile <policy-file> --target "$HOME/.claude" --host claude --output "$artifact" --managed-settings-dir "$managed_dir"
sudo install -m 644 "$artifact" "$managed_dir/managed-settings.d/50-openaca-policy.json"
```

For every test, start a fresh Claude Code session and run `/status`. Its
**Setting sources** line must identify the file drop-in. If it identifies a
remote or MDM source instead, that higher-priority source is active and this
file is not the effective policy.

## MCP admission

Use `mcp-admission.example.yaml` with two direct MCP servers you already have
configured. Copy the exact command array or URL from `openaca scan endpoint`:
put the server that should remain usable in `allowed`, and the other one in
`blocked`.

After compiling and installing the policy, start Claude Code in a fresh
session and open `/mcp`. The allowed direct MCP should remain available and
the blocked one should not. This fixture defaults all other direct MCP servers
to blocked.

Do not use a plugin-provided MCP for this particular comparison. An admitted
plugin's MCP servers are separately included in Claude's managed MCP allowlist
so the plugin can keep working.

## Explicit plugin block

Edit `plugin-blocked.example.yaml` and replace
`replace-me@marketplace` with the exact identifier of an installed plugin that
you can temporarily disable. Find the identifier in `/plugin` before applying
the policy.

Compile and install the edited policy. In a fresh Claude Code session:

1. `/plugin` → **Installed** must not show the blocked plugin.
2. A slash command supplied by that plugin must be unavailable.

`claude plugins list` is not the assertion for this test: it may still show
the persisted user-scope installation record as enabled even though the
managed policy prevents the plugin from loading.

## Standalone skill block

Install the harmless test skill into Claude Code's direct skill root:

```bash
mkdir -p "$HOME/.claude/skills/openaca-policy-test"
cp policy/standalone-skill/SKILL.md "$HOME/.claude/skills/openaca-policy-test/SKILL.md"
```

Before applying a skill policy, start a fresh Claude Code session and confirm
`/openaca-policy-test` appears in slash-command completion. This is the
baseline.

Then compile and install `skills-blocked.yaml`. In a new session:

1. `/openaca-policy-test` must no longer appear or run.
2. A skill supplied by an admitted plugin must still appear.

This proves the intended boundary: `skills.default: blocked` restricts direct
user or project skills, while skills packaged inside an admitted plugin inherit
that plugin's admission result.

When finished, remove only the fixture you added:

```bash
rm -r -- "$HOME/.claude/skills/openaca-policy-test"
```
