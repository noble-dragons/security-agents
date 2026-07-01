---
name: 'OpenShift & Kubernetes Deployment'
description: 'Secure deployment guidance for OpenShift/Kubernetes/Helm: secret wiring, hardened workloads, rollout and rollback.'
applyTo: 'deployment/**,openshift/**,helm/**,k8s/**,**/Dockerfile,**/*.yaml,**/*.yml'
---

# OpenShift & Kubernetes Deployment

Authoritative for: **deployment-layer security wiring**. Choosing a secret
backend lives in `secrets-management.instructions.md`; app code in the Java/
Spring files.

Only apply this when the repo actually targets OpenShift/Kubernetes (confirm via
`deployment/`, `openshift/`, `helm/`, `k8s/`, or a `Dockerfile`). If the
platform is ambiguous, ask before recommending manifests.

## Injecting secrets (do not bake them into images)

Prefer Vault (via Vault Agent sidecar / CSI) or a platform Secret mounted at
runtime. Reference, never inline:

```yaml
envFrom:
  - secretRef:
      name: app-db-credentials   # created out-of-band, encrypted at rest
```

- Enable **encryption at rest** for etcd/Secrets; base64 is not encryption.
- Restrict access with RBAC; scope the ServiceAccount to only the secrets it
  needs.
- For Vault, authenticate with the Kubernetes/OpenShift ServiceAccount token,
  not a static token.

## Hardened workload (Pod/Deployment)

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
resources:
  requests: { cpu: "100m", memory: "256Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```

- OpenShift: prefer the `restricted-v2` SCC; don't grant `anyuid`/`privileged`.
- Set liveness/readiness probes; don't expose Actuator publicly (see
  `spring-security.instructions.md`).

## Container image hygiene

- Minimal, patched base image; run as a non-root UID.
- No secrets in build args, layers, or `ENV`.
- Pin versions; scan the image in CI.

## Network egress (supports SSRF remediation)

Where the platform allows, apply `NetworkPolicy` / egress rules so workloads
can only reach required destinations. This is a compensating control that
complements — does not replace — the application-level SSRF fix.

## Rollout & rollback

- Create/update the Secret **before** rolling the Deployment.
- Use a rolling update with probes; keep the previous ReplicaSet for quick
  `rollout undo`.
- Rollback must not re-introduce a removed hardcoded secret — keep the new
  secret in the backend and revoke the old one only after verification.
