---
name: 'Spring Security Configuration'
description: 'Secure Spring Boot/Spring Security configuration: CSRF, authN/authZ, security headers, SSRF egress, redirects.'
applyTo: '**/*.java'
---

# Spring Security Configuration

Authoritative for: **Spring-level security configuration**. Language-level code
fixes live in `java-secure-coding.instructions.md`. Secrets live in
`secrets-management.instructions.md`.

Match the repo's existing style (component-based `SecurityFilterChain` vs. legacy
`WebSecurityConfigurerAdapter`). Examples use the modern `SecurityFilterChain`.

## CSRF (CWE-352)

Keep CSRF protection **on** for browser/session-based apps. Never disable it as
a "fix" for a Checkmarx CSRF finding.

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

Stateless token-authenticated APIs (no cookies) may disable CSRF **only** if
they truly carry no ambient credentials — document that reasoning; don't disable
reflexively.

## Authentication & Authorization

- Deny by default; explicitly permit public endpoints.
- Enforce authorization at the method layer for service logic:
  `@EnableMethodSecurity` + `@PreAuthorize("hasRole('...')")`.
- Never disable authentication or authorization to clear a finding.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/actuator/health", "/login").permitAll()
    .anyRequest().authenticated());
```

## Security headers (supports XSS/clickjacking defense)

```java
http.headers(h -> h
    .contentSecurityPolicy(csp -> csp.policyDirectives(
        "default-src 'self'; object-src 'none'; frame-ancestors 'none'"))
    .frameOptions(fo -> fo.deny())
    .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true)));
```

Set an explicit CSP; do not rely on inline scripts. Add
`X-Content-Type-Options: nosniff` (on by default in Spring Security).

## SSRF egress policy (CWE-918)

When an outbound call target comes from user input:

- Restrict scheme to `https` (and `http` only if required).
- Allow-list destination hosts; reject everything else.
- Resolve the hostname and **block private/link-local/loopback/metadata**
  ranges (`10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`,
  `::1`, `fc00::/7`) to defeat DNS-rebinding to internal targets.
- Disable automatic redirect-following to new hosts.

```java
InetAddress addr = InetAddress.getByName(uri.getHost());
if (addr.isLoopbackAddress() || addr.isSiteLocalAddress()
        || addr.isLinkLocalAddress() || addr.isAnyLocalAddress()) {
    throw new SecurityException("Blocked SSRF target: " + uri.getHost());
}
```

## Open redirect (CWE-601)

Never pass user input straight into `sendRedirect` / `RedirectView`. Allow-list
relative paths or known hosts:

```java
if (!target.startsWith("/") || target.startsWith("//")) {
    return "redirect:/home"; // reject absolute/external
}
```

## Password storage

Use a `PasswordEncoder` bean (`BCryptPasswordEncoder` or a
`DelegatingPasswordEncoder`). Never store or compare plaintext, never MD5/SHA-1.

## Actuator & error handling

- Lock down Actuator: expose only what's needed; require auth for sensitive
  endpoints.
- Don't leak stack traces to clients; use a controlled error response.
