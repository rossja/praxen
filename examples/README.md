<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Example Scans

Real analyses from **Praxa** against two deliberately vulnerable AI agents, so you can see Praxa in action.

For each example we followed the standard Praxa analysis workflow:

1. Wrote a `WORKER_REMIT.md` describing the agent's *intended* scope — what a legitimate version of this agent should and shouldn't do, who it can talk to, what requires approval.
2. Ran Praxa against the public source repository.
3. Collected the two output artifacts Praxa produces for every scan.

**HTML vs. JSON:** The `*-analysis.html` file is a human-readable pretty-print of the findings data. The `*-findings.json` file is the same information structured for automated ingestion — use it for dashboards, ticketing, compliance pipelines, or diffing results across analyses.

---

## FinBot — invoice processing agent

**Source:** [OWASP-ASI/finbot-ctf-demo](https://github.com/OWASP-ASI/finbot-ctf-demo) — CineFlow Productions autonomous invoice processor from the OWASP Agentic AI CTF.

Praxa produced 15 findings (6 Critical, 6 High, 3 Medium) — including unauthenticated goal injection via `/admin/finbot/goals`, a toggleable fraud detection flag, and a business-context scoring path that silently overrides security flags on "urgent" invoices.

- [`finbot/WORKER_REMIT.md`](finbot/WORKER_REMIT.md) — intended-scope policy
- [`finbot/finbot-analysis.html`](finbot/finbot-analysis.html) — human-readable analysis report
- [`finbot/finbot-findings.json`](finbot/finbot-findings.json) — machine-readable findings (preferred for automated ingestion)

---

## HelperBot — internal employee assistant

**Source:** [opena2a-org/damn-vulnerable-ai-agent](https://github.com/opena2a-org/damn-vulnerable-ai-agent) — the HelperBot persona from the DVAA training platform.

A general-purpose assistant with `read_file`, `write_file`, and `search_web` tools. Praxa produced 8 findings (3 Critical, 3 High, 1 Medium, 1 Informational) — including no input validation on file paths, a system-prompt-embedded API key, and compound indirect-injection risk where web search content enters the LLM context unsanitized.

- [`helperbot/WORKER_REMIT.md`](helperbot/WORKER_REMIT.md) — intended-scope policy
- [`helperbot/helperbot-analysis.html`](helperbot/helperbot-analysis.html) — human-readable analysis report
- [`helperbot/helperbot-findings.json`](helperbot/helperbot-findings.json) — machine-readable findings (preferred for automated ingestion)
