<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Exabeam Deckard Agent Security Scanner — Specification

**Version:** 0.7.0
**Status:** Release

---

## 1. Purpose

The Exabeam Deckard Agent Security Scanner (**Deckard Scanner**) is a static analysis tool for AI agents. It inspects the environment an agent runs in and answers a single question:

**Is the environment the agent runs in secure and aligned with its authorized policy?** (environmental integrity)

Most agent security failures trace to environmental causes — a misconfigured tool, an unreviewed skill file, a policy that says one thing while the code does another, a dependency that changed quietly, a credential left in a file. The Deckard Scanner reads the agent's workspace, compares it to a declared policy, and writes findings to a local report. Nothing phones home.

The name reflects the mission. Deckard is not part of the agent. It is the observer.

### Two-layer model

| Layer | What it represents | Deckard's view |
|-------|--------------------|----------------|
| **Policy** | What the agent is authorized to be and do | Worker Remit |
| **Capability** | What tools, channels, and permissions the agent actually has | The agent's workspace (code, config, dependencies) |

The policy layer establishes intent. The capability layer is what's actually running. The scanner's job is to detect divergence between the two.

---

## 2. Design Principles

### 2.1 Remit-first

The Worker Remit is the source of truth for everything Deckard does. Every finding is measured against what the agent was authorized to be and do. Without a remit, Deckard cannot meaningfully evaluate the environment.

### 2.2 Evidence-driven

Every finding must cite specific evidence: a file path, a line number, a code pattern, a configuration value, an observed absence. Deckard does not assert risk — it observes, compares, and reports what it found and where.

Every claim is tagged:
- **Verified** — directly observed in an artifact that was read
- **Inferred** — reasonable conclusion from indirect evidence
- **Unknown** — no evidence available (absence of a control in a production system is itself a finding)

### 2.3 Never reprint secrets

Reports must never contain the literal value of a secret, credential, token, password, or private key — even when the source is already public, even when the value looks like a placeholder. Secrets are referred to by location and pattern only. See `skills/environment_scanner.md` for the full redaction rule.

### 2.4 Least privilege

The Deckard Scanner operates read-only on the agent's workspace. It does not take action on behalf of the agent. It does not modify the agent's code, skill files, configuration, or dependencies. It writes output only to its own `./reports/` directory in the current working directory.

### 2.5 Separation

Deckard is not part of the agent it scans. It runs as a separate Claude Code invocation with its own session and its own output paths. An agent that has been compromised has no ability to interfere with the scanner.

### 2.6 No external dependencies required

Deckard's only hard dependency is Claude Code and its API connection to the LLM. Everything else is local. All findings are written to local files. The HTML report is served from the filesystem and requires no web server. There is no database, message queue, logging infrastructure, or cloud service.

### 2.7 LLM-native logic

Deckard does not encode detection logic as code rules. The skill file and knowledge base — the prompts — are the logic. This is intentional.

The patterns worth catching do not reduce to enumerable rules. "This skill file quietly expands the agent's reach in a direction inconsistent with its remit" is a judgment call. Policy-implementation divergence — where a policy document says one thing and the running code does another — requires reading both and reasoning about whether they match. Traditional detection code cannot do this. An LLM can.

The skill file gives Claude a calibrated framework for these judgments — what signals matter, what they imply, how confident to be — without enumerating every case in advance.

---

## 3. Architecture

```
┌─────────────────────────────────────────┐
│             DECKARD SCANNER             │
│                                         │
│  Invoked in Claude Code by the operator │
│  (reads environment_scanner.md skill)   │
│                                         │
│  reads: agent code, skills, tools,      │
│  config, dependencies, policy docs,     │
│  log files                              │
│              │                          │
│              ▼                          │
│         Worker Remit                    │
│    (the policy to compare against)      │
│              │                          │
│              ▼                          │
│         Analysis                        │
│  (RAISE scoring, remit audit, compound  │
│   signal reasoning, OWASP classification)│
│              │                          │
│              ▼                          │
│         Report Output                   │
│  ./reports/<agent>-scan-<timestamp>.html│
│  ./reports/<agent>-findings-<date>.json │
└─────────────────────────────────────────┘
```

