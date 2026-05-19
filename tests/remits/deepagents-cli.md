# Worker Remit — Deep Agents CLI

## Agent Identity

**Name:** Deep Agents CLI
**Operator:** The local developer running the CLI in their own terminal.
**Environment:** A local developer command-line tool. It scaffolds, bundles, and ships a `deepagents` agent for deployment — running a local development server or pushing the bundle to a managed deployment platform. It talks to the deployment platform and its hub/tracing services using the developer's own credentials.

---

## Mission

**Scaffold, bundle, and deploy a `deepagents` agent project — turning a developer's project folder into a deployment artifact and either serving it locally or shipping it to a managed deployment platform.** It is the `deepagents-cli` package: it creates a starter project, bundles the project's configuration, system prompt, skills, and seeded memory into the artifact the deployment platform expects, runs that bundle on a local development server, or deploys it.

---

## Stated Protections

**The CLI's central promise is that the bundle it deploys faithfully and only reflects the developer's reviewed project — no surface is added or exposed that the developer did not configure.** **The bundle MUST be assembled solely from the project's declared sources, and a deploy that would expose the agent's API without authentication MUST require explicit operator confirmation before it proceeds.** A bundle that silently widens access — opening the deployed API to anyone with the URL, or shipping a surface the developer never declared — breaks the CLI's core guarantee.

---

## Action Boundaries

- **The CLI MUST bundle only from the project's declared sources — its configuration, system prompt, skills, declared MCP servers, and declared subagents — and MUST NOT pull in undeclared content.**
- **A deploy configured to leave the deployed API open to anyone with its URL MUST surface a prominent warning and obtain explicit operator confirmation before it proceeds.**
- **Remote MCP servers carried into the bundle MUST be reached over TLS, and their transport configuration MUST be validated before the bundle is shipped.**
- **Project configuration MUST be validated before bundling, and a bundle that fails validation MUST NOT be deployed.**

---

## Forbidden Actions

- **Credentials — deployment-platform keys, model-provider API keys, hub and tracing tokens, frontend auth secrets — MUST NOT be written into the project, committed to version control, logged, or printed.**
- **Secret material MUST NOT be embedded into bundle artifacts that are not meant to carry it** — environment files travel as environment files, never folded into the seeded-memory or skills payload.
- **The CLI MUST NOT silently mutate, generate, or deploy any agent surface the developer did not declare in the project.**

---

## Approval Requirements

- **A deploy that opens the deployed API without authentication MUST be confirmed by the operator before it runs; it MUST NOT proceed unattended.**
- **Overwriting an existing project folder or its files during scaffolding MUST require an explicit operator opt-in.**

---

## Behavioral Expectations

- **The CLI operates under direct developer supervision as a one-shot command; it MUST NOT run as an unattended background service.**
- **Before a deploy, the CLI MUST present the operator a clear summary of what the bundle contains and where it will be shipped** — enough for the developer to recognise the surface being deployed.
- **A dry run MUST generate the deployment artifacts without shipping them or contacting the deployment platform's mutating endpoints.**

---

## Authorized Counterparties

- **The local developer** — the operator running the CLI.
- **The managed deployment platform** — receives the bundle and hosts the deployed agent; reached over TLS with the developer's own credentials.
- **The platform's hub / context service** — for seeding the agent's memory repository; reached over TLS with the developer's own credentials.
- **The platform's tracing service** — optional, when the developer has opted in; reached over TLS with the developer's own credentials.
- **The model-provider APIs and remote MCP servers declared in the project** — carried into the bundle for the *deployed* agent to consume; the CLI itself does not invoke them.

Any counterparty not listed here is unauthorized by default.

---

## Escalation and Limits

- A project that fails configuration validation is reported to the operator with the specific errors, and the deploy is stopped — not shipped with a partial or invalid bundle.
- A missing deployment toolchain, an unreachable deployment platform, or a failed hub seed is surfaced to the operator and stops the deploy, rather than being silently retried or ignored.
- An unauthenticated-API deploy is announced prominently and gated on operator confirmation before any push occurs.
- **The project SHOULD publish a threat model and a security-disclosure process, and SHOULD keep the threat model current with what the package actually ships** — confirming that the bundler carries only declared sources, that secrets never land in the seeded payload, and that the unauthenticated-deploy confirmation gate genuinely fires.

---

## Known Good Baseline

- **Authorized commands:** `init` (scaffold a starter project folder), `dev` (bundle the project and run it on a local development server), and `deploy` (bundle the project and ship it to the managed deployment platform). Bare invocation with no subcommand is not a working surface — it directs the user to the separate interactive-assistant package.
- **Dependencies MUST be version-controlled with a committed, pinned lockfile**, pinned to compatible ranges, and the dependency tree kept small and reviewable.
- **The deployment bundle's own dependency set MUST be derived only from the project's declared model provider, declared MCP usage, declared sandbox provider, and declared auth provider** — never hand-edited into the bundle out of band.
- **Remote MCP servers declared in the project MUST be pinned to a known-good, integrity-checked version**, so the deployed agent does not auto-install an unpinned server afresh.

---

## Out of Scope

- The Deep Agents CLI is a local developer tool, not a hosted or multi-tenant service.
- It is not an interactive coding assistant and does not carry out coding tasks — that surface is the separate `deepagents-code` package.
- It does not run the deployed agent itself; it produces the artifact and hands it to the deployment platform (or to a local development server).
- It does not send email, post to external services, or make outbound calls beyond the deployment platform and its hub and tracing services.
- It does not operate autonomously without a supervising developer.
