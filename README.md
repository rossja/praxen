<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Exabeam Deckard Agent Security Scanner
**Version 0.7.0**

**A security scanner for AI agents.** The Exabeam Deckard Agent Security Scanner (**Deckard Scanner**) compares an agent's declared policy (its Worker Remit) against whatever evidence is available about that agent — source code in a repository, live memory and log files from a running deployment, or chat transcripts and behavioral records — and reports where observed behavior diverges from declared intent. Findings land in a local HTML report. Nothing phones home.

Deckard is **not just a source-code scanner.** Any artifact that reveals what the agent actually does, has done, or is configured to do is valid input.

---

## What It Detects

Every scan classifies findings against **four industry-standard frameworks simultaneously**, drawing on the curated [knowledge base](knowledge/) that ships with the scanner:

- **OWASP Top 10 for LLM Applications 2025** — every finding applicable to LLM-level risks is tagged with the correct `LLM0X` category and full name (e.g., `LLM01 — Prompt Injection`, `LLM02 — Sensitive Information Disclosure`). Source: [genai.owasp.org/llm-top-10](https://genai.owasp.org/llm-top-10/).
- **OWASP Top 10 for Agentic AI Applications 2026** — agentic-specific patterns are tagged with the correct `ASI0X` category (e.g., `ASI01 — Agent Goal Hijack`, `ASI06 — Memory and Context Poisoning`). Source: [genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/).
- **OWASP Secure MCP Server Development Guide 2026** — when the scanner finds MCP server configuration in the workspace, it applies the full MCP minimum-bar checklist from the OWASP guide. Source: [genai.owasp.org/resource/a-practical-guide-for-secure-mcp-server-development](https://genai.owasp.org/resource/a-practical-guide-for-secure-mcp-server-development/).
- **Responsible AI Software Engineering (RAISE) Framework** — a six-category AI security maturity model. Every scan produces a 0–5 score per category plus a weighted overall posture. See [`docs/RAISE.md`](docs/RAISE.md) for the reference and maturity scale.

### The detection patterns themselves

On top of the framework coverage above, every scan runs these named detections:

- **Policy-implementation divergence** — the code doesn't do what the policy document says
- **Credential exposure** — secrets in unexpected locations across the workspace
- **Configuration gaps** — auto-approved exec, disabled loop detection, missing rate limits
- **Capability drift** — new tools or outbound destinations not in the authorized baseline
- **Supply chain risk** — unpinned dependencies, unreviewed plugins, unknown provenance
- **Declared-but-never-consulted config / secret** — half-wired controls (`WEBHOOK_SECRET` defined, never checked; `ADMIN_TOKEN` declared, never enforced)
- **Empty stub files in security-relevant paths** — planned-but-not-implemented sandboxes, approval gates, redactors
- **Secondary prompt discovery** — session-loaded identity files (`SOUL.md`, `AGENTS.md`, `MEMORY.md`, etc.) audited as system prompts, with compound escalation when a writable session file meets a confirmed injection path (ASI06 memory-poisoning chain)
- **Compound signal reasoning** — individual findings chained together when they combine into a high-severity attack path

All findings are evaluated against a single source of truth: the **Worker Remit** — a markdown document that defines what the agent is authorized to be and do.

---

## What Deckard Scans

The scanner accepts any artifact that reveals what the agent does, has done, or is configured to do. Three common input shapes — used individually or in combination:

| Shape | Example | What Deckard does with it |
|---|---|---|
| **Source repository** | A GitHub repo, agent project directory, plugin source tree | Reads code, configs, skill files, dependencies, prompts; checks code against the remit; runs supply-chain and credential-exposure detectors. |
| **Running deployment** | Live memory files (`MEMORY.md`, `SOUL.md`, daily logs), action logs, postmortems, config files from a deployed instance | Treats live state as the agent's autobiography; scans for behavioral drift, half-wired controls, planned-but-not-deployed gaps, and session-loaded files audited as system prompts. |
| **Behavioral artifacts** | Chat transcripts, email histories, conversation logs, decision records | Treats observed behavior as evidence; checks for control-logic disclosure, identity-claim acceptance without verification, scope drift, and missing escalations. |

The methodology adapts. A repo-only scan covers code-level findings — Manage Your Supply Chain, credential exposure, configuration gaps — but can't observe behavior. A behavior-only scan covers Implement Zero Trust violations the agent demonstrably committed but can't assess code quality the transcript doesn't reveal. A scan with both inputs gets the most complete picture, and the report's confidence levels are calibrated accordingly. See [`examples/`](examples/) for repo-shape scans and the test plan in `tests/README.md` (in the source repo) for the breadth of targets we've validated.

---

## What You Need

- **A coding agent** (tested against Claude Code). The scanner is distributed as a skill file the agent reads and executes — any coding agent capable of tool use and multi-step instruction-following should work.
- **A Worker Remit** for the agent you're scanning (or your coding agent can help you write one).

---

## Running a Scan

Deckard ships as a Claude Code plugin inside a plugin marketplace. You can install it through the marketplace mechanism or run it directly from an unzipped release — both paths work.

### Option A — Install via Claude Code plugin marketplace

From a Claude Code session:

```
/plugin marketplace add Exabeam/deckard
/plugin install deckard@exabeam
```

That's the install step. The skill is now available as `environment-scanner`.

### Option B — Use directly from an unzipped release

Unzip the release somewhere your coding agent can see it. No install step required.

### Write a Worker Remit for the agent you're scanning

Use [`WORKER_REMIT_template.md`](WORKER_REMIT_template.md) as the starting point. Describe what the target agent is authorized to do — its tools, channels, counterparties, and behavioral norms. Save the file as `WORKER_REMIT.md` (or `WORKER_REMIT_<agent>.md`) anywhere your coding agent can read it — typically alongside the agent workspace you're scanning, or in the directory you run the scan from. If you skip this step, your coding agent can help you draft one before the scan.

### Run the scan

Tell your coding agent:

```
Please run the environment-scanner skill to scan [path to the agent's repo, deployment workspace, or behavioral artifacts].
```

(Or if you're using the unzipped release directly, point the agent at `skills/environment-scanner/SKILL.md`.)

Your coding agent reads whatever you provide — source code, live agent files, or transcripts — evaluates it against the RAISE framework and Worker Remit, and writes the results to `./reports/`. The methodology adapts to what's available; categories the input doesn't cover are scored at lower confidence and explicitly noted in the report.

### Open the report

```
./reports/<agent-name>-scan-<timestamp>.html
```

Self-contained static HTML. Open directly in a browser — no server needed.

---

## The Worker Remit

The Worker Remit is the only artifact that needs to be customized per agent. Everything else in the Deckard package is generic.

**The remit is a policy document, not a system description.** Declare the intent — what the agent is for, what it's allowed to do, what it's forbidden to do, who it can communicate with, what requires your approval. You don't need to list tool names, file paths, or implementation details. Deckard reads the actual code and compares it against the policy you've declared.

What makes a rule good is that it states a verifiable constraint on behavior:

- ✓ *"Message bodies must never be retrieved for senders not in the authorized counterparty list"*
- ✗ *"Handle email appropriately"*

The first rule gives Deckard something to check against the code. The second doesn't. Write rules about what the agent *does*, not about how it does it.

A template is included in `WORKER_REMIT_template.md`.

---

## Example Scans

The [`examples/`](examples/) directory contains two real scans against deliberately vulnerable agents from the OWASP Agentic AI CTF and the Damn Vulnerable AI Agent project. Each example includes the Worker Remit we wrote, the human-readable HTML report, and the machine-readable JSON findings — see [`examples/README.md`](examples/README.md) for the full walkthrough.

---

## Tests (source repository only)

The [`tests/`](tests/) directory in the source repository is Deckard's pre-release regression suite. It lists nine test-target agents — ranging from intentionally-vulnerable CTF agents to mature production tools — along with the Worker Remit developed for each and the baseline findings a healthy scan should produce. See [`tests/README.md`](tests/README.md) for the full test plan and release checklist.

The test suite exists so that every Deckard release can be validated against the same agent posture spectrum (intentionally-broken → defense-conscious) and any regression — a missed critical theme, a dropped coverage area, an unexplained weighted-score shift — is visible before the release ships.

*`tests/` is not included in the distribution zip — users of the shipped scanner don't need the regression harness. Contributors and maintainers work from the source repository.*

---

## Files

```
deckard/
  README.md                   ← You are here
  DECKARD_SPEC.md             ← Full specification
  WORKER_REMIT_template.md    ← Starting point for writing your own remit
  .claude-plugin/
    plugin.json               ← Claude Code plugin manifest
    marketplace.json          ← Marketplace catalog for this repo
  skills/
    environment-scanner/
      SKILL.md                ← The scanner skill prompt
      report_template.html    ← Canonical HTML report template
      knowledge/              ← RAISE + OWASP knowledge base loaded by the skill
  docs/
    RAISE.md                  ← RAISE framework reference + maturity scale
  examples/                   ← Example scans against real vulnerable agents
```

