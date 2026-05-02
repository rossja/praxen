<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Praxa Pre-Release Test Plan

Praxa's regression test suite. Before every release, analyze each target in this document and review the resulting reports against the baseline expectations below. Reports themselves are **not** kept in this directory — they regenerate on each run and change between analyses.

## Directory contents

- `README.md` — this file
- `remits/` — the Worker Remits developed for each test agent. These are reusable and do not change between analyses.

## Pre-release checklist

1. Build the candidate release zip: `./build.sh` from the repo root.
2. For each target below, either:
   - Scan the already-built zip against the target workspace (confirms the distributed zip works), **or**
   - Scan from the repo's `skills/` + `knowledge/` directly (confirms skill edits land correctly).
3. Review each report against the baseline expectations in this document.
4. Any regression — a material finding dropped, a critical theme missed, a weighted score that moves without code changes on Praxa — blocks the release.
5. New findings or severity shifts that reflect calibration improvements are fine. Note them in the release notes.

## How to run an analysis

For each target:

1. Clone or re-extract the target repository (URLs below).
2. Stage the workspace scope — the paths inside the target repo that constitute the agent code (notes below for each target).
3. Create an analysis working directory, e.g., `/tmp/<target>_scan/reports/`.
4. Copy the corresponding remit from `remits/` into the analysis working directory as `WORKER_REMIT.md`.
5. Open a Claude Code session with the working directory as CWD.
6. Instruct Claude Code to read `skills/behavior-verifier/SKILL.md` and analyze the workspace path.
7. Review `<target>-analysis-<timestamp>.html` in `reports/`.

## Test targets

Ordered from simplest (intentionally-vulnerable CTF) to most complex (active production agent). Run them in order for a release; the earlier analyses catch skill-execution issues fast, the later analyses exercise subtle detection.

---

### 1. FinBot — OWASP Agentic AI CTF

**Remit:** `remits/finbot.md`
**Source:** https://github.com/OWASP-ASI/finbot-ctf-demo
**Scope:** full repo root (the agent code is small — Flask + SQLAlchemy app)
**Notes:** Deliberately vulnerable CTF agent. Autonomous invoice processor. The Praxa should catch runtime-mutable goal overrides, unauthenticated admin endpoints, fraud-detection toggles, business-context bypass of manual review thresholds, invoice-description injection into LLM context. This is the canonical "hobby-grade insecure agent" test — if Praxa fails to produce 4+ Critical findings here, something is broken.
**Baseline expectation:** 5-10 Critical / 5-10 High, weighted ≈ 0.6-0.9 / 5.0.

### 2. HelperBot — DVAA training agent

**Remit:** `remits/helperbot.md`
**Source:** https://github.com/opena2a-org/damn-vulnerable-ai-agent (HelperBot persona in `src/core/agents.js`)
**Scope:** a minimal workspace containing `agents.js`, `vulnerabilities.js`, `index.js`, and the LLM client files. The HelperBot definition is in `agents.js` lines ~43-78.
**Notes:** Intentionally vulnerable training agent from the DVAA platform. Smaller and simpler than FinBot — good quick smoke test. Regression test of choice: fast turnaround, clear baseline, exercises common findings (input validation, system-prompt API key embed, write_file without path guard, context manipulation, no audit logging, no rate limit).
**Baseline expectation:** 3-4 Critical / 6-11 High / 1-2 Medium, weighted ≈ 0.45 / 5.0 (Absent). Stable across Praxa versions.

### 3. LangChain SQL Agent

**Remit:** `remits/langchain-sql.md`
**Source:** https://github.com/langchain-ai/langchain-community (the classic `create_sql_agent` is in `libs/community/langchain_community/agent_toolkits/sql/` and `libs/community/langchain_community/tools/sql_database/`)
**Scope:** the `agent_toolkits/sql/` + `tools/sql_database/` trees + `utilities/sql_database.py`.
**Notes:** Mature library with explicit maintainer security warnings in the docstring. Scanner correctly identifies DML-prohibition-is-prompt-only pattern. Not a disclosure target (maintainer has already warned). Kept as a "skill validates on a mature codebase" test. A clean 5-finding set with reasonable severity distribution confirms the Policy-Implementation Divergence detection is working.
**Baseline expectation:** 5 Critical / 5 High / 4 Medium, weighted ≈ 0.60 / 5.0.

