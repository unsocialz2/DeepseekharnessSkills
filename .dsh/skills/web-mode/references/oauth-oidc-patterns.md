# OAuth / OIDC Pattern Families — on-demand reference

Load only after the normal authorization/login flow is mapped. These are protocol questions to consider when the observed deployment makes them relevant; they are not a default mutation checklist.

## Redirect binding

Check whether authorization responses are bound to the exact registered client and redirect destination intended by the deployment. Compare only variants that the product or provider actually accepts.

## Request / callback correlation

Relevant guarantees may include state, nonce, transaction identifiers, PKCE, or server-side pending-login records. The security question is whether the callback consumed by the application is bound to the same browser/session/client flow that initiated it.

## Authorization-code lifecycle

Where applicable, verify intended one-time use, client binding, redirect binding, expiry, and PKCE verification. A reusable artifact matters only if it grants capability outside the intended flow.

## Identity and account binding

The application should bind the verified identity-provider subject/issuer to the correct local account. Pay attention to merge/link/unlink flows, provider changes, verified-email assumptions, and duplicate-account reconciliation.

## Scope / audience / client boundaries

Ask whether the application accepts a token or authorization result intended for a different client, audience, resource, tenant, or scope.

## Alternate flows

Mobile, SPA, native-app, legacy, and enterprise SSO paths may use different protocol features. Compare them only when they result in the same local account or protected capability.

## Use rule

Start with one observed protocol guarantee, identify the artifact/binding that enforces it, and alter one assumption at a time. Do not force an account-takeover narrative; report the actual capability reached.
