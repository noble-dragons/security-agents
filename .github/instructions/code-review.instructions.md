---
name: 'Security Code Review'
description: 'Checklist for reviewing a Checkmarx remediation PR before merge.'
applyTo: '**'
---

# Security Code Review

Authoritative for: **reviewing a remediation before merge**. This is the final
gate. Detailed fix patterns live in the language/framework/secrets/deployment
instruction files — reference them; don't restate them.

## What a good remediation PR must have

1. **Traceability** — links the `Similarity ID` / `Ticket` to the change.
2. **Right root cause** — fixes the source→sink path, not just the symptom
   Checkmarx pointed at.
3. **Minimal & idiomatic** — least change needed; uses libraries already in the
   project; matches existing style.
4. **Tests present and meaningful** — exploit-blocked + legitimate-path tests
   that actually fail on the old code (see `testing.instructions.md`).
5. **No forbidden patterns** — see below.

## Reject if the PR contains any of these

- Trust-all `TrustManager`, disabled TLS/hostname validation, or a
  `HostnameVerifier` returning `true`.
- Disabled CSRF, authentication, or authorization used as the "fix".
- `@SuppressWarnings`, `// NOSONAR`, or a no-op sanitizer to silence the scanner.
- New hardcoded secret/key/token, or a secret added to logs, config, or the
  image.
- A "fix" with no test, or a test whose assertions were weakened to pass.
- Broadened scope: unrelated refactors bundled with the security fix.

## Category quick-checks

- **SQLi:** query is bound, not concatenated; dynamic identifiers are
  allow-listed.
- **XSS:** contextual encoding at the sink / default-escaping template; CSP set.
- **SSRF:** scheme + host allow-list; private/metadata ranges blocked; redirects
  constrained.
- **XXE:** DOCTYPE/external entities disabled on the specific factory used.
- **Path traversal:** canonicalize + `startsWith(base)` containment check.
- **Command injection:** no shell; argument array; validated inputs.
- **Crypto:** SHA-256+/bcrypt as appropriate; AES-GCM; `SecureRandom`; keys from
  the secret backend.
- **Deserialization:** no native `ObjectInputStream` on untrusted data; type
  allow-list.
- **Secrets:** externalized, rotated, and (if committed) history purged.

## False-positive PRs

If the change closes a finding as **Not Exploitable** rather than editing code,
require the evidence from `false-positive-analysis.instructions.md`: the specific
eliminations with file:line references and the paste-ready Checkmarx
justification. Do not accept a bare "false positive" with no reasoning.

## Reviewer output

Approve, or request changes with specific, actionable comments tied to the file
and line and the relevant instruction file. Confirm the confidence score is
justified before approving.