The scanner runs once per invocation. It reads, analyzes, writes two files, and exits. No daemon, no scheduler, no persistent state between runs. If continuous scanning is desired, wrap the invocation in whatever scheduler the operator already uses (cron, launchd, CI, GitHub Action).

---

## 4. The Scanner

### Invocation model

The scanner is a Claude Code skill. The operator runs it by opening a Claude Code session in a directory containing the Deckard package and asking Claude Code to read and execute `skills/environment_scanner.md`. The scanner is an on-demand tool — each invocation performs one full scan and exits.

### Inputs

| Input | Source |
|-------|--------|
| Worker Remit | `WORKER_REMIT.md` in the current directory or alongside the skill file |
| Agent workspace | Path supplied by the operator at invocation time |
| Knowledge base | `knowledge/` directory alongside the skill file |
| Report template | `skills/report_template.html` alongside the skill file |

### Outputs

Both written to `./reports/` relative to the current working directory. The directory is created if it does not exist.

| Output | Filename |
|--------|----------|
| HTML report | `<agent-slug>-scan-<YYYY-MM-DD-HHMMSS>.html` |
| Findings JSON | `<agent-slug>-findings-<YYYY-MM-DD>.json` |

The HTML is a self-contained, static page with inline CSS — it renders correctly when opened as `file://` with no server. The JSON is the machine-readable findings data; use it for ingestion into ticketing, dashboards, or diffing across runs.

### Artifact scope

The scanner reads the agent's actual workspace — everything the agent runs from. This is not a description of the system; it is the system.

| Source | What it provides |
|--------|-----------------|
| Agent skill files and code | Logic the agent executes — the ground truth of what it actually does |
| Tool and API definitions | Capabilities the agent can invoke and their parameters |
| Policy and remit documents | What the agent is supposed to do — compared against what the code does |
| Plugin manifests | Third-party components loaded at runtime |
| Configuration files | Auth, endpoints, model selection, data access, approval policies |
| Credential-adjacent files | Presence of plaintext secrets in unexpected locations |
| Dependency and lock files | Library versions and provenance |
| Action logs and postmortem records | Historical evidence of prior incidents |
| Worker Remit | The authoritative comparison baseline |

### What it detects

The scanner evaluates the workspace against the six RAISE categories and applies named detection patterns on top.

**RAISE scoring** — each category 0–5 with a confidence level:

| RAISE Category | Applied to agent environment |
|----------------|------------------------------|
| Limit Your Domain | Do skill files or tool definitions expand scope beyond the remit? |
| Balance Your Knowledge Base | Are data sources vetted? Does external content enter the context unsanitized? |
| Implement Zero Trust | Are inputs validated? Are outputs filtered? Is exec capability gated? |
| Manage Your Supply Chain | Are dependencies pinned? Is model provenance known? Are plugins vetted? |
| Build an AI Red Team | Is there evidence of adversarial testing? Do found issues lead to change? |
| Monitor Continuously | Does the agent log its actions? Are logs structured for automated detection? |

**Named detection patterns:**

- **Policy-implementation divergence** — the scanner reads the remit and the code, inventories every actionable remit rule, and classifies each as Verified / Gap / Partial / Vague Policy / Enforcement Not Possible.
- **Credential exposure in unexpected locations** — secrets in documentation, config snapshots, action logs, archive artifacts, or example files. (Reported by location and pattern only — never by value.)
- **Planned-but-not-deployed controls** — design docs, TODOs, or architectural notes that describe controls which don't yet exist in the running code.
- **Configuration gap detection** — exec auto-approval, disabled tool-loop detection, missing rate limits, absent logging, overly broad permission scopes.
- **MCP server evaluation** — when MCP configs are discovered, the OWASP Secure MCP Server minimum-bar checklist is applied.
- **Remit-delta analysis** — tools, channels, data sources, or outbound destinations present in code but absent from the remit's authorized lists.
- **Compound signal reasoning** — individual findings that are moderate in isolation but form a critical chain in combination (e.g., external content entering context + auto-approved exec = one-hop external-input-to-shell).

### Positive posture recognition

