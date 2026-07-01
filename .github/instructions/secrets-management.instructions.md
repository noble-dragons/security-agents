---
name: 'Secrets Management'
description: 'Externalizing secrets and keys: choosing a backend by deployment model, Vault usage, migration and rollback.'
applyTo: '**'
---

# Secrets Management

Authoritative for: **removing hardcoded secrets and sourcing keys/credentials**
(CWE-798, CWE-259, CWE-321). Crypto primitives live in
`java-secure-coding.instructions.md`; platform wiring in
`openshift.instructions.md`.

## Determine the deployment model FIRST

Do not recommend a backend until you know where the app runs. If it's unclear,
STOP and ask (this is one of the mandatory confidence-gate questions). The
priority list below is **conditional**, not absolute.

### Selection order (once deployment is known)

1. **HashiCorp Vault** — preferred when available (dynamic secrets, leasing,
   rotation, audit).
2. **OpenShift Secret** — only when the cluster has **encryption at rest**
   enabled and RBAC restricts access.
3. **Kubernetes Secret** — same caveat: base64 is *not* encryption; requires
   etcd encryption at rest + RBAC.
4. **Cloud secret manager** — AWS Secrets Manager / Azure Key Vault / GCP Secret
   Manager, matching the cloud.
5. **Encrypted properties** — e.g. Jasypt with the master key sourced from one
   of the above. Prefer this over an *unencrypted* platform Secret.
6. **Environment variables** — last resort; visible in process listings and
   crash dumps; acceptable only for low-sensitivity config.

> Important: an **unencrypted** Kubernetes/OpenShift Secret is weaker than
> encrypted properties. Rank by the actual protection in place, not the label.

## HashiCorp Vault (preferred pattern)

Prefer Spring Cloud Vault or the Vault Agent sidecar so the app reads secrets at
startup/refresh rather than embedding them.

```yaml
# application.yml — Spring Cloud Vault (KV v2)
spring:
  cloud:
    vault:
      uri: ${VAULT_ADDR}
      authentication: KUBERNETES
      kubernetes:
        role: ${VAULT_ROLE}
      kv:
        enabled: true
        backend: secret
        default-context: ${spring.application.name}
```

Use Kubernetes/OpenShift auth (service-account token) rather than a static Vault
token. Enable rotation/leasing for database and dynamic credentials.

## Remediating a hardcoded-credential finding

1. **Remove** the literal from source/config.
2. **Rotate** the exposed secret — assume it's compromised once committed.
3. **Externalize** to the chosen backend; inject via config, not code.
4. **Purge history** if it was committed (BFG / `git filter-repo`) and note it.
5. **Prove** it: the app boots reading the secret from the backend; the literal
   is gone from the tree and history.

## Migration, deployment & rollback guidance

Every secrets change must include:

- **Migration:** how the secret moves into the backend (Vault write / sealed
  secret / cloud entry) and how the app is granted read access.
- **Deployment:** manifest/config changes (see `openshift.instructions.md`),
  ordering (create secret before rolling the app).
- **Rollback:** how to revert safely **without** re-introducing the literal —
  keep the old secret in the backend until the new one is verified, then revoke.

## Forbidden

No hardcoded secrets, no secrets in logs or exceptions, no committing real
secrets to `application*.properties`/`.yml`, no plaintext secrets in container
images or Dockerfiles.