### 4. OpenAI Agents SDK — Customer Service Example

**Remit:** `remits/openai-customer-service.md`
**Source:** https://github.com/openai/openai-agents-python (`examples/customer_service/main.py` + the `agents` SDK snapshot in `src/agents/`)
**Scope:** the customer_service example + enough of the SDK to reason about handoffs, guardrails, and tool approval.
**Notes:** Demonstrates the "framework ships guardrails; example uses none" pattern. Scanner should find that the SDK has `InputGuardrail`, `OutputGuardrail`, `needs_approval`, `is_enabled`, `input_filter` — and that the example uses zero of them. Also flags the `on_seat_booking_handoff` generating a random flight number (synthetic identifier) which violates the remit.
**Baseline expectation:** 3-4 Critical / 6-7 High / 3 Medium, weighted ≈ 0.90 / 5.0.

### 5. AutoGen Code Executor

**Remit:** `remits/autogen-code-executor.md`
**Source:** https://github.com/microsoft/autogen (`python/packages/autogen-ext/src/autogen_ext/code_executors/` + `python/packages/autogen-core/src/autogen_core/code_executor/`)
**Scope:** the 5 executor implementations (local, docker, docker_jupyter, jupyter, azure) + the core abstraction.
**Notes:** "Defaults undermine sandbox" pattern. Scanner should find: LocalCommandLineCodeExecutor uses `warnings.warn` instead of an approval gate; silent Docker→Local fallback swallows exceptions; Docker containers default to no `user=`/`read_only=`/`mem_limit=`/`cap_drop=`; `os.environ.copy()` leaks parent env; Jupyter timeouts are soft.
**Baseline expectation:** 2 Critical / 7 High / 6 Medium, weighted ≈ 1.50 / 5.0.

### 6. Sweep — GitHub issue-to-code agent

**Remit:** `remits/sweep.md`
**Source:** https://github.com/sweepai/sweep (`sweepai/` subtree: agents, core, web, config)
**Scope:** `sweepai/agents/`, `sweepai/core/`, `sweepai/web/`, `sweepai/config/`, plus `sweep.yaml`, `Dockerfile`, `docker-compose.yml`, `pyproject.toml`.
**Notes:** Exercises the **declared-but-never-consulted-config** detector (`WEBHOOK_SECRET` defined but never used). Three `subprocess.run(shell=True)` sites with LLM-derived arguments. Hardcoded PostHog analytics key. This is a good Step 6 detector test. Also represents the "disclosure-worthy in theory, dormant maintainer in practice" class.
**Baseline expectation:** 7 Critical / 7 High / 5 Medium, weighted ≈ 0.90 / 5.0.

### 7. Devika — autonomous software engineer

**Remit:** `remits/devika.md`
**Source:** https://github.com/stitionai/devika
**Scope:** `devika.py` + `src/` (agents, llm, memory, apis) + `sample.config.toml`, `devika.dockerfile`, `requirements.txt`, `ARCHITECTURE.md`.
**Notes:** Exercises the **empty-file signal** detector — `src/sandbox/firejail.py` and `src/sandbox/code_runner.py` are 0-line stubs. Runner calls `subprocess.run` directly. Unauthenticated `/api/settings` POST on `0.0.0.0:1337`. Path traversal in `save_code_to_project`. Compound RCE chain (web → researcher → formatter → coder/runner → subprocess). Best validation that Step 4 empty-file heuristic is working.
**Baseline expectation:** 5 Critical / 6-7 High / 4 Medium, weighted ≈ 0.60 / 5.0.

### 8. Aider — interactive pair programming agent

