---
name: payloads
description: Exact payload lookup by vulnerability class, sink context, database/engine/framework, and WAF. Use after a concrete context or hypothesis exists. Load only one matching reference sheet; never dump the whole library by default.
---

# PAYLOADS — runtime payload router

This skill is a lookup layer, not a discovery methodology.

## Serve rules

1. Identify the exact class and context first.
2. Identify DB/engine/framework/WAF only when evidence supports it.
3. Load **one** matching file from `references/`.
4. Return the smallest useful payload set for the current test.
5. If the first payload fails, reason from the observed parser/sink behavior before expanding variants.

Do not spray unrelated payload classes and do not load the whole library.

## Canonical runtime sheets

| Need | Canonical reference |
|---|---|
| XSS by sink/context | `references/1-xss-by-context.md` |
| SQLi by database | `references/2-sqli-by-db.md` |
| SSRF targets/bypasses | `references/3-ssrf-targets-bypasses.md` |
| SSTI by template engine | `references/4-ssti-by-engine.md` |
| XXE | `references/5-xxe.md` |
| command injection/RCE by language | `references/6-rce-command-injection-by-language.md` |
| LFI/path traversal/PHP wrappers | `references/7-lfi-path-traversal-php-wrappers.md` |
| JWT | `references/8-jwt-exploits.md` |
| CSRF | `references/9-csrf.md` |
| open redirect | `references/10-open-redirect.md` |
| prototype pollution | `references/11-prototype-pollution.md` |
| deserialization | `references/12-deserialization.md` |
| polyglots | `references/13-polyglots.md` |
| prompt injection | `references/14-prompt-injection-llm.md` |
| WAF bypass by vendor | `references/15-waf-bypass-by-vendor.md` |

## Duplicate-library rule

`SKILL_FILES/PAYLOADS/` is retained as upstream/deep-source material because other restored content may reference it. It is **not the normal runtime source** when an equivalent `references/` sheet exists.

Prefer `references/` to avoid loading two copies of essentially the same payload knowledge. Only inspect a `SKILL_FILES/PAYLOADS/*` file when the canonical reference is missing a needed detail.

## Context order

Choose payloads in this order:

```text
sink/context -> parser/engine/DB -> encoding/transport -> WAF/control variant
```

Examples:
- XSS: HTML body vs attribute vs JS string vs URL/DOM sink before WAF bypasses.
- SQLi: DB family and query context before obfuscation.
- SSTI: template engine before generic polyglots.
- SSRF: URL parser/redirect/DNS behavior before cloud-specific targets.

## Boundary

The owning technical skill (`web-mode`, `api-mode`, `llm-mode`, etc.) decides **whether** a payload test makes sense and evaluates the result. This skill only supplies context-matched payload material.
