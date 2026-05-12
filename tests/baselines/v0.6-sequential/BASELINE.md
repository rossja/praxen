<!--
  Copyright 2026 Exabeam, Inc.
  SPDX-License-Identifier: Apache-2.0
-->

# Baseline — v0.6-sequential (partial: MCP-coverage targets)

Frozen runs of the two **MCP-coverage** test targets — `deepagents-cli` and `yaah` — on the Praxa v0.6.x skill: the canonical-JSON + `render.py` sequential pipeline, `schema_version: "2.0"`, with the "Calibration anchors" scoring discipline.

**This is a partial baseline.** It holds only the targets added when MCP-path coverage was tested on real repos — `langchain-ai/deepagents` (a `.mcp.json` + OAuth/trust-store MCP subsystem) and `dirien/yet-another-agent-harness` (a `.mcp.json` + a `.claude/settings.json` `mcpServers` block + a built-in Go MCP server). The other nine targets are still frozen at [`../v0.3-sequential/`](../v0.3-sequential/BASELINE.md), which remains the comparator for the rest of the suite until a full v0.6 re-freeze of all eleven. When that happens, the nine move here and these two become rows #10 and #11.

## Targets

| Target | Weighted RAISE | Label | Per-category (LYD / BYKB / IZT / MYSC / BART / MC) | Findings (C/H/M/L/Info, total) | Remit (V/G/P/Vague/ENP, total) |
|---|---:|---|---|---|---|
| deepagents-cli | **2.00** | Partial | 2 / 2 / 2 / 3 / 2 / 1 | 1 / 4 / 5 / 2 / 1 — 13 | 6 / 4 / 3 / 0 / 1 — 14 |
| yaah | **2.30** | Partial | 2 / 2 / 2 / 3 / 2 / 3 | 0 / 1 / 4 / 3 / 1 — 9 | 4 / 4 / 5 / 0 / 0 — 13 |

*(LYD = Limit Your Domain, BYKB = Balance Your Knowledge Base, IZT = Implement Zero Trust, MYSC = Manage Your Supply Chain, BART = Build an AI Red Team, MC = Monitor Continuously.)*

### Dominant pattern, per target

- **deepagents-cli** — a capable harness whose strong primitives are opt-in while its riskiest path is the default. Real controls exist (path-scoped filesystem permissions, a tested human-in-the-loop interrupt hook, a SHA-256 fingerprint trust gate for project `.mcp.json`, careful 0600 credential storage, committed `uv.lock` lockfiles + Dependabot) — but none of the high-impact gates are on by default: `execute` runs on the host via the explicitly-unsandboxed `LocalShellBackend`, nothing requires approval before a write or a command, and the CLI's `default_agent_prompt.md` is a session-loaded notes file the agent is told it may rewrite. With external content (file contents, shell output, MCP doc-server results) reaching the model unfiltered, the default config is a one-hop persistence-and-execution chain (PRAX-2026-05-12-001). On the MCP surface specifically the practical exposure today is low — the two declared servers are read-only first-party LangChain doc endpoints over TLS — but the structural gaps are real: no tool-poisoning check on tool descriptions or sanitization of tool outputs, no approval gate on MCP tool calls, and user-level `.mcp.json` files loaded with no trust prompt at all. *Real controls credited, not zeroed* — IZT and BYKB at **2** rather than 0 because path validation and the project-MCP trust gate are operative on the agent's path; MYSC at **3** for the lockfile + exact internal pin + Dependabot; per the bidirectional-calibration anchor.

- **yaah** — the inverse pattern: a security-conscious harness whose own controls are real and operative but unevenly delivered. It ships a deterministic command-guard hook (it actually blocks `rm -rf /`, force-push to main, `DROP TABLE`, `mkfs`, raw disk writes on every Bash call), a 13-pattern secret scanner on every file edit, a structured per-session audit log, a clean built-in MCP server, and exact-pinned Go dependencies — a coherent safety model, so it scores in the Partial band and a notch above a bare harness. The headline gap is consistency: `pkg/generator/hookmap.go` leaves `PreToolUse`/`PostToolUse` blank for the Codex CLI target, so `yaah generate --agent codex` produces a config with none of the "5 hooks out of the box" the README advertises (PRAX-2026-05-12-001, the one **High**). On the MCP surface: the built-in `yaah` server's tool descriptions are clean (credited as a positive, not a finding) and `pulumi` is TLS; the structural gaps are the standard ones plus an unpinned `npx -y @context7/mcp` server (a silent-update vector) and the fact that MCP tool calls aren't covered by the PreToolUse/PostToolUse hooks so they fall outside the session log. *Bidirectional calibration:* MC at **3** for the real structured session audit log; MYSC at **3** for `go.mod` exact pins + `go.sum`; IZT/BYKB at **2** because the command guard and secret scanner are operative on the agent's path.

