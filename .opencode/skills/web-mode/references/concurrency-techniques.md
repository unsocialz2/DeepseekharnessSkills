# Concurrency Synchronization Techniques — on-demand reference

Load this only after a target-specific invariant and sequential control show that concurrency is a plausible security question. The vulnerability is the broken invariant; these are only ways to reduce timing uncertainty.

## Technique families

- **Grouped parallel requests** — send a small synchronized group and compare the durable result with the sequential control.
- **HTTP/2 multiplexing** — useful when multiple streams to the same origin can reduce connection/setup skew.
- **Single-connection synchronization** — useful when the application or proxy behavior makes connection reuse important.
- **Multiple connections** — useful when a front-end serializes work per connection but backend state is shared.
- **Two-session / two-actor coordination** — useful when the invariant spans separate principals or sessions.
- **Async job / webhook coordination** — relevant when the race is between a foreground action and delayed external/background processing.

## Warm-up and noise control

Where appropriate, establish connections and application state before the synchronized test so transport setup does not dominate the timing. Keep request counts minimal and within program rules.

## Evidence rule

A synchronization technique matters only if it produces a durable result impossible in the sequential control. Different response order, latency, duplicate UI messages, or multiple success-looking responses without multiple committed effects are not enough.

## Tool choice

Burp Repeater groups, Turbo Intruder, or a small async client can all be appropriate. Choose the simplest tool that matches the observed protocol and lets you preserve a clean sequential control.
