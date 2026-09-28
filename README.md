# SKOS Update Packs

Official distribution point for **Update Packs** for
[Security Knowledge OS](https://github.com/kagioneko/security-knowledge-os)
(SKOS). A pack adds rules to the engine as **data only** - rule YAML, a
manifest, and checksums. It contains no code, and the engine does not
execute anything from it.

[Security Knowledge OS](https://github.com/kagioneko/security-knowledge-os)
に診断ルールを追加する Update Pack の公式配布ページです。パックはデータ
（ルールの YAML・マニフェスト・チェックサム）だけで、コードは含みません。

Pack ZIPs are published on the [Releases](https://github.com/kagioneko/skos-packs/releases) page. This
repository holds only this README and the licence; the pack sources are
maintained separately.

## Packs

| pack | latest | engine | licence | what it checks |
|---|---|---|---|---|
| `mcp` - MCP Connection Risk Pack | [2026.10.0](https://github.com/kagioneko/skos-packs/releases/tag/mcp-2026.10.0) | >= 0.2.0 | Apache-2.0 | risky MCP server configurations, before you connect, along four axes: Capability, Exposure, Impact, Defense |
| `capgraph` - Capability Graph Pack | [2026.10.0](https://github.com/kagioneko/skos-packs/releases/tag/capgraph-2026.10.0) | >= 0.3.0 | Apache-2.0 | dangerous combinations of individually harmless capabilities across a whole agent (all its MCP servers plus the client's built-in tools) |

> Packs are screening aids for risky configurations, **not safety
> guarantees**. A PASS means "none of these checks failed", nothing more.

## Install

```bash
pip install "security-knowledge-os>=0.2.0"

# download mcp-2026.10.0.zip from the Releases page, then:
skos pack inspect mcp-2026.10.0.zip   # manifest and contents; changes nothing
skos pack verify  mcp-2026.10.0.zip   # checksums, signature, compatibility
skos pack install mcp-2026.10.0.zip

skos pack list
skos pack rollback mcp <VERSION>      # switch back to a kept version
skos pack remove mcp
```

`skos pack install` verifies the ZIP again, stages it, runs a smoke test and
only then switches the active version. An update that weakens a rule
(lower severity, changed conditions or checks, a removed rule, ...) is
refused unless you pass `--approve-sensitive`; `skos pack diff OLD.zip
NEW.zip` shows what changed.

## Using the MCP pack

```bash
# one assessment input per server, from your MCP client config
skos scan mcp .mcp.json --out mcp-assessments --assess

# answer the remaining nulls in mcp-assessments/<server>.yaml, then
skos assess mcp-assessments/<server>.yaml --only-pack mcp
```

`skos scan mcp` reads `.mcp.json`, `claude_desktop_config.json` or
`~/.claude.json`. Credentials in the config are only checked for presence;
no value from the config is written or printed. Anything the config does
not show is left `null` and reported as UNKNOWN, not guessed.

| group | id | finding | severity |
|---|---|---|---|
| Capability | MCP-001 | arbitrary shell commands without per-call approval | high |
| | MCP-002 | reaches the whole home directory or `/` without a sandbox | high |
| Exposure | MCP-003 | network-reachable (not loopback) HTTP/SSE server without authentication | critical |
| | MCP-004 | untrusted content (web, email, issues) can steer write/send/shell actions | high |
| | MCP-005 | third-party server whose tool descriptions were not reviewed (tool poisoning) | medium |
| Impact | MCP-006 | credential written literally in the client config | high |
| | MCP-007 | credential broader than the server needs | medium |
| Defense | MCP-008 | locally launched server not pinned to an exact version | medium |
| | MCP-009 | write/shell-capable server without a sandbox | medium |

## Using the capgraph pack

`capgraph` looks at the **whole agent** instead of one server: every MCP
server in a client config plus the client's own built-in tools. Each gets
six capability labels - `untrusted_input`, `sensitive_read`, `egress`,
`exec`, `write`, `persistence` - as `true`, `false` or `null` (unknown), and
the rules flag combinations that are harmless one by one but dangerous
together.

```bash
# needs security-knowledge-os >= 0.3.0
skos pack install capgraph-2026.10.0.zip
skos scan mcp .mcp.json --client claude-code --out mcp-assessments --assess
# answer the questions (the nulls in _agent.yaml), then
skos assess mcp-assessments/_agent.yaml --only-pack capgraph
```

`--client` adds the client's built-in tools (`claude-code`, or `none` for a
client without any). Without it, no capability is claimed absent. The scan
also writes `capability-labels.json` - the agent's labels, the labels of
each server, and which server contributed each label - for runtime
enforcement.

| id | combination | severity |
|---|---|---|
| CAPGRAPH-001 | outside content + reads private data + sends data out (lethal trifecta) | critical |
| CAPGRAPH-002 | outside content + runs code | critical |
| CAPGRAPH-003 | outside content + writes something that runs or loads later | high |
| CAPGRAPH-004 | reads private data + sends data out, no outside content | medium |

A rule passes only when you confirm a gate (human approval or runtime
enforcement) for that flow; while unanswered it stays UNKNOWN with a
question. Finding no dangerous combination does not prove there is none.

## Signatures

Every official pack is signed with Ed25519 (publisher `kagioneko`, key ID
`kagioneko-2026-01`). The public key ships inside the engine, so
`skos pack verify` reports `trust=signed` for an untampered official pack.
A pack that is unsigned, or signed by a key the engine does not trust, is
not installed unless you explicitly pass `--allow-unsigned`. Each release
also lists the ZIP's SHA-256.

## Commercial packs

Paid packs (for example, industry- or customer-specific rule sets) are
planned. They use the same format with `classification: commercial`, and
also need a signed licence file that the engine checks offline (it never
contacts a licence server). Details will be announced
here.

## Contact

contact@kagioneko.com

## Licence

This README: Apache-2.0 (see `LICENSE`). Each pack carries its own licence
in its manifest and `LICENSE.txt`; the `mcp` and `capgraph` packs are Apache-2.0.
