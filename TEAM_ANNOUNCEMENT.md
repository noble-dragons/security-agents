# Team Announcement — Checkmarx Remediation Copilot Agent

> Draft email to share with the team. Copy the body below into your mail client.
> (This file is temporary — delete it from the repo once you've sent the email.)

---

**Subject:** New: Checkmarx Remediation Copilot Agent — try it in your repos

Hi team,

We now have a **Checkmarx Remediation** custom agent for GitHub Copilot (VS Code)
to help triage and fix Checkmarx findings faster.

**What it does**
- Reads your repo's `checkmarx.csv`, traces each finding source→sink, and checks
  whether it's *actually* exploitable (filters false positives — never trusts the
  scanner blindly).
- Proposes a **minimal secure fix + tests**, with a full report (CWE/OWASP,
  business impact, diff, confidence score).
- Stops and asks when confidence is <80% (secrets, deployment, business rules).
  Never suggests insecure shortcuts (disabling CSRF/TLS/auth, `NOSONAR`, etc.).

**How to use (2 min)**
1. Copy the `.github/` folder from the central agents repo into your project.
2. Drop your `checkmarx.csv` in the repo root.
3. In Copilot Chat, select **Checkmarx Remediation** from the agent dropdown and
   ask, e.g. *"Show the triage table"* then *"Remediate the High/Critical findings."*

Full instructions: see **SETUP.md** in the central repo.

**Scope**
- ✅ **Now:** Checkmarx findings for **Java / Spring Boot** projects.
- 🔜 **Next (in progress):** **Angular / JavaScript** and **Python** support —
  same agent workflow and false-positive discipline, language-specific fixes.
- 🔭 **Later:** additional scanners (CodeQL, Snyk, SonarQube, Semgrep, Fortify,
  Veracode) as separate agents sharing the same secure-coding knowledge base.

Feedback and PRs welcome.

Thanks,
Alex