**Remit:** `remits/aider.md`
**Source:** https://github.com/Aider-AI/aider
**Scope:** `aider/*.py` (top-level) + `aider/coders/`.
**Notes:** Mature, production-quality agent with a developer-in-the-loop safety model. The findings are subtle — "Trust this message" directive in prompt, `# ai!` comment auto-execution in `--watch-files`, `/add` accepts absolute paths, declared-but-never-consulted sensitive-file list. Tests that Praxa produces actionable findings even on well-engineered agents, and doesn't get fooled by confirmation-prompt theater. Also a Jinja2 evidence-block test — catches if the Step 11 verification grep regex broke.
**Baseline expectation:** 3 Critical / 6 High / 4 Medium, weighted ≈ 1.50 / 5.0.

### 9. OpenHands — autonomous software engineering platform

**Remit:** `remits/openhands.md`
**Source:** https://github.com/All-Hands-AI/OpenHands
**Scope:** `openhands/` core — `core/`, `controller/`, `runtime/`, `events/`, `server/`, `llm/`, `mcp/`, `integrations/` — plus `config.template.toml`, `docker-compose.yml`. Exclude `enterprise/`, `frontend/`, `kind/`.
**Notes:** Best-architected agent in the test set. Sandboxed Docker runtime, per-integration OAuth scoping, structured event log, secret redaction primitives. Scanner should still find: CLIRuntime / LocalRuntime ship alongside the sandboxed runtime; `confirmation_mode` defaults False; `SESSION_API_KEY` auth only when env var set; `.openhands/setup.sh` auto-sourced from connected repo (supply-chain shaped); Docker socket mounted in default docker-compose; declared-but-never-consumed `save_trajectory_path`/`replay_trajectory_path`. Highest weighted score in the suite (~2.45) but still rated CRITICAL due to the default-gap chain. Best demonstration that the RAISE maturity model functions honestly on a mature target.
**Baseline expectation:** 2 Critical / 6 High / 5 Medium / 2 Low, weighted ≈ 2.45 / 5.0 (Partial).

---

## What a release review looks like

For each target, open the HTML report and check:

**Structural correctness**
- All three output files landed (`.html`, `.json`, `.txt`)
- Report renders without errors in a browser (static HTML, no external fetches)
- No unfilled `{{PLACEHOLDER}}` in the rendered output
- Footer counts match the Findings Register
- Posture summary entry in the JSON has `scan_summary` populated

**Finding quality**
- The Behavior Summary narrative reads as diagnostic, not templated
- Every Critical / High finding has specific file:line evidence
- Recommended actions name the file and change, not generic advice
- OWASP tags include the full category name (`LLM01 — Prompt Injection`, not `LLM01`)
- Policy references quote the exact remit text

**RAISE Maturity Posture section (end of report)**
- Weighted score reasonable relative to the baseline above
- Maturity label matches the score (Absent / Ad hoc / Partial / Established / Strong / Exemplary)
- Rubric table present and unmodified
- No traffic-light coloring on category cards (uniform blue styling)

**Secrets discipline**
- No literal API keys, tokens, or passwords in the HTML or JSON — any credential is referenced by `[REDACTED — pattern at file:line]`

**If any check fails**, investigate before releasing. Finding-count shifts within baseline bands are expected; theme-level coverage regressions are not.

## Notes on the test set composition

The nine targets deliberately span a spectrum:

- **Intentionally vulnerable** (FinBot, HelperBot, Devika) — calibration anchors. Findings here should be dense and unambiguous.
- **Mature library, pre-warned maintainer** (LangChain SQL) — confirms the skill handles well-documented targets without producing value-free findings.
- **Framework + example pattern** (OpenAI CS) — exercises the "guardrails shipped, not used" detection.
- **Framework defaults pattern** (AutoGen, OpenHands) — exercises the "sandbox exists but defaults bypass it" detection.
- **Production agent, solo-maintainer territory** (Sweep, Aider) — exercises subtle and novel finding detection; the targets most likely to produce disclosure-worthy output.
- **Production agent, well-funded team** (OpenHands) — the ceiling of what well-engineered agents look like today. Establishes realistic maturity-scale interpretation.

A release that produces solid reports on all nine has been validated across the full range of agent postures we've encountered.
