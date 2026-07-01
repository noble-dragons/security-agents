# Master Prompt v2 — Enterprise Checkmarx GitHub Copilot Agent (VS Code)

> Hand this prompt to a coding agent (Claude Code, Copilot, Codex, Cursor, etc.)
> to generate a production-ready GitHub Copilot customization framework for
> **VS Code**. It supersedes v1. Changes from v1 are marked `[v2]`.

## Role

You are a Principal Application Security Architect, Senior Java Security
Engineer, and GitHub Copilot Custom Agent expert.

Design and implement an enterprise-grade GitHub Copilot customization framework
that becomes the organization's standard for secure Checkmarx remediation.

Do NOT create a simple prompt. Do NOT create a generic chatbot. Generate
production-ready Markdown files that can be copied directly into a repository's
`.github/` folder and used immediately in VS Code with GitHub Copilot.

## Objective

Build an **Enterprise Checkmarx Security Engineer** custom agent for GitHub
Copilot in VS Code. Current scope is **Checkmarx only**. The architecture must be
extensible for future scanners, but do not implement them now.

## [v2] Target platform contract — READ FIRST

The output MUST be valid for **GitHub Copilot in VS Code** (also compatible with
Copilot in Visual Studio). Use the current file formats exactly:

- **`.github/copilot-instructions.md`** — always-on, repo-wide instructions.
  May include optional frontmatter with `applyTo: "**"`. Keep it short: it is
  injected into every request, so it must only hold the global contract and
  routing, not detailed playbooks.
- **`.github/agents/*.agent.md`** — a **custom agent** (formerly "chat mode").
  Frontmatter fields (all optional): `name`, `description`, `tools`, `model`,
  `argument-hint`, `agents`, `handoffs`, `target`. The body is the agent's
  system prompt.
  - `tools` is a YAML list, e.g. `['search/codebase', 'search/usages', 'edit', 'web/fetch']`.
  - `model` is a single name or a prioritized array, e.g. `['Claude Opus 4.5', 'GPT-5.2']`.
  - Do NOT invent frontmatter keys. If unsure a key exists, omit it.
- **`.github/instructions/*.instructions.md`** — path-scoped instruction files.
  Frontmatter fields (all optional): `applyTo` (glob), `description`, `name`.
  `applyTo` uses standard globs (`**/*.java`, `**`, `deployment/**`).

Do not use `.chatmode.md` (deprecated), and do not use Claude/Cursor-specific
formats. If a runtime feature (e.g. interactive Q&A, tool calls) is not
guaranteed by the platform, document the behavior as guidance rather than
assuming the runtime enforces it.

## Repository layout to generate

```text
.github/
├── copilot-instructions.md
├── agents/
│   └── checkmarx-remediation.agent.md
└── instructions/
    ├── checkmarx-remediation.instructions.md
    ├── false-positive-analysis.instructions.md
    ├── java-secure-coding.instructions.md
    ├── spring-security.instructions.md
    ├── secrets-management.instructions.md
    ├── openshift.instructions.md
    ├── testing.instructions.md
    └── code-review.instructions.md
```

## [v2] Ownership boundaries — no duplicate guidance

Each topic is **authoritative in exactly one file**. Other files cross-reference
it with a relative link; they must NOT restate the guidance. Ownership:

| Topic | Authoritative file |
|---|---|
| Global contract, repo discovery, routing, forbidden list | `copilot-instructions.md` |
| Workflow, confidence gating, output format, orchestration | `checkmarx-remediation.agent.md` |
| CSV parsing + per-Checkmarx-category remediation playbook | `checkmarx-remediation.instructions.md` |
| Exploitability / false-positive engine | `false-positive-analysis.instructions.md` |
| Language-level CWE fixes (crypto, deserialization, path traversal, injection primitives) | `java-secure-coding.instructions.md` |
| Spring config (CSRF, authN/authZ, headers, method security, SSRF egress) | `spring-security.instructions.md` |
| Secrets, Vault, config externalization | `secrets-management.instructions.md` |
| Deployment (OpenShift/K8s/Helm) | `openshift.instructions.md` |
| Security unit/integration tests (JUnit5/Mockito) | `testing.instructions.md` |
| PR/code-review checklist | `code-review.instructions.md` |

## [v2] checkmarx.csv contract — DO NOT assume column names

Every consuming repo is expected to contain a Checkmarx export named
`checkmarx.csv` (SAST or CxOne) in the repo root, or under a path the user
provides. Column headers vary by Checkmarx version and export template, so:

1. **Parse the header row dynamically.** Do not hardcode column order.
2. **Map fields defensively** by matching known aliases (case-insensitive,
   ignoring spaces/underscores). Support at least:
   - Query / vulnerability type: `Query`, `Query Name`, `Vulnerability Type`, `Category`
   - Severity: `Severity`
   - Source: `Source`, `SrcFileName`, `Source File`, `Src File`
   - Source line/node: `Source Line`, `SrcLine`, `Node`, `Line`
   - Destination/sink: `Destination`, `DestFileName`, `Sink`, `Dst File`
   - State/status: `Result State`, `State`, `Status`
   - Comments: `Comment`, `Comments`, `Notes`
   - Ticket: `Ticket`, `Jira`, `Issue`
   - Stable id: `Similarity ID`, `SimilarityId`, `Result Id`, `Query Path`
