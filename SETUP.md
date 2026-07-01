# Checkmarx Remediation Agent — Setup Guide

How to use the **Checkmarx Remediation** GitHub Copilot agent in your repository
to triage and fix Checkmarx findings in Java / Spring projects.

The agent definitions live in the central security-agents repository. This guide
covers installing them into your own repo and running them in VS Code.

> Replace `<AGENTS_REPO_URL>` in the commands below with your central
> agents repository URL.

---

## 1. Prerequisites

- **VS Code** (latest).
- **GitHub Copilot** and **GitHub Copilot Chat** extensions, signed in with a
  plan that includes agents.
- Custom agents enabled (recent Copilot Chat auto-detects `.github/agents/`).
  If your org disables custom agents by policy, ask your Copilot admin to allow
  them.
- A **Java / Spring** repository to remediate.
- A Checkmarx export named **`checkmarx.csv`** for that repo (see step 3).

---

## 2. Install the agent into your repository

You have two options. Most teams use **Option A**.

### Option A — Per-repository (recommended)

Copy the `.github/` folder from the central repo into the root of your project:

```bash
# from the root of YOUR repo
git clone --depth 1 <AGENTS_REPO_URL> /tmp/security-agents
cp -r /tmp/security-agents/.github ./.github        # merges the agents/instructions in
git add .github && git commit -m "Add Checkmarx remediation Copilot agent"
```

> If your repo already has a `.github/` folder, only copy the `agents/` and
> `instructions/` subfolders and `copilot-instructions.md` — don't overwrite
> existing workflows.

Your project should end up like this:

```text
your-service/
├── .github/
│   ├── copilot-instructions.md
│   ├── agents/
│   │   └── checkmarx-remediation.agent.md
│   └── instructions/
│       ├── checkmarx-remediation.instructions.md
│       ├── false-positive-analysis.instructions.md
│       ├── java-secure-coding.instructions.md
│       ├── spring-security.instructions.md
│       ├── secrets-management.instructions.md
│       ├── openshift.instructions.md
│       ├── testing.instructions.md
│       └── code-review.instructions.md
├── checkmarx.csv
└── src/...
```

### Option B — User-level (available in every repo, nothing committed)

Copy the files once into your user profile so the agent shows up in all
workspaces without adding it to each repo:

```text
Windows: %USERPROFILE%\.copilot\agents\        and   %USERPROFILE%\.copilot\instructions\
macOS/Linux: ~/.copilot/agents/                and   ~/.copilot/instructions/
```

Copy `agents/*.agent.md` into the `agents` folder and `instructions/*.instructions.md`
into the `instructions` folder. (Note: user-level installs don't carry the
always-on `copilot-instructions.md`; per-repo is better if you want the global
contract applied.)

---

## 3. Add the Checkmarx export

Place your Checkmarx result export in the repo root as **`checkmarx.csv`**
(or note its path — you can tell the agent where it is).

- Works with **SAST** and **CxOne** CSV exports; the agent maps columns
  dynamically, so header naming differences are handled.
- If the file is missing, the agent will ask for it rather than invent findings.
- `checkmarx.csv` often contains sensitive path/finding data — consider adding it
  to `.gitignore` if you don't want it committed:
  ```
  echo "checkmarx.csv" >> .gitignore
  ```

---

## 4. Run the agent

1. Open your repo in VS Code.
2. Open **Copilot Chat** (`Ctrl+Alt+I` / `Cmd+Alt+I`).
3. In the chat input's **agent picker dropdown**, select **Checkmarx Remediation**.
   - Alternatively type **`/agent`** and pick it for a single turn.
4. Type a request (examples below).
5. Review each proposed edit as a diff → **Keep** or **Discard** per file.

### Example prompts

```
Show me the triage table for checkmarx.csv. Don't fix anything yet.
```
```
Remediate all the High and Critical exploitable findings. Add tests and give the full report.
```
```
Remediate the SQL Injection in OrderRepository.
```
```
Is finding <Similarity ID> a false positive? Show your reasoning.
```

### "Fix everything" prompt

```
Triage every finding in checkmarx.csv, then remediate all exploitable ones in
severity order. For each: apply the minimal secure fix with tests and give the
full remediation report. For anything under 80% confidence, stop and ask me
before changing code. Skip and list the false positives with justification.
```

> Tip: work in batches (Critical/High first, review, commit; then Medium/Low).
> Big "fix all in one go" runs are harder to review.

---

## 5. What the agent does

For each finding it follows a fixed workflow:

1. Discovers your stack (build files, config, deployment, libraries).
2. Parses and maps `checkmarx.csv`.
3. Traces the source → sink path in your code.
4. Assesses **exploitability** — it does not trust Checkmarx blindly.
5. **Stops and asks** if its confidence is below 80% (e.g. secret backend,
   deployment target, business rules).
6. Applies a **minimal secure fix** using libraries already in your project.
7. Generates unit + integration tests that prove the fix.
8. Produces a structured remediation report (impact, CWE/OWASP, diff,
   regression risk, validation steps, confidence score).

It **never** recommends insecure shortcuts (disabling CSRF/TLS/auth, trust-all
certificates, `NOSONAR`, fake sanitizers, hardcoded secrets).

If a finding isn't genuinely exploitable, it returns a **"Likely False
Positive"** verdict with evidence and a paste-ready Checkmarx justification —
instead of changing code.

---

## 6. Reviewing and committing fixes

- Review every diff before keeping it; the agent writes tests, but you are the
  final gate.
- Run the tests it generates (`mvn test` / `./gradlew test`).
- Commit per severity tier so a bad fix is easy to isolate:
  ```bash
  git add -A && git commit -m "Fix Critical/High Checkmarx findings"
  ```
- Re-scan with Checkmarx to confirm the findings are resolved.

---

## 7. Troubleshooting

| Symptom | Fix |
|---|---|
| Agent not in the dropdown | Confirm `.github/agents/checkmarx-remediation.agent.md` exists in the open workspace; reload VS Code; check Copilot custom agents aren't disabled by org policy. |
| Agent "does nothing" | It needs `checkmarx.csv`. Add it or tell the agent the path. |
| Model greyed out | The `model:` names must match your Copilot model list. It falls back to the picker's model automatically — harmless. |
| Instructions not applying | The `instructions/*.md` auto-apply via `applyTo` globs (e.g. `**/*.java`); open a matching file. Check the "References" in the chat response. |
| `@checkmarx-remediation` doesn't work | Custom agents can't be `@`-mentioned. Use the dropdown or `/agent`. |

---

## 8. Staying up to date

The agent evolves in the central security-agents repository. To pull the latest
into your repo:

```bash
git clone --depth 1 <AGENTS_REPO_URL> /tmp/security-agents
cp -r /tmp/security-agents/.github ./.github
git add .github && git commit -m "Update Checkmarx Copilot agent to latest"
```

Questions or improvements → open an issue/PR against the central security-agents
repository.
