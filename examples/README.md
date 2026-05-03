<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Example Scans

Real analyses from **Praxa** against two deliberately vulnerable AI agents, so you can see Praxa in action.

For each example we followed the standard Praxa analysis workflow:

1. Wrote a `WORKER_REMIT.md` describing the agent's *intended* scope — what a legitimate version of this agent should and shouldn't do, who it can talk to, what requires approval.
2. Ran Praxa against the public source repository.
3. Collected the two showcase artifacts Praxa produces for every analysis (Praxa also writes a `.txt` stdout summary; the HTML and JSON are what we link below).

**HTML vs. JSON:** The `*-analysis.html` file is a human-readable pretty-print of the findings data. The `*-findings.json` file is the same information structured for automated ingestion — use it for dashboards, ticketing, compliance pipelines, or diffing results across analyses.

---

## FinBot — invoice processing agent

**Source:** [OWASP-ASI/finbot-ctf-demo](https://github.com/OWASP-ASI/finbot-ctf-demo) — CineFlow Productions autonomous invoice processor from the OWASP Agentic AI CTF.

Praxa produced 16 findings (8 Critical, 6 High, 2 Medium), weighted RAISE posture 0.75 / 5.0 (Absent) — including unauthenticated goal injection via `/api/admin/finbot/goals`, a runtime-mutable `fraud_detection_enabled` flag toggleable from an unauthenticated endpoint, vendor-supplied invoice description text flowing unfiltered into LLM context, and a compound chain that turns vendor invoice submission into unauthorized payment release.

- [`finbot/WORKER_REMIT.md`](finbot/WORKER_REMIT.md) — intended-scope policy
- [`finbot/finbot-analysis.html`](finbot/finbot-analysis.html) — human-readable analysis report
- [`finbot/finbot-findings.json`](finbot/finbot-findings.json) — machine-readable findings (preferred for automated ingestion)

---

## HelperBot — internal employee assistant

**Source:** [opena2a-org/damn-vulnerable-ai-agent](https://github.com/opena2a-org/damn-vulnerable-ai-agent) — the HelperBot persona from the DVAA training platform.

A general-purpose assistant with `read_file`, `write_file`, and `search_web` tools. Praxa produced 16 findings (5 Critical, 9 High, 2 Medium), weighted RAISE posture 0.45 / 5.0 (Absent) — including `inputValidation:false` wired into the agent config (the remit explicitly requires path validation), `leakSystemPrompt:true` and `acceptFalseHistory:true` exposing prompt and identity manipulation surfaces, no audit logging of tool calls, and a compound chain where unvalidated input reaches `write_file` with no approval gate or audit trail.

- [`helperbot/WORKER_REMIT.md`](helperbot/WORKER_REMIT.md) — intended-scope policy
- [`helperbot/helperbot-analysis.html`](helperbot/helperbot-analysis.html) — human-readable analysis report
- [`helperbot/helperbot-findings.json`](helperbot/helperbot-findings.json) — machine-readable findings (preferred for automated ingestion)