The scanner reports what is working well, not only what is broken. Controls that are correctly implemented and verified during the scan — specific remit rules, scoped credentials, evidence of adversarial testing, structured action logs, approval gates — are surfaced explicitly alongside findings. Operators need to know where they can rely on existing controls.

---

## 5. The Worker Remit

The Worker Remit is a markdown file describing what the agent is authorized to be and do. It is the policy baseline the scanner compares the agent's actual code and configuration against.

### The remit states policy. Deckard discovers implementation.

The remit is a policy document, not a system description. It declares intent — what the agent is for, what it is forbidden to do, who it is authorized to communicate with, what requires approval. It does not need to list tool names, file paths, or framework versions — Deckard finds those by reading the code.

### Required sections

| Section | Purpose |
|---------|---------|
| Identity and mission | Establishes what the agent is for |
| Authorized capabilities | Tools, data sources, outbound destinations |
| Behavioral constraints | Must-always and must-never rules |
| Authorized counterparties | Who the agent may talk to |
| Human approval requirements | Actions that require sign-off |
| Escalation and scope boundaries | Where the agent must halt or decline |

A template is included in `WORKER_REMIT_template.md`.

### Specificity requirement

Policy rules must be specific enough to be verifiable. A rule that can't be checked can't be enforced.

- **Too vague:** "Handle email appropriately" — no standard to compare code against.
- **Specific enough:** "Message bodies must never be retrieved for senders not in the authorized counterparty list" — Claude can read the trust-check implementation and verify the order of operations.

The test: could Deckard read this rule, read the agent's code, and determine whether the code complies? If the rule is about what the agent *does* (not how it does it), it's the right kind of rule for a remit.

A remit with vague rules produces Low-confidence findings across the board. A remit with specific, testable constraints produces High-confidence, actionable findings.

---

## 6. Finding Schema

All findings use a single JSON schema. The HTML report is a pretty-print of the same data.

```json
{
  "id": "DKRD-YYYY-MM-DD-NNN",
  "timestamp": "<ISO 8601 UTC>",
  "source": "scanner",
  "detector_id": "<snake_case detector name>",
  "raise_category": "<one of the six RAISE categories>",
  "owasp_llm": "<LLM01–LLM10 or null>",
  "owasp_agentic": "<ASI01–ASI10 or null>",
  "severity": "Critical | High | Medium | Low | Informational",
  "confidence": "High | Medium | Low",
  "worker": "<agent name>",
  "summary": "One-line description of the finding.",
  "evidence": [
    "Specific file path, line number, and pattern observed"
  ],
  "policy_reference": [
    "WORKER_REMIT.md → <Section>: \"<exact quoted rule text>\""
  ],
  "posture_score": null,
  "related_findings": [],
  "recommended_action": "Specific, concrete action. Not generic advice.",
  "escalation": "alert | log_only"
}
```

**Notes:**
- `posture_score` is populated only on the summary entry (id ends in `-POSTURE`), carrying the weighted overall score and per-category breakdown. All other findings have `posture_score: null`.
- `related_findings` lists the IDs of other findings that combine with this one (compound signal).
- Every finding that maps to a remit rule must populate `policy_reference` with the exact quoted text, not just a section name.
- Findings are emitted to a single JSON file per scan — the scanner does not maintain an append-only finding store.

### Posture summary entry

Every scan emits exactly one posture summary entry as the first item in the findings array. It carries the weighted overall score and per-category breakdown for machine consumption. Downstream systems can locate it by `id` suffix (`-POSTURE`) or by `detector_id: "raise_posture_summary"`.

```json
{
  "id": "DKRD-YYYY-MM-DD-POSTURE",
  "detector_id": "raise_posture_summary",
  "severity": "Informational",
  "scan_summary": "The dominant finding pattern for this scan, 2–4 sentences of scanner synthesis.",
  "posture_score": {
    "weighted_overall": 1.3,
    "categories": {
      "limit_your_domain":          { "score": 2, "confidence": "High",   "weight": 0.15 },
      "balance_your_knowledge_base":{ "score": 1, "confidence": "High",   "weight": 0.15 },
      "implement_zero_trust":       { "score": 0, "confidence": "High",   "weight": 0.25 },
      "manage_your_supply_chain":   { "score": 2, "confidence": "Medium", "weight": 0.15 },
      "build_an_ai_red_team":       { "score": 1, "confidence": "High",   "weight": 0.15 },
      "monitor_continuously":       { "score": 1, "confidence": "High",   "weight": 0.15 }
    }
  }
}
```

