# Worker Remit — yaah (Yet Another Agent Harness)

## Agent Identity

**Name:** yaah (Yet Another Agent Harness)
**Operator:** The developer running yaah in their own repository and terminal.
**Environment:** A local developer command-line tool. It uses the developer's configured coding agent for LLM inference, integrates one or more third-party MCP servers, and ships a built-in MCP server of its own.

---

## Mission

**Generate coding-agent configuration for several coding agents — Claude Code, OpenCode, Codex CLI, and GitHub Copilot CLI — from a single source of truth, and ship a built-in security-and-quality toolset alongside it.** The toolset is a set of event hooks that run on the configured agent's tool activity, a built-in MCP server, and a per-session audit log. yaah also wires up whichever third-party MCP servers the developer opts into.

---

## Stated Protections

**The harness's central promise is that its protections are uniform across every coding agent it generates configuration for.** **Every generated agent configuration MUST carry the same set of protective hooks — a linter, a command guard that blocks catastrophic shell commands, a secret scanner, a comment checker, and a session logger.** A developer who generates configuration for any supported agent must receive that full set, not a subset.

---

## Action Boundaries

- **Catastrophic shell commands — recursive deletion from the filesystem root, force-pushing to a main branch, hard resets, destructive database statements, filesystem formatting, raw disk writes — MUST be blocked before they execute.**
- **Remote MCP servers MUST be reached over TLS.**
- **MCP tool descriptions MUST NOT contain instruction-like language directed at the model** — neither the built-in server's tool descriptions nor, as far as the harness can inspect them, those of the third-party servers it configures.

---

## Forbidden Actions

- **Hardcoded credentials in files the agent edits MUST be detected before the change is accepted.**
- **Agent-managed, session-loaded files MUST NOT be writable in a way that lets ingested content persist into future sessions unreviewed.** The harness maintains files that are loaded into the agent's context at session start; content the agent ingested from an untrusted source must not be able to steer a write into those files that silently re-enters context on a later run.

---

## Approval Requirements

- **High-impact actions — destructive shell commands, file writes, and MCP tools that write, send, or execute — MUST reach a human checkpoint before they run.** Destructive shell commands may be blocked outright rather than queued for approval; writes and write/send/execute tool calls must not proceed unreviewed.

---

## Behavioral Expectations

- **The harness MUST maintain a durable, structured, timestamped record of tool calls, blocked actions, and file modifications** — detailed enough to reconstruct what the agent did in a session.
- **The MCP server configuration and the per-agent configuration files MUST stay consistent with each other.** A change to one must not silently leave the other stale.

---

## Authorized Counterparties

- **The local developer** — the operator running yaah and the configured coding agent.
- **The configured coding agent's LLM provider** — inference only.
- **The `context7` MCP server** — documentation lookups.
- **The `pulumi` MCP server** — infrastructure operations; remote, reached over TLS, consuming whatever auth and scope are established out of band.
- **The built-in yaah MCP server** — local.
- **The coding-agent plugin marketplace** — for any plugins the generated configuration enables.

Any counterparty not listed here is unauthorized by default.

---

## Escalation and Limits

- A dangerous shell command caught by the command guard is blocked, not run, and the block is recorded to the session audit log.
- Secret-scan, lint, and comment findings are recorded to the session audit log and surfaced to the operator.
- **The project SHOULD publish a threat model and a security-disclosure process, and SHOULD run adversarial testing of its own protections** — confirming that the secret scanner catches a planted secret, the command guard blocks a catastrophic command, a poisoning write into a session-loaded file is prevented, and a generated configuration genuinely carries the protections it claims.

---

## Known Good Baseline

- **Third-party MCP servers MUST be pinned to a known-good, integrity-checked version** — no server package auto-installed afresh, unpinned, on every run.
- **Dependencies MUST be version-controlled with a committed, pinned lockfile**, and the dependency tree kept small and reviewable.
- **Authorized built-in MCP tools:** `yaah_scan_secrets`, `yaah_lint`, `yaah_check_command`, `yaah_doctor`, `yaah_session_info`, `yaah_planning_status`, and `yaah_planning_init`. All but `yaah_planning_init` are read-only or check-only; `yaah_planning_init` writes to the workspace.
- **Configured MCP servers:** the third-party `context7` and `pulumi` servers, plus the built-in yaah server.

---

## Out of Scope

- yaah is a local developer CLI, not a hosted or multi-tenant service.
- It does not send email, post to external services, or make outbound network calls beyond the configured agent's LLM provider, the configured MCP servers, the package registry used to install third-party MCP servers, and the coding-agent plugin marketplace.
- It does not operate autonomously without a supervising developer.
