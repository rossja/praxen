<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Exabeam Deckard Agent Security Scanner
**Version 0.6.6**

**A security scanner for AI agents.** The Exabeam Deckard Agent Security Scanner (**Deckard Scanner**) inspects the environment an AI agent runs in — its code, skill files, tool definitions, configuration, and dependencies — and evaluates it against a defined policy. Findings land in a local HTML report. Nothing phones home.

---

## What It Detects

- **Policy-implementation divergence** — the code doesn't do what the policy document says
- **Credential exposure** — secrets in unexpected locations across the workspace
- **Configuration gaps** — auto-approved exec, disabled loop detection, missing rate limits
- **Capability drift** — new tools or outbound destinations not in the authorized baseline
- **Supply chain risk** — unpinned dependencies, unreviewed plugins, unknown provenance
- **MCP server security posture** — evaluated against the OWASP Secure MCP Server guide
- **Planned-but-not-deployed controls** — security plans that haven't made it into production code

All findings are evaluated against a single source of truth: the **Worker Remit** — a markdown document that defines what the agent is authorized to be and do.

---

## What You Need

- **Claude Code CLI**, installed and authenticated
- **An Anthropic API key**
- **A Worker Remit** for the agent you're scanning (or Claude Code can help you write one)

---

## Running a Scan

**There is no install step.** Unzip the release and open a Claude Code session in the unzipped directory — that's the whole setup.

**1. Drop the Deckard directory somewhere Claude Code can see it.**

**2. Optionally, place a `WORKER_REMIT.md` in the same directory.**

The Worker Remit describes what the monitored agent is authorized to do — its tools, channels, counterparties, and behavioral norms. If you include one, Deckard picks it up automatically. If you don't, Claude Code will help you write one before scanning.

**3. Open a Claude Code session and read the scanner skill:**

```
Please read and run skills/environment_scanner.md to scan [agent workspace path].
```

Claude Code reads the workspace, evaluates it against the RAISE framework and Worker Remit, and writes the results to `./reports/`.

**4. Open the report.**

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

## Security Foundations

Deckard's knowledge base is built from:

- **RAISE Framework** — six-category AI security framework from *The Developer's Playbook for Large Language Model Security* by Steve Wilson (O'Reilly Media). See [`docs/RAISE.md`](docs/RAISE.md) for the framework reference, the maturity scoring scale, and how to interpret Deckard scores.
- **OWASP Top 10 for LLM Applications 2025**
- **OWASP Top 10 for Agentic Applications 2026**
- **OWASP Secure MCP Server Development Guide 2026**

Findings are classified against all applicable frameworks simultaneously.

---

## Files

```
deckard/
  README.md                   ← You are here
  DECKARD_SPEC.md             ← Full specification
  WORKER_REMIT.md             ← (optional) drop your pre-built remit here
  WORKER_REMIT_template.md    ← Remit template
  knowledge/                  ← RAISE + OWASP knowledge base
  skills/
    environment_scanner.md    ← The scanner skill prompt
    report_template.html      ← Canonical HTML report template
  docs/
    RAISE.md                  ← RAISE framework reference + maturity scale
  examples/                   ← Example scans against real vulnerable agents
```

---

*The Deckard Scanner is built on the RAISE framework from* ***The Developer's Playbook for Large Language Model Security*** *by Steve Wilson — O'Reilly Media*