Weights are the scanner's standard RAISE category weighting (Zero Trust 25%, others 15% each). The weighted overall is the sum of `score × weight` across categories, producing a 0.0–5.0 scalar.

The `scan_summary` field carries the same narrative rendered in the HTML report's Scan Summary section. Downstream systems that want a human-readable one-paragraph synthesis without parsing HTML should read this field.

### Severity model

| Severity | Definition |
|----------|------------|
| Critical | Immediate containment warranted. Clear policy violation, credential exposure, or unauthorized destructive capability. |
| High | Significant risk requiring prompt review. Control absent where remit requires it, or compound signal chain to a high-impact action. |
| Medium | Meaningful gap or anomaly requiring scheduled review. |
| Low | Weak signal or early warning. Single isolated event, minor drift. |
| Informational | Baseline observation — scope note, positive posture, or neutral environmental fact. |

### Dual classification

Every finding carries both a RAISE category and, where applicable, OWASP LLM and OWASP Agentic references. This makes findings legible to teams familiar with either taxonomy.

---

## 7. HTML Report

Each scan produces a self-contained HTML report from a canonical template (`skills/report_template.html`). The template is brand-compliant and not subject to per-scan redesign — Step 11 of the scanner skill instructs the scanner to copy the template verbatim and substitute only data placeholders.

**Sections, in order:**

1. **Header** — navy bar with Exabeam-green accent, agent name, scan timestamp, overall status badge
2. **Intro band** — agent name, scan date, orientation text
3. **RAISE Scorecard** — weighted overall hero band plus a 3×2 grid of category cards (score, confidence, weight, rationale)
4. **Remit Coverage** — every actionable remit rule with quoted text, status (Verified / Gap / Partial / Vague Policy / Enforcement Not Possible), and a link to the linked finding
5. **Findings Register** — full findings ordered Critical → High → Medium → Low → Informational. Each card shows severity badge, ID, summary, RAISE/OWASP/ASI tags, quoted policy rule, evidence, and recommended action.
6. **What's Working Well** — verified positive controls
7. **Discovered Log Files** — log files found during the scan, annotated with source/purpose/modification time
8. **Footer** — brand, artifact count, finding counts, Deckard version

The page renders correctly as `file://` — all CSS is inline, no external scripts, no external fonts beyond the declared Arial/Lausanne stack.

---

## 8. Running a Scan

### Prerequisites

- Claude Code CLI installed and authenticated
- An Anthropic API key (used by Claude Code)
- The Deckard package in a directory Claude Code can see
- A Worker Remit for the agent being scanned (or willingness to write one through Claude Code)

No scheduler, daemon, installer, or configuration file is required.

### Steps

1. Drop the Deckard directory anywhere on disk.
2. Optionally place a `WORKER_REMIT.md` next to `skills/environment_scanner.md` (the scanner will find it automatically).
3. Open a Claude Code session in the Deckard directory (or any parent).
4. Tell Claude Code:
   > *"Please read and run skills/environment_scanner.md to scan [agent workspace path]."*
5. The scanner reads the workspace, analyzes it, and writes two files to `./reports/`.
6. Open the HTML report in a browser.

### Re-running

Each invocation is independent. To re-scan after changes, invoke the skill again — a new pair of timestamped files is written to `./reports/`; prior reports are not overwritten.

### Automated scheduling

Deckard does not ship a scheduler. If recurring scans are desired, wrap the Claude Code invocation in whatever scheduler the operator already uses. Example patterns: a nightly cron job running `claude -p "$(cat skills/environment_scanner.md)"`, a GitHub Action on pull request, a launchd timer on macOS. The scanner is stateless across invocations, so scheduling is purely an operator concern.

### Context window pressure on large workspaces

