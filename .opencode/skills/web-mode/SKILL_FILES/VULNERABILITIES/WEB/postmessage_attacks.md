# postMessage Security — origin, data, and sink reasoning guide

Use this file when cross-window/frame communication via `postMessage`, `message` events, or equivalent browser messaging appears in the target.

## Goal

Determine whether an attacker-controlled window/origin can send data that the receiving application treats as trusted **without adequate sender/origin validation and downstream data validation**.

A missing `event.origin` check alone is not automatically a vulnerability; the received message must reach a security-relevant action, data disclosure, navigation, or executable sink.

## Map the message channel

For each relevant handler, record:

- receiver page/origin and state required,
- who can obtain a reference to the receiver window/frame,
- accepted sender origin/source assumptions,
- message data shape/schema,
- validation/normalization before use,
- downstream sinks/actions reached,
- whether a reply leaks data back to `event.source` or another window.

The key question is: **what trust decision is made based on message sender and message contents?**

## Minimal hunt loop

1. Capture one legitimate message exchange and its expected sender/receiver relationship.
2. Identify the exact origin/source/data checks performed by the receiver.
3. Alter one trust assumption at a time using a controlled sender context.
4. Trace accepted data to the final semantic action/sink.
5. Confirm the effect in the real application state.
6. Falsify benign explanations such as inert data handling, unreachable frames, source-window checks elsewhere, same-origin-only embedding, or a sink that escapes/sanitizes the value.

## High-signal questions

- Is origin validation exact and based on the canonical origin value?
- Is `event.source` or frame relationship important to the intended protocol?
- Are wildcard target origins used for sensitive replies?
- Can attacker-controlled message fields influence HTML/URL/navigation/script-like sinks?
- Does a message trigger authenticated actions without a user-intent/authorization check?
- Can the receiver expose sensitive data to an untrusted sender through a reply channel?
- Are nested frames, popup/opener flows, or alternate embedding contexts treated differently?

## Relationship to DOM XSS

If message data reaches an executable DOM sink, the final primitive is a DOM-XSS path. Use `xss_reflected_stored_dom.md` to reason about execution/reach after this file establishes the cross-origin message trust failure.

Do not automatically label every weak origin check as XSS.

## Evidence contract

A strong finding should preserve:

1. receiver context,
2. intended sender/origin relationship,
3. actual attacker-controlled sender context,
4. accepted message shape,
5. final security-relevant sink/action/data disclosure,
6. runtime proof,
7. negative/control message showing the intended boundary where possible.

## Stop conditions

Reject or narrow when:

- attacker cannot obtain a usable window/frame relationship,
- receiver rejects the untrusted sender,
- accepted data remains inert and causes no security-relevant effect,
- the only issue is stylistic/weak-looking origin code with no bypassable trust boundary,
- the tested effect is self-only with no meaningful cross-origin/security consequence.

## Tools on demand

Browser instrumentation/DOM messaging inspection can help enumerate real message flows, but use it to understand the protocol rather than to spray arbitrary message bodies.
