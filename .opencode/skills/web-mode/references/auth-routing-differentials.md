# Authentication / Routing Differentials — on-demand inspiration

Load this reference only after the authentication flow suggests that multiple layers may disagree about routing, normalization, protocol, or identity state. It is not a default bypass checklist.

## When this reference is useful

Use it when evidence suggests one of these conditions:

- an edge/gateway and origin enforce different rules,
- two routes or API versions expose the same protected capability,
- authentication is enforced before URL/path normalization,
- a reverse proxy rewrites paths or trusts forwarding headers,
- HTTP/1.1 and HTTP/2 take materially different routing paths,
- virtual-host selection changes the backend reached,
- a client-visible route and a backend route disagree about auth state.

## Differential families

### Path and normalization

Compare only variants relevant to the observed stack:

- trailing slash / dot-segment normalization,
- percent-encoding or double-decoding,
- path parameters / matrix parameters,
- duplicate slashes,
- case sensitivity,
- alternate route aliases or legacy versions,
- reverse-proxy rewrite paths.

The useful signal is not a different status code; it is a protected backend action/data set becoming available through a differently normalized path.

### Method and protocol

When the same resource is reachable by more than one method or protocol, compare the authentication decision across:

- GET/POST/PUT/PATCH/DELETE only where the app exposes equivalent semantics,
- method override mechanisms only if the stack uses them,
- HTTP/1.1 vs HTTP/2 when a proxy/backend split is visible,
- alternate content types/parsers when auth middleware placement appears parser-dependent.

### Proxy / rewrite headers

Only test forwarding or rewrite headers if the architecture actually contains a trusted proxy/gateway layer. Relevant families include:

- original/rewrite URL headers,
- client IP / forwarding headers,
- forwarded host/proto headers,
- internal routing headers revealed by the application or infrastructure.

A header is security-relevant only if it changes the authenticated identity, protected route, or authorization decision at the origin.

### Host / virtual-host routing

Use only when host-based routing is visible. Compare whether a protected internal/admin virtual host can be selected from an external path and whether authentication is enforced after routing.

### Alternate service/version

If two endpoints implement the same sensitive action, compare the required auth state rather than merely whether both exist. Old mobile endpoints, legacy API versions, and alternate callback handlers are useful only when they reach the same security boundary with weaker proof.

## Authentication-specific state differentials

When relevant, compare:

- normal login vs remembered-device flow,
- password login vs OAuth/SAML/SSO,
- primary-authenticated vs MFA-complete session,
- password-reset/recovery state vs normal authenticated state,
- fresh vs stale/parallel login flows,
- session before vs after security-sensitive changes.

## Guardrails

- Do not spray every header/path variant against every endpoint.
- Do not call a `200`, redirect, or different error body a bypass without protected semantic success.
- Do not treat CDN/WAF behavior as application auth bypass until the origin-side privilege change is proven.
- Prefer one architecture-driven differential with a clean control over a large generic bypass wordlist.
