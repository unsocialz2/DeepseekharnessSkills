---
name: recon-mode
description: Reconnaissance and attack-surface discovery for an authorized target. Use for subdomains, DNS, alive probing, JS/archive mining, content discovery, API discovery, ports, fingerprinting, code leaks, cloud/mobile recon, or auditing existing recon. Audit target-local artifacts first and load only the exact stage file needed.
---

# RECON MODE — attack-surface router

Target context, scope, rules, autonomy, and no-repeat behavior come from `HUNT_RULES.md`.

## First action: audit before running

Inspect `./recon_data/`, `./RECON_LOG.md`, and obvious target-local recon outputs.

For each applicable stage mark:
- `DONE` — useful artifact exists and is fresh enough for the current decision;
- `PARTIAL` — artifact exists but a material part is missing/stale;
- `MISSING` — no usable evidence;
- `N/A` — scope/rules make the stage irrelevant or prohibited.

Do not rerun `DONE` work just because this skill loaded. Refresh only for changed scope, stale evidence, changed target behavior, or explicit operator request. Merge/diff instead of overwriting.

## Coverage gate

Core stages for ordinary web domains are:

1. passive seeds
2. enum + resolve
3. DNS analysis
4. alive inventory

If an applicable core stage is missing, recon is not complete. Other stages are selected by signal and scope rather than blindly running all 16.

Use this lightweight verdict:

```text
RECON AUDIT — <target>
CORE: <done/partial/missing summary>
HIGH-SIGNAL GAPS: <exact stages>
SKIPPED/N/A: <scope/policy reason>
NEXT: <single best next stage>
```

## Stage router

Load **one stage file at a time**:

| Stage | When needed | Canonical file |
|---|---|---|
| passive recon | initial seeds, org/domain pivots | `SKILL_FILES/RECON/01_passive_recon.md` |
| subdomain enum | permutations/bruteforce/resolution | `SKILL_FILES/RECON/02_subdomain_enumeration.md` |
| DNS | records, dangling CNAME, AXFR, email DNS | `SKILL_FILES/RECON/03_dns_analysis.md` |
| alive/assets | HTTP probing and alive inventory | `SKILL_FILES/RECON/04_asset_discovery.md` |
| JavaScript | endpoints, secrets, sourcemaps | `SKILL_FILES/RECON/05_javascript_analysis.md` |
| historical | Wayback/archive URLs and params | `SKILL_FILES/RECON/06_wayback_historical.md` |
| ports/services | explicit in-scope service discovery | `SKILL_FILES/RECON/07_port_scanning_services.md` |
| fingerprint | tech/WAF/CDN/service fingerprinting | `SKILL_FILES/RECON/08_fingerprinting_tech_stack.md` |
| GitHub/code leaks | public code exposure / secret leads | `SKILL_FILES/RECON/09_github_dorks_code_leaks.md` |
| search-engine dorks | indexed exposures | `SKILL_FILES/RECON/10_google_dorks.md` |
| internet search engines | Shodan/Censys/FOFA pivots | `SKILL_FILES/RECON/11_shodan_censys_fofa.md` |
| cloud recon | AWS/GCP/Azure asset discovery | `SKILL_FILES/RECON/12_cloud_recon_aws_gcp_azure.md` |
| mobile recon | app metadata/APK-derived backend map | `SKILL_FILES/RECON/13_mobile_app_recon.md` |
| API discovery | Swagger/OpenAPI/GraphQL/gRPC/WS discovery | `SKILL_FILES/RECON/14_api_discovery.md` |
| content discovery | paths/params/files fuzzing | `SKILL_FILES/RECON/15_content_discovery_fuzzing.md` |
| recon automation | recurring/diff-driven pipeline | `SKILL_FILES/RECON/16_recon_automation_pipeline.md` |

`references/pipelines.md` is a large command library. Do **not** load it by default. Use it only when an exact multi-stage command/flag set is needed and the smaller stage file is insufficient.

## Scope/rate rules

- `rules.txt` and scope files override every generic rate, port range, brute-force list, recursion depth, or automation suggestion in recon docs.
- A sibling/acquired/brand-related domain found during OSINT is a lead, not permission to test it.
- Port scanning, content fuzzing, and aggressive permutations are conditional on program rules; passive discovery can still be useful when active work is restricted.
- Customer/CPE/dynamic-pool looking infrastructure is never assumed in scope from ownership resemblance alone.

## Recon output contract

Every executed stage should leave:

1. a target-local artifact under `recon_data/` (or a clearly named target-local file),
2. a concise `RECON_LOG.md` row with date/stage/artifact/delta,
3. deduped output suitable for downstream use,
4. only the **delta** fed into expensive downstream stages when practical.

## Handoff boundaries

- A live web surface -> `web-mode`.
- REST/GraphQL/gRPC/API authz -> `api-mode`.
- Confirmed cloud/container/network surface -> `infra-mode`.
- APK/IPA/device analysis -> `mobile-mode`.
- Public source repository/code -> `scr-mode`.
- A concrete candidate needing validation/chaining -> corresponding domain skill first; `exploit-mode` only after a real primitive exists.

Recon discovers and prioritizes. It does not need to finish every possible stage before high-signal manual hunting starts.
