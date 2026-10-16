---
name: infra-mode
description: Cloud and infrastructure security testing for explicitly in-scope AWS/GCP/Azure, S3/storage, Kubernetes, Docker, and network services. Use only after target scope/rules establish that the infrastructure or service is testable; brand/ownership resemblance alone is not enough.
---

# INFRA MODE — cloud / container / network router

Scope and program restrictions from `HUNT_RULES.md` are mandatory here because infrastructure testing can expand far beyond a web asset if boundaries are guessed.

## Scope gate

Before active infrastructure testing, answer:

```text
ASSET:        <host/IP/account/bucket/service>
SCOPE BASIS:  <exact scope/rules evidence tying it to this target>
ALLOWED TEST: <what program rules permit>
BOUNDARY:     <public unauth / provided creds / explicitly allowed assume-breach>
```

If scope basis is not explicit enough, keep the asset as a lead and do not actively test it.

Do not infer permission from ASN ownership, cloud account resemblance, certificate names, acquisition history, or brand association alone.

## Deep-dive router

Load one file at a time from `SKILL_FILES/VULNERABILITIES/INFRA/`:

| Surface | File |
|---|---|
| AWS cloud configuration | `cloud_misconfig_aws.md` |
| GCP cloud configuration | `cloud_misconfig_gcp.md` |
| Azure cloud configuration | `cloud_misconfig_azure.md` |
| S3/storage-specific testing | `s3_bucket_attacks.md` |
| Kubernetes | `kubernetes_attacks.md` |
| Docker | `docker_attacks.md` |
| network/service methodology | `network_pentest_methodology.md` |

## Workflow

1. Confirm the exact service and scope basis.
2. Start with the least-privileged/least-invasive observation that can prove exposure.
3. Load only the relevant deep-dive file.
4. Validate the trust boundary and permissions actually granted.
5. Falsify expected/public/by-design behavior before calling it a bug.
6. Expand privilege or downstream impact only when the current primitive and program rules justify it.
7. Record demonstrated capabilities; numeric severity belongs to `reporting-mode`.

## Boundary rules

- A web SSRF that reaches cloud metadata starts in `web-mode`; use this skill only for the downstream cloud identity/permission evidence after SSRF itself is proven.
- Public source/IaC misconfiguration found in code starts in `scr-mode`; use this skill if runtime cloud validation is needed and allowed.
- Recon may discover ports/cloud assets, but discovery is not proof that active exploitation is allowed.
- Assume-breach/lateral movement is **not** a default bug-bounty phase. Use it only when target rules or the operator explicitly define that engagement boundary.
- Avoid destructive mutations, persistence, broad data extraction, or availability impact unless explicitly required and authorized for a minimal proof.

## Evidence shape

```text
ENTRY STATE: <unauth / provided identity / leaked credential scope>
RESOURCE:    <exact in-scope service>
ACTION:      <minimal command/request>
RESULT:      <identity/permission/data/action proven>
CONTROL:     <why this is not expected public behavior>
IMPACT:      <what the proven permission enables>
```

Use `exploit-mode` only after a concrete primitive exists and a compact chain/PoC is needed. Use `reporting-mode` for CVSS/platform-ready severity language.
