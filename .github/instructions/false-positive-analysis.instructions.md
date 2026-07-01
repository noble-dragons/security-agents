---
name: 'False-Positive Analysis Engine'
description: 'How to decide whether a Checkmarx finding is genuinely exploitable before touching code.'
applyTo: '**'
---

# False-Positive Analysis Engine

Authoritative for: **deciding whether a finding is exploitable**. Run this
before proposing any fix. Never trust Checkmarx blindly, and never mark
something a false positive without evidence.

## Core question

> Can a realistic attacker drive untrusted input from the reported **source**
> to the reported **sink** in a way that causes the vulnerability, given the
> code and controls that actually exist in this repo?

If yes → remediate. If demonstrably no → **Likely False Positive**. If you can't
tell → **Needs Info** (ask, don't guess).

## Evaluation checklist

Walk the real trace and answer each. Cite the file:line evidence.

1. **Reachability** — Is the sink reachable from an entry point (controller,
   listener, scheduled job, message consumer)? Dead/unreferenced code → likely FP.
2. **User-controlled input** — Is the source actually attacker-influenced, or is
   it a constant, enum, server-generated value, or trusted config?
3. **Existing validation** — Is the input constrained upstream (Bean Validation,
   allow-list, type conversion, regex) enough to neutralize the payload?
4. **Existing encoding/escaping** — For XSS/injection, does the framework or
   code already encode at the sink (e.g. default-escaping template, ORM binding)?
5. **Parameterized queries** — For SQLi, is the query actually bound, not
   concatenated? Checkmarx often flags safe `PreparedStatement` usage.
6. **Authentication / Authorization** — Does reaching the sink require auth/roles
   that meaningfully reduce or eliminate exposure? (Note: authn is mitigation,
   not always elimination — insiders/authz gaps still count.)
7. **Dead code / feature flags** — Is the path disabled or unreachable in prod?
8. **Generated code** — Is the flagged file generated (e.g. from a schema)?
   Fix the generator or template, not the output.
9. **Test code** — Is it under `src/test`? Usually not shipped; note it, don't
   "fix" production-shaped test fixtures unless they leak.
10. **Compensating controls** — WAF, network egress policy, DB permissions,
    read-only replicas, output filters. Treat as risk-reducing, but prefer a
    code-level fix; document any control you rely on.

## Weighing the evidence

- Any **one** solid, verifiable elimination (unreachable, non-user input, proper
  binding/encoding at the sink) can justify a false-positive verdict.
- Mitigations that merely *reduce* risk (authn, WAF) do **not** by themselves
  make it a false positive — they lower priority, not exploitability.
- Prefer a code fix over relying on an external control you can't guarantee.

## Confidence scoring

Assign 0–100%:

- **≥ 80%** — Act (remediate or record FP) and state the evidence.
- **< 80%** — STOP. Ask targeted questions (secret backend, deployment target,
  business rules, unseen code). List exactly what would raise your confidence.

## False-positive output

When you conclude Likely False Positive, return:

- **Verdict:** Likely False Positive
- **Traceability:** `Similarity ID`, `Ticket`
- **Why not exploitable:** the specific eliminations with file:line evidence.
- **Residual risk / caveats:** what would change the verdict.
- **Suggested Checkmarx action:** e.g. set `Result State` to `Not Exploitable`
  with this justification (paste-ready).
- **Confidence Score.**

Do **not** modify code for a false positive. Do not use `NOSONAR`,
`@SuppressWarnings`, or fake sanitizers to silence the scanner.
