---
name: 'Checkmarx Remediation Playbook'
description: 'How to parse checkmarx.csv and remediate each supported Checkmarx query category.'
applyTo: '**'
---

# Checkmarx Remediation Playbook

Authoritative for: **parsing `checkmarx.csv`** and the **per-category fix
strategy**. Exploitability lives in `false-positive-analysis.instructions.md`;
concrete Java code patterns live in `java-secure-coding.instructions.md` and
`spring-security.instructions.md`.

## 1. Parsing `checkmarx.csv` (dynamic, never hardcode columns)

Checkmarx SAST and CxOne exports use different headers. Read the **header row
first**, then map columns by matching aliases case-insensitively, ignoring
spaces and underscores.

| Logical field | Accepted header aliases |
|---|---|
| Query / type | `Query`, `Query Name`, `Vulnerability Type`, `Category` |
| Severity | `Severity` |
| Source file | `Source`, `SrcFileName`, `Source File`, `Src File` |
| Source line | `Source Line`, `SrcLine`, `Line`, `Node` |
| Destination / sink | `Destination`, `DestFileName`, `Sink`, `Dst File` |
| State / status | `Result State`, `State`, `Status` |
| Comments | `Comment`, `Comments`, `Notes` |
| Ticket | `Ticket`, `Jira`, `Issue` |
| Stable id | `Similarity ID`, `SimilarityId`, `Result Id`, `Query Path` |

Rules:

- If a **required** field (query, severity, source, sink) has no matching
  column, list the headers you actually found and ask the user to map them.
- If `checkmarx.csv` is **absent**, ask for the path or pasted rows. Never
  invent findings.
- Deduplicate by stable id. Prefer the highest severity when duplicates differ.
- Read `Comments` and `Result State` before triaging: an existing
  `Not Exploitable` / accepted-risk note is evidence to verify, not to ignore.
- Carry `Similarity ID` and `Ticket` into every output for traceability.

## 2. Triage order

Critical → High → Medium → Low. Within a severity, exploitable-confirmed before
needs-info. Skip anything you verify as a false positive (record the reason).

## 3. Per-category remediation strategy

For each category: the CWE, the root cause to look for, and the fix direction.
Implement using the repo's existing libraries and the code patterns in the
`java-secure-coding` / `spring-security` files.

### SQL Injection — CWE-89
- **Look for:** string-concatenated SQL/JPQL/HQL, `Statement.execute*`,
  `createQuery`/`createNativeQuery` with interpolation, dynamic `ORDER BY`.
- **Fix:** parameterized queries / `PreparedStatement`, JPA bind parameters or
  Criteria API. For dynamic identifiers (column/table/sort), validate against an
  **allow-list** — you cannot bind identifiers. See `java-secure-coding`.

### Cross-Site Scripting (XSS) — CWE-79
- **Look for:** user data written to HTML/JS without contextual encoding,
  `@ResponseBody` returning HTML, Thymeleaf `[(...)]` unescaped, `th:utext`.
- **Fix:** contextual output encoding (OWASP Java Encoder), default-escaping
  templates, correct `Content-Type`, and CSP header (see `spring-security`).

### SSRF — CWE-918
- **Look for:** HTTP client target host/URL derived from user input.
- **Fix:** allow-list of destination hosts/schemes, resolve-and-validate the IP
  (block private/link-local/metadata ranges), disable redirects to new hosts.
  Egress policy details in `spring-security`.

### XML External Entity (XXE) — CWE-611
- **Look for:** `DocumentBuilderFactory`, `SAXParserFactory`,
  `XMLInputFactory`, `TransformerFactory`, unmarshallers created without
  hardening.
- **Fix:** disable DOCTYPE/external entities/DTDs. See `java-secure-coding`.

### Path Traversal — CWE-22
- **Look for:** file paths built from user input, `new File(base, userInput)`,
  `Files.get`, zip extraction (`../`).
- **Fix:** canonicalize and confirm the resolved path stays within the intended
  base directory; reject `..`; use an allow-list of names where possible.

### Command Injection — CWE-78
- **Look for:** `Runtime.exec(String)`, `ProcessBuilder` with a shell and
  interpolated arguments.
- **Fix:** avoid the shell; pass an argument array; validate against an
  allow-list; prefer a library API over shelling out.

### LDAP Injection — CWE-90
- **Look for:** DN/filter strings built from user input.
- **Fix:** escape via `javax.naming` encoding utilities, use parameterized
  filters, validate input.

### Hardcoded Credentials — CWE-798
- **Look for:** passwords/keys/tokens literal in source or committed config.
- **Fix:** externalize to the secret backend. See
  `secrets-management.instructions.md`. Also rotate the exposed secret.

### Weak / Broken Cryptography — CWE-327 / CWE-328
- **Look for:** MD5/SHA-1 for security, DES/RC4/ECB, static IVs, `Math.random`
  for tokens, hardcoded keys.
- **Fix:** SHA-256+/bcrypt/argon2 for the right purpose, AES-GCM, `SecureRandom`,
  keys from the secret backend. See `java-secure-coding`.

### Unsafe Deserialization — CWE-502
- **Look for:** native `ObjectInputStream` on untrusted data, unsafe
  polymorphic Jackson typing, XStream/SnakeYAML with default typing.
- **Fix:** avoid native deserialization of untrusted data; use safe formats,
  type allow-lists, `SnakeYAML SafeConstructor`, disable Jackson default typing.

### Sensitive Data in Log — CWE-532
- **Look for:** logging of passwords, tokens, secrets, full PII/PAN.
- **Fix:** remove/mask the sensitive fields; structured logging with redaction.
  Never log the secret to "debug" the finding.

### CSRF — CWE-352
- **Fix:** keep Spring Security CSRF enabled; correct token handling for the
  client type. Never disable it as a "fix". See `spring-security`.

### Open Redirect — CWE-601
- **Look for:** redirect target from user input.
- **Fix:** allow-list of relative paths/known hosts; reject absolute external
  URLs; never echo raw user input into `sendRedirect`/`RedirectView`.

## 4. If the category isn't listed

Map it to the closest CWE, apply the same discipline (source→sink, exploitability,
minimal fix, tests), and state the mapping explicitly.