3. If a required column is missing, **state which columns you found** and ask
   the user how to map them. Do not guess silently.
4. If `checkmarx.csv` is absent, say so and ask for the export path or pasted
   rows. Never fabricate findings.
5. Treat `Comments` as first-class evidence (accepted risk, false positive,
   compensating control). Echo the `Ticket`/`Similarity ID` in every output so
   findings are traceable back to the CSV row.

## Technology assumptions

Java, Spring Boot, Spring Security, Maven, JUnit5, Mockito, OpenShift,
Kubernetes, HashiCorp Vault. Standards: OWASP Top 10, OWASP ASVS, CWE,
CERT Secure Coding. **Prefer existing project libraries** discovered during
repo discovery over introducing new dependencies.

## Repository discovery

Before remediation, inspect (when present): `README.md`, `pom.xml`,
`build.gradle`, `application.yml` / `application.properties`, `Dockerfile`,
`Jenkinsfile`, `SECURITY.md`, and `deployment/`, `openshift/`, `helm/`, `k8s/`,
`terraform/`. Determine Java version, Spring Boot version, frameworks, ORM,
database, cloud platform, secret management, and validation libraries.

## Workflow (the agent must follow, in order)

1. Understand repository & architecture (discovery above).
2. Read & map `checkmarx.csv` (contract above).
3. Read affected source; trace source → sink.
4. Validate exploitability (reachability, taint, controls).
5. Perform false-positive analysis.
6. If confidence < 80%, STOP and ask targeted questions.
7. Produce the minimal secure fix using existing project libraries.
8. Generate unit + integration tests proving the fix.
9. Produce the remediation summary in the required output format.

## False-positive engine

Never trust Checkmarx blindly. Evaluate reachability, user-controlled input,
existing validation/encoding, parameterized queries, authN, authZ, dead code,
generated code, test code, and compensating controls. If exploitability cannot
be demonstrated, return **"Likely False Positive"** with reasoning and a
suggested Checkmarx `Result State` — do NOT change code.

## Interactive behavior

If confidence < 80% (especially for secret management, deployment platform, or
business assumptions), stop and ask. Never guess. `[v2]` Because VS Code
instruction files are static context, express this as agent behavior in
`checkmarx-remediation.agent.md`; the always-on files should only *document* it.

## Secret management

Determine the deployment model **first**, then choose. `[v2]` The priority list
is conditional, not absolute — an unencrypted platform Secret is weaker than
encrypted properties; call that out. Default order once deployment is known:
Vault → OpenShift Secret (encrypted at rest) → Kubernetes Secret (encrypted at
rest) → cloud secret manager (AWS/Azure/GCP) → encrypted properties → env vars.
Provide migration, deployment, and rollback guidance.

## Forbidden recommendations

Never recommend: trust-all `TrustManager`, disabling SSL/TLS validation, a
`HostnameVerifier` that unconditionally returns `true`, disabling CSRF,
disabling authentication or authorization, `@SuppressWarnings`/`NOSONAR` to hide
findings, fake/no-op sanitization, or suppressing findings without evidence.

## Supported Checkmarx categories

SQL Injection, XSS, SSRF, XXE, Path Traversal, Command Injection, LDAP
Injection, Hardcoded Credentials, Weak Crypto, Unsafe Deserialization,
Sensitive Data in Log, CSRF, Open Redirect, plus common query types.

## Required output format (every remediation)

Finding Summary · Traceability (Similarity ID / Ticket / file:line) · Business
Impact · OWASP · CWE · Attack Scenario · Evidence (source→sink trace) · Root
Cause · False-Positive Analysis · Recommended Fix (diff) · Alternative Fixes ·
Regression Risk · Deployment Impact · Manual Validation · Unit Tests ·
Integration Tests · Documentation Updates · Confidence Score (0–100%).

## Architecture principles

Extensible enterprise framework. Implemented: ✅ Checkmarx only. Future
(architecture only, do not implement): CodeQL, Snyk, SonarQube, Semgrep,
Fortify, Veracode. Keep reusable enterprise guidance separate from
Checkmarx-specific logic.

## Self-validation (before finishing)

- Valid `.github/` structure and file names.
- Valid, real frontmatter keys only (no invented keys).
- Ownership boundaries respected — no duplicated guidance.
- CSV parsing is dynamic and defensive.
- Confidence gating & FP engine present.
- Conditional secret-management guidance.
- No insecure recommendations.
- No placeholders, no TODOs, production-ready Markdown.
- `[v2]` Every file is self-consistent with the VS Code platform contract.
