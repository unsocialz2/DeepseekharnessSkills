---
name: web-mode
description: General web application security testing for browser/session/form flows and classic web vulnerabilities. Use for auth/session flaws, IDOR on web flows, business logic, injections, XSS, CSRF, SSRF, request smuggling, cache issues, uploads, redirects, WebSockets, postMessage, CSP/CORS, and takeover. Route REST/GraphQL/gRPC-specific testing to api-mode.
---

# WEB MODE — focused web router

Target scope/rules and the evidence contract come from `HUNT_RULES.md`.

## Boundary

Use this skill when the trust boundary is primarily a **web application flow**: browser session, cookies, forms, redirects, rendering, caches, HTTP intermediaries, or ordinary server-side request handling.

Default to `api-mode` when the surface is primarily REST/GraphQL/gRPC/schema-driven API behavior. GraphQL belongs to `api-mode` by default even though a legacy web deep-dive file still exists.

## Hunt loop

1. Map the specific flow/state transition under test.
2. Pick the single most plausible vulnerability class from observed behavior.
3. Load only that class file.
4. Run the viability gate and the smallest proving test.
5. Actively falsify the strongest benign explanation.
6. If confirmed, map the primitive's real reach and only then test a plausible chain.
7. Record evidence; do not assign numeric severity here unless explicitly requested.

Do not run the entire class library against every endpoint.

## Routing by class

Deep-dive files live under `SKILL_FILES/VULNERABILITIES/WEB/`.

### Auth, session, authorization, business logic
- `auth_bypass_all_types.md`
- `2fa_mfa_bypass.md`
- `account_takeover_chains.md`
- `idor_bola_bfla.md` — use here for browser/web flows; API-wide object authorization belongs in `api-mode`
- `mass_assignment.md` — use here only for ordinary web backends; schema/API-focused work belongs in `api-mode`
- `jwt_attacks.md`
- `oauth_misconfig.md`
- `saml_attacks.md`
- `rate_limiting_bypass.md`
- `business_logic_bugs.md`
- `race_conditions.md`

### Injection / server-side processing
- `sqli_all_types.md`
- `ssrf_all_types.md`
- `ssti_all_engines.md`
- `xxe_all_types.md`
- `deserialization_all_langs.md`
- `rce_all_vectors.md`
- `lfi_rfi_path_traversal.md`
- `file_upload_bypass.md`
- `prototype_pollution.md`
- `dependency_confusion.md`

### Client-side / browser trust
- `xss_reflected_stored_dom.md`
- `dom_clobbering.md`
- `postmessage_attacks.md`
- `cors_misconfig.md`
- `csp_bypass.md`
- `clickjacking.md`
- `iframe_attacks.md`
- `open_redirect.md`
- `csrf_all_types.md`
- `csrf_websocket.md`
- `websocket_attacks.md`

### HTTP/cache/intermediary
- `http_request_smuggling.md`
- `http_parameter_pollution.md`
- `host_header_attacks.md`
- `cache_poisoning.md`
- `web_cache_deception.md`
- `cloudflare_bypass.md`
- `waf_bypass_all_techniques.md`

### DNS ownership edge
- `subdomain_takeover.md`

## On-demand pattern references

These preserve useful technique/pattern memory without turning the default hunt into a checklist. Load one only after target evidence creates the matching question:

| Question | Reference |
|---|---|
| edge/origin, path, method, host, protocol auth differential | `references/auth-routing-differentials.md` |
| authorization representation/capability differential | `references/authorization-differentials.md` |
| business-logic pattern inspiration | `references/business-logic-patterns.md` |
| account-control / takeover transition inspiration | `references/account-control-patterns.md` |
| OAuth/OIDC binding pattern inspiration | `references/oauth-oidc-patterns.md` |
| synchronization options after a race invariant exists | `references/concurrency-techniques.md` |

`references/vuln-library.md` is a large compatibility/reference bundle. **Do not load it by default.** Prefer the exact class file or small reference above; consult the big library only when the focused material lacks a needed detail.

## Falsification baseline

Examples of the mindset, not a checklist to run blindly:

- `200` is not auth bypass until the body/action proves privileged success.
- Reflection is not XSS until the actual sink executes in the real context.
- A timing spike is not SQLi until controlled repetitions distinguish it from jitter/WAF behavior.
- A URL fetch is not SSRF until server-side origin/reach is demonstrated.
- A single concurrent success is not a race until the sequential control behaves differently.
- A dangling CNAME is not takeover until the provider state shows practical claimability under program rules.

## Cross-skill use

- Exact payload variants -> `payloads`.
- REST/GraphQL/gRPC object/function authorization -> `api-mode`.
- Cloud credentials/services reached from SSRF -> validate the web primitive here, then use `infra-mode` for the cloud boundary.
- Source-code root cause -> `scr-mode`.
- Confirmed finding needing a compact PoC/chain -> `exploit-mode`.
- CVSS/platform-ready report -> `reporting-mode`.
