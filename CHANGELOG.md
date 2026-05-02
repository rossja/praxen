<!--
  Copyright © 2026 Exabeam, Inc. All Rights Reserved.
  Confidential and Proprietary. Do not distribute. Use by permission only.
-->

# Changelog

All notable changes to Praxa will be recorded here. Format roughly follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

---

## [0.1.0] — 2026-05-01

**First internal Praxa release.**

This release is the rebranding of the project formerly known as the Exabeam Deckard Agent Security Scanner to **Praxa — agent behavior verifier**. No detection logic, severity rules, RAISE scoring, OWASP mappings, or evaluation behavior changed in this release. The change is naming, terminology, and documentation only.

### Renamed
- Project name: *Exabeam Deckard Agent Security Scanner* → **Praxa**
- Descriptor: *agent behavior verifier*
- Tagline: *Make sure your agent does its job — and only its job.*
- Skill directory: `skills/environment-scanner/` → `skills/behavior-verifier/`
- Skill identifier: `environment-scanner` → `behavior-verifier`
- Plugin name: `deckard` → `praxa` (in `.claude-plugin/plugin.json` + `marketplace.json`)
- Specification document: `DECKARD_SPEC.md` → `PRAXA_SPEC.md`
- Report output filenames: `<agent>-scan-<timestamp>.{html,txt}` → `<agent>-analysis-<timestamp>.{html,txt}` (the JSON keeps `<agent>-findings-<date>.json`)
- Distribution zip: `deckard-X.Y.Z.zip` → `praxa-X.Y.Z.zip`

### Terminology
- *scan* → *analysis* / *verification* (in user-facing labels)
- *scanner* → *verifier* (in user-facing labels)
- *Scan Summary* → *Behavior Summary* (report section title)
- *Scan Report* → *Analysis Report* (report header)
- *security issue* → *finding* (where it appeared in user-facing copy)

### Repositioned
- Headline framing: "policy compliance scanner for AI agents" → "agent behavior verifier — verifies intended vs observed behavior, ensures agents stay within defined boundaries"
- OWASP coverage list moved lower in the README; Praxa identity (name, descriptor, tagline, Worker Remit explanation, behavior-vs-intent) leads.
- Security framing preserved throughout — security references repositioned, not removed.

### Unchanged (explicitly preserved)
- All detection patterns and named detectors
- Severity rules and finding schema
- RAISE 0–5 scoring model and weighting
- OWASP LLM Top 10, OWASP Agentic Top 10, OWASP MCP Guide cross-references
- Worker Remit structure and required sections
- Report template structure (RAISE Maturity Posture, Findings Register, Remit Coverage, etc.)
- Knowledge base content (`knowledge/KB_*.md` — attribution-line wording updated; methodology untouched)
- License text (commercial / by-permission; open-source flip deferred to a later release)

### Deferred to a later release
- License conversion to an open-source license (Apache-2.0 likely target)
- "Praxa is open source / Hosted under Exabeam Labs / Project sponsor: Exabeam" positioning language in the README
- GitHub repository rename (`Exabeam/deckard` stays; will be moved separately by an admin)

---

## Pre-Praxa history (legacy: Deckard releases)

For reference. Detail kept short — these were internal releases under the prior name.

### [0.7.0] — 2026-04-24 *(Deckard)*
- Restructured as a Claude Code plugin marketplace (`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`)
- Skill moved to `skills/environment-scanner/SKILL.md` with proper frontmatter

### [0.6.6] — 2026-04-24 *(Deckard)*
- Step 4b — Secondary Prompt Discovery for session-loaded files (`SOUL.md`, `AGENTS.md`, `MEMORY.md`, etc.)
- KB additions: Session Bootstrap Files / Agent Memory Files; writable-session-loaded-file compound chain

### [0.6.5] — 2026-04-23 *(Deckard)*
- Three input shapes documented (source / deployment / behavioral); behavior-only scanning validated against the Zoidberg target
- `RAISE Maturity Posture` moved to end of report; rubric table baked in; neutralized blue palette
- Scan Summary section added (renamed Behavior Summary in 0.1.0)
- Empty-file signal detector + Declared-but-Never-Consulted-Config detector
- "Never Reprint Secrets" rule

### [0.6.4] and earlier *(Deckard)*
- Initial scanner-only MVP, separating from the deferred runtime-watcher concept
- Canonical HTML report template; standardized product naming
- Examples directory (FinBot, HelperBot)
