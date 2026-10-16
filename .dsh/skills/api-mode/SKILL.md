---
name: api-mode
description: API security testing for REST, GraphQL, and gRPC. Use for API discovery, auth/authz matrices, BOLA/BFLA, mass assignment, versioning, API fuzzing, rate limits, GraphQL schema/introspection, and gRPC reflection/method testing. Browser/session-specific flows stay in web-mode.
---

# API MODE — REST / GraphQL / gRPC router

Target scope/rules and the global evidence contract come from `HUNT_RULES.md`.

## Boundary

Use this skill when the surface is primarily an API protocol or schema:
- REST/JSON/XML endpoints,
- GraphQL queries/mutations/subscriptions,
- gRPC services/methods,
- OpenAPI/Swagger-described functionality,
- API-wide authorization matrices and versioning behavior.

Use `web-mode` instead when the key trust boundary is a browser/session/form/rendering flow.

## API hunt loop

1. Inventory only the endpoints/schemas relevant to the current question.
2. Build the minimum auth matrix needed: unauthenticated / low-priv / higher-priv only where available and permitted.
3. Prioritize authorization before broad injection fuzzing.
4. Load one exact deep-dive file.
5. Confirm behavior with a minimal differential request.
6. Falsify alternate explanations such as cached data, error bodies with 200 status, leaked identifiers, or role mismatch.
7. Record demonstrated impact facts; scoring belongs to `reporting-mode` unless explicitly requested.

## Deep-dive router

All canonical files live under `SKILL_FILES/VULNERABILITIES/API/`:

| Need | File |
|---|---|
| REST-specific behavior and attack surface | `rest_api_attacks.md` |
| authentication/token/API-key handling | `api_authentication_attacks.md` |
| BOLA/BFLA/object/function authorization | `api_authorization_bypass.md` |
| writable hidden/read-only fields | `api_mass_assignment.md` |
| focused parameter/method/content-type fuzzing | `api_fuzzing_methodology.md` |
| rate-limit behavior | `api_rate_limiting_bypass.md` |
| old/alternate API versions | `api_versioning_attacks.md` |
| GraphQL schema/introspection/aliases/batching | `graphql_introspection_attacks.md` |
| gRPC reflection/metadata/method authorization | `grpc_attacks.md` |

GraphQL is canonical here. Do not load the legacy `web-mode/.../graphql_attacks.md` unless this API file is clearly missing something needed.

## High-value ordering

Prefer this order when evidence supports it:

1. missing authentication,
2. horizontal/vertical authorization gaps,
3. mass assignment / sensitive mutation fields,
4. version drift / forgotten endpoints,
5. rate-limit/state-machine weaknesses,
6. parser/content-type/method differentials,
7. protocol-specific injection or downstream bugs.

Do not spray every payload class across every parameter. Use `payloads` only after a context/engine or concrete hypothesis exists.

## API evidence rules

For authorization findings, keep a clean actor/resource matrix:

```text
ACTOR A: <role/account>
RESOURCE A: <owned id>
ACTOR B: <role/account>
RESOURCE B: <other id>
CONTROL: A -> A expected success
TEST:    A -> B unexpected success/failure detail
```

For GraphQL, name the exact operation/field/mutation and auth state. For gRPC, name service/method plus metadata/auth state.

## Cross-skill boundaries

- API endpoint discovery from recon -> `recon-mode` first, then this skill.
- Browser-only OAuth/SAML/session behavior -> `web-mode`.
- Cloud/container service reached through an API primitive -> confirm the API primitive here, then `infra-mode` for downstream infrastructure validation.
- Source-code root cause -> `scr-mode`.
- Minimal PoC/chaining after confirmation -> `exploit-mode`.
- Numeric severity/report -> `reporting-mode`.
