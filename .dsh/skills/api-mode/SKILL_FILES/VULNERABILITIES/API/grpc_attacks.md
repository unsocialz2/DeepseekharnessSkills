# gRPC Security — service/method and metadata reasoning guide

Use this file when the target exposes gRPC and the security question involves service discovery, method-level authentication/authorization, protobuf input authority, streaming state, or gateway differences.

## Goal

Determine whether a caller can invoke a **service/method or resource action** without the intended identity, role, tenant, or input-state guarantees.

gRPC reflection or binary protocol visibility is not a vulnerability by itself.

## Map the service boundary

For the relevant service, record:

- transport/TLS and reachable endpoint,
- service and method name,
- unary vs streaming behavior,
- protobuf request/response fields relevant to the question,
- authentication metadata or mTLS/service identity,
- actor/tenant/resource context,
- gateway/proxy translation if REST or grpc-web reaches the same backend method.

Prefer reflection, known `.proto` files, client artifacts, or captured traffic for discovery. Do not fuzz arbitrary methods when the service model is unknown.

## Minimal hunt loop

1. Capture or construct one legitimate method call.
2. Identify the exact authentication/authorization/input-state guarantee under test.
3. Alter one actor/resource/metadata/input relationship at a time.
4. Verify semantic result and authoritative state, not only gRPC status codes.
5. Compare alternate gateway/transport paths only when they invoke the same backend method.
6. Falsify public health/discovery methods, intentionally anonymous services, default/error responses, and mutations that do not persist.

## High-signal questions

- Is authentication metadata enforced consistently across methods in the same service?
- Does method-level authorization bind the actor to the requested object/tenant?
- Can client-supplied protobuf fields set server-owned role/owner/state values?
- Do streaming methods retain auth/tenant state correctly across the stream lifecycle?
- Do REST/grpc-web gateways and direct gRPC paths apply equivalent security controls?
- Does reflection expose only schema metadata, or does it reveal a method that is itself improperly protected?

## Evidence contract

A strong finding should preserve:

1. service/method,
2. transport/auth metadata state,
3. actor/resource relationship,
4. normal control call,
5. altered call,
6. semantic data/action/state proof,
7. negative/falsification control.

## Stop conditions

Reject or narrow when:

- only reflection/service names are exposed,
- different status codes do not produce protected capability,
- the method is intentionally public,
- the caller legitimately has the required service/tenant relationship,
- a gateway difference does not change backend authorization or state.

## Cross-skill use

Use `api_authentication_attacks.md`, `api_authorization_bypass.md`, and `api_mass_assignment.md` for the corresponding underlying boundary. Use protocol tooling to reproduce the exact method call, not as a substitute for a security hypothesis.