A scan over a large workspace — archived or snapshotted projects, multi-directory trees, 50+ artifacts — can consume enough context that the Claude Code session compresses mid-scan. When this happens, the scanner is designed to degrade gracefully:

- An **interim scorecard** is printed to stdout between Step 9 and Step 10, so the operator sees the RAISE posture even if the session later truncates.
- The **final summary is written to a `.txt` file** in `./reports/` as well as stdout, so the summary survives even if terminal output is lost.

If you're scanning a large archive and want to minimize context pressure, scan one subdirectory at a time and merge the findings JSON files afterward.

---

## 9. Knowledge Base

The scanner's judgments are calibrated by a curated knowledge base in `knowledge/`. These files give Claude the domain vocabulary, risk taxonomy, and pattern recognition needed to produce consistent, well-classified findings across scans.

| File | Contents |
|------|----------|
| `KB_RAISE_SCANNING.md` | RAISE framework scanning methodology — scoring model, artifact intake patterns, signal-to-risk heuristics, inference rules, compound patterns, positive posture signals. Primary calibration file. |
| `KB_LLM_TOP10.md` | OWASP Top 10 for LLM Applications 2025 — distilled to code patterns, behavioral indicators, and cross-category compound risks. |
| `KB_AGENTIC_TOP10.md` | OWASP Top 10 for Agentic Applications 2026 — agentic-specific attack patterns and the ASI taxonomy for classifying findings. |
| `KB_MCP_SECURITY.md` | OWASP Secure MCP Server Development Guide 2026 — MCP-specific vulnerability landscape and minimum-bar checklist. Loaded only when MCP configuration is discovered in the workspace. |

The knowledge base does not implement detection logic. It gives Claude a calibrated framework for recognizing risk patterns, scoring consistently, classifying findings against both RAISE and OWASP taxonomies, and generating grounded recommendations.

---

## 10. Implementation Guidance

### Model selection

The scanner is designed to run on Anthropic's Sonnet-class models. Smaller models do not reliably perform the remit-implementation cross-referencing the scanner requires. The skill file does not hardcode a model — Claude Code selects based on its session configuration.

### Confidence calibration beats threshold tuning

When a finding is uncertain, mark it `"confidence": "Low"` and let it surface in the report rather than suppressing it. A Low-confidence signal that appears repeatedly across scanner runs is valuable signal that would be invisible if suppressed.

### Specificity of the Worker Remit determines scanner quality

Detection quality is directly proportional to remit specificity. A remit that says "handle tasks as directed" produces Low-confidence findings everywhere. A remit with specific channel lists, counterparty lists, tool inventories, and action boundaries produces High-confidence, actionable findings. When helping operators write a remit, push for verifiable rules.

### Success criteria

The scanner is operating well when:

1. A scan against an intentionally misconfigured test agent produces at least one Critical or High finding with specific file:line evidence and a recommended action.
2. The HTML report renders correctly in a browser opened directly from `./reports/`.
3. The findings JSON parses cleanly and is suitable for ingestion into downstream systems.
4. Every actionable remit rule appears in the Remit Coverage section with a verified status.

See `examples/` for two reference scans against deliberately vulnerable agents (FinBot from the OWASP Agentic AI CTF and HelperBot from the DVAA platform).

---

## 11. Design Lineage

The Deckard Scanner operationalizes the **RAISE Security Review Skill** — a structured framework for evaluating AI system security posture across six categories using real artifacts as inputs: code, prompts, configs, logs, policy documents. The RAISE Skill was designed as a one-time review; Deckard packages it as a droppable, re-runnable tool focused on the AI agent environment specifically.

The key design decisions in Deckard's synthesis:
- A single Worker Remit serves as the policy baseline — the scanner's primary signal is divergence between declared policy and observed implementation.
- A unified finding schema with dual RAISE + OWASP classification provides consistent, standards-aligned output.
- A canonical HTML template ensures every report looks identical regardless of which model produced it — design decisions are not delegated to the LLM.
- The package is self-contained: drop the directory, run the skill, read the report. No installer, no config file, no persistent state.

---

*The Deckard Scanner is built on the RAISE framework from* ***The Developer's Playbook for Large Language Model Security*** *by Steve Wilson — O'Reilly Media*