### MCP-path coverage (why these targets are in the suite)

These two keep the `SKILL.md` Step 6 "MCP Server Evaluation" path under regression. A healthy run on either must: discover the MCP config(s) in Step 4 (`deepagents-cli` has a top-level `.mcp.json`; `yaah` has `.mcp.json` plus an `mcpServers` block in `.claude/settings.json`); load `knowledge/KB_MCP_SECURITY.md`; apply the MCP minimum-bar checklist; and emit findings carrying `{ "kind": "mcp", … }` tags.

- `deepagents-cli`: **5** MCP-tagged findings across PRAX-2026-05-12-005, -006, -007, -011 — "All tool invocations logged with identity and parameters", "Tool descriptions contain no hidden model instructions", "All tool outputs sanitized before returning to model", "High-risk actions require human-in-the-loop approval", "Secrets stored in vault, not in files".
- `yaah`: **6** MCP-tagged findings across PRAX-2026-05-12-002, -003, -004 — "Tool definitions are signed and version-pinned", "Dependencies pinned and scanned for CVEs", "Tool descriptions contain no hidden model instructions", "All tool outputs sanitized before returning to model", "High-risk actions require human-in-the-loop approval", "All tool invocations logged with identity and parameters" — and a *positive* noting the built-in `yaah` MCP server's descriptions are clean (the bidirectional-calibration side of the MCP eval).

If a fresh run on either loses the `mcp` tags or stops surfacing the tool-poisoning / output-sanitization / MCP-approval / supply-chain gaps, the MCP path has regressed.

## Provenance

- **deepagents-cli** — remit: [`../../remits/deepagents-cli.md`](../../remits/deepagents-cli.md). Workspace: a shallow clone of `https://github.com/langchain-ai/deepagents`, scoped to `libs/deepagents` (SDK) + `libs/cli` (`deepagents-cli`) plus top-level config (the top-level `.mcp.json`, `pyproject.toml`/`uv.lock` for both packages, `.github/dependabot.yml`, `AGENTS.md`). Generated 2026-05-12 on the Praxa v0.6.0 skill (`praxa_version: "0.6.0"` in the JSON).
- **yaah** — remit: [`../../remits/yaah.md`](../../remits/yaah.md). Workspace: a shallow clone of `https://github.com/dirien/yet-another-agent-harness`, scoped to the harness itself — `cmd/yaah`, `pkg/{harness,hooks,mcpserver,mcp,session,generator,schema}`, the root `.mcp.json` / `.claude/settings.json` / `go.mod` / `go.sum` / `AGENTS.md`. Generated 2026-05-12 on the Praxa v0.6.1 skill (`praxa_version: "0.6.1"` in the JSON).

Each `<target>/` directory holds `<target>-findings-<date>.json` (the canonical record — the thing the review process diffs), plus `<target>-analysis-<timestamp>.html` and `…-<timestamp>.txt` (deterministically re-renderable from the JSON via `render.py` with the v0.6.x renderer; see [`../README.md`](../README.md)).

## How to compare a fresh run against this baseline

Same procedure as the main baseline (see [`../../README.md`](../../README.md) and [`../v0.3-sequential/BASELINE.md`](../v0.3-sequential/BASELINE.md) → "How to compare"): for each target, weighted RAISE within ±0.3–0.5 of the number above and inside the per-target band in [`../../README.md`](../../README.md); severity counts in the same neighbourhood; dominant pattern intact; **and** the MCP-path coverage above still holds (the `mcp` tags and the MCP-specific findings are present, and `yaah`'s built-in-server-descriptions-clean positive is still credited).
