# Security & privacy — what to add to every system design

Security is not a “nice-to-have” section at the end — it changes architecture.

This guide lists the security and privacy decisions that interviewers and real production systems expect.

---

## 1) Threat model (start simple)

Ask:

- Who are the actors? (users, admins, services)
- What are the assets? (PII, money, secrets, availability)
- What are the threats?
  - account takeover
  - data leaks
  - abuse/spam
  - fraud
  - DDoS

You don’t need a perfect model; you need to show you think adversarially.

---

## 2) Authentication (who are you?)

Common patterns:

- OAuth2/OIDC for user auth
- API keys for B2B integrations
- short-lived access tokens + refresh tokens

JWT vs opaque tokens:

- JWT:
  - fast verification at gateway
  - harder revocation (use short TTL, blocklists)
- Opaque:
  - easy revocation via introspection
  - adds dependency and latency

Best practice:

- validate tokens at gateway
- propagate identity claims to services
- services still authorize actions (defense in depth)

---

## 3) Authorization (what can you do?)

Types:

- RBAC: roles (admin, user)
- ABAC: attributes (tenant_id, region)

Rules:

- **least privilege**: give only needed permissions
- separate admin access paths and audit them
- multi-tenant isolation:
  - every request must carry `tenant_id`
  - every query must filter by tenant

---

## 4) Secrets management

Never hardcode secrets.

Use:

- secret managers (AWS Secrets Manager, Vault, etc.)
- rotation policies
- separate secrets per environment (dev/stage/prod)

Operational tip:

- restrict who can read production secrets
- audit secret access

---

## 5) Encryption

### In transit

- TLS everywhere (client→gateway, gateway→services, service→DB)
- mTLS for service-to-service in high-security environments

### At rest

- encrypt disks/storage
- encrypt sensitive fields (defense in depth)

Key management:

- central KMS
- rotation and access controls

---

## 6) PII handling and data minimization

Principles:

- collect only what you need
- store only what you need
- retain only as long as needed

Classify data:

- PII: email, phone, addresses
- sensitive PII: government IDs, precise location
- payment data: card numbers (avoid storing)

Practices:

- masking/redaction in logs
- tokenization (for payment methods)
- access audits

---

## 7) Logging and audit trails

You need audit logs for:

- admin actions
- permission changes
- payment/refund events
- data exports

Audit logs should be:

- append-only
- tamper-resistant (restricted write access)
- retained longer

---

## 8) Abuse prevention

Abuse is a security problem and a reliability problem.

Common controls:

- rate limiting and quotas
- CAPTCHA for suspicious flows
- anomaly detection signals (login attempts, OTP frequency)
- email/phone verification
- device fingerprinting signals (careful with privacy)

For APIs:

- per-key quotas
- request signing
- WAF rules

---

## 9) Privacy and compliance basics

Examples:

- GDPR: right to access/delete, data minimization
- SOC2: controls and audits
- PCI: payment data handling rules

Design implications:

- build deletion workflows (and propagation to derived systems)
- define retention policies
- document data flows

---

## 10) Secure-by-default architecture checklist

- validate and sanitize inputs
- protect against SSRF and injection
- set request size limits
- separate public and internal networks
- use allowlists for outbound connections
- run services with minimal permissions

---

## 11) Interview-ready summary

“We secure the system with strong authn/authz (least privilege, tenant isolation), secrets management with rotation, TLS in transit and encryption at rest via KMS, and careful PII handling (minimization, masking in logs, retention policies). We add audit logs for sensitive actions, and we mitigate abuse with multi-layer rate limiting, WAF rules, and anomaly detection. We design for compliance by supporting data deletion/export workflows and documenting data flows.”

