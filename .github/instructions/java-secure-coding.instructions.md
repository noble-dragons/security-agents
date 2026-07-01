---
name: 'Java Secure Coding'
description: 'Language-level secure code patterns for Java: injection, crypto, deserialization, XXE, path/command handling.'
applyTo: '**/*.java'
---

# Java Secure Coding

Authoritative for: **language-level CWE fixes in Java**. Spring configuration
(CSRF, authN/authZ, security headers, SSRF egress policy) lives in
`spring-security.instructions.md`. Secrets/keys sourcing lives in
`secrets-management.instructions.md`.

Prefer libraries already on the classpath. The snippets below are patterns, not
mandated dependencies — adapt to what the repo uses.

## SQL / JPQL Injection (CWE-89)

Bind parameters; never concatenate untrusted input into a query.

```java
// BAD
stmt.executeQuery("SELECT * FROM orders WHERE id = '" + id + "'");

// GOOD (JDBC)
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT * FROM orders WHERE id = ?")) {
    ps.setLong(1, id);
    // ...
}

// GOOD (JPA)
em.createQuery("select o from Order o where o.id = :id", Order.class)
  .setParameter("id", id);
```

You cannot bind identifiers (column/table/sort direction). Validate them against
an allow-list:

```java
private static final Set<String> SORTABLE = Set.of("id", "createdAt", "total");
String col = SORTABLE.contains(input) ? input : "id";
```

## XSS output encoding (CWE-79)

Encode at the sink for the correct context. Prefer default-escaping templates
(Thymeleaf `th:text`, not `th:utext`). For manual output use OWASP Java Encoder:

```java
import org.owasp.encoder.Encode;
out.print(Encode.forHtml(userValue));      // HTML body
out.print(Encode.forHtmlAttribute(v));     // attribute
out.print(Encode.forJavaScript(v));        // inline JS
```

## XXE hardening (CWE-611)

Disable DOCTYPE and external entities on every XML factory before use:

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

For `XMLInputFactory`: `setProperty(SUPPORT_DTD, false)` and
`IS_SUPPORTING_EXTERNAL_ENTITIES, false`. For `TransformerFactory`, set
`ACCESS_EXTERNAL_DTD` and `ACCESS_EXTERNAL_STYLESHEET` to `""`.

## Path traversal (CWE-22)

Resolve against a fixed base and confirm containment:

```java
Path base = Paths.get("/srv/data").toRealPath();
Path target = base.resolve(userName).normalize();
if (!target.startsWith(base)) {
    throw new SecurityException("Path traversal blocked");
}
```

Reject `..` in zip entry names before extraction (zip-slip).

## Command injection (CWE-78)

Don't invoke a shell; pass an argument array and validate inputs:

```java
// BAD: new ProcessBuilder("sh", "-c", "convert " + file).start();
ProcessBuilder pb = new ProcessBuilder("/usr/bin/convert", file, out);
pb.redirectErrorStream(true);
```

Where possible, use a library API instead of an external process.

## LDAP injection (CWE-90)

Escape values used in DNs/filters and prefer parameterized search; do not build
filter strings by concatenation.

## Cryptography (CWE-327 / CWE-328 / CWE-330)

- **Hashing for integrity/signatures:** SHA-256 or better. Never MD5/SHA-1 for
  security.
- **Password storage:** bcrypt / argon2 / PBKDF2 (via Spring Security's
  `PasswordEncoder`), never plain hashes.
- **Symmetric encryption:** AES-256-GCM with a unique random 96-bit IV per
  message. Never ECB, never a static IV.
- **Randomness for tokens/keys/IVs:** `java.security.SecureRandom`, never
  `java.util.Random` / `Math.random()`.
- **Keys:** never hardcoded — source from the secret backend
  (`secrets-management.instructions.md`).

```java
SecureRandom rng = new SecureRandom();
byte[] iv = new byte[12]; rng.nextBytes(iv);
Cipher c = Cipher.getInstance("AES/GCM/NoPadding");
c.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
```

## Unsafe deserialization (CWE-502)

- Do not `ObjectInputStream.readObject()` on untrusted data. Prefer JSON with
  explicit types.
- Jackson: keep default typing **off**; if polymorphism is required use
  `@JsonTypeInfo` with a `@JsonSubTypes` allow-list or a validated
  `PolymorphicTypeValidator`.
- SnakeYAML: use `new Yaml(new SafeConstructor(...))`.
- XStream: configure permission allow-lists.

## Sensitive data in logs (CWE-532)

Never log secrets, tokens, passwords, full PANs, or full PII. Mask/redact and
log identifiers, not payloads. Do not add a "temporary" debug log of the
sensitive value while remediating.

## TLS / certificate validation (CWE-295)

Never install a trust-all `X509TrustManager` or a `HostnameVerifier` that
returns `true`. Use the platform default trust store; if a private CA is
needed, add it to a trust store — do not disable validation.
