# Race Conditions — concurrency and invariant reasoning guide

Use this file when a security/business invariant may depend on the ordering or atomicity of two or more valid operations.

## Goal

Determine whether concurrent execution can produce a state that **cannot be produced by the same operations sequentially** and that gives a meaningful attacker advantage.

Do not assume every check-then-act flow is exploitable, and do not default to a fixed number of parallel requests or a specific transport trick.

## Identify the invariant first

Write the expected rule in target-specific terms, for example:

- one capability can be consumed once,
- balance/inventory/quota cannot be spent twice,
- only one conflicting state transition may commit,
- ownership/role checks and mutation must refer to the same authoritative state.

Then identify the operations that may contend on that invariant.

## Minimal hunt loop

1. Establish a stable **sequential control** showing the intended result.
2. Identify the smallest pair/group of operations that could overlap on the same state.
3. Synchronize only enough to test that specific overlap.
4. Compare the final durable state with the sequential control.
5. Repeat to establish reproducibility and distinguish a real race window from retries/eventual consistency.
6. Verify downstream effect from an independent read or ledger/state view when possible.
7. Falsify benign explanations such as idempotent responses, duplicate UI messages, delayed rollback, queue retries, non-authoritative caches, or intentionally multi-use capabilities.

## Concurrency model

Choose synchronization based on the observed stack rather than habit:

- same connection vs multiple connections,
- HTTP/1.1 vs HTTP/2,
- browser/UI concurrency,
- asynchronous jobs or webhook delivery,
- two different actors/sessions,
- read-check-write vs write-write conflicts.

The transport technique is secondary; the security claim is the broken invariant.

## Sub-state / TOCTOU reasoning

High-signal questions include:

- Is authorization/eligibility checked before the state that it depends on can change?
- Can one operation invalidate a condition while another still commits based on the old state?
- Are two services/transactions using different snapshots of the same business fact?
- Does a temporary intermediate state expose a capability that should not be externally reachable?

## Evidence contract

A strong race finding should preserve:

1. target-specific invariant,
2. sequential control,
3. concurrent operation set and synchronization method,
4. final durable state,
5. attacker advantage,
6. reproducibility rate or enough repeated evidence to establish the condition,
7. strongest benign explanation tested and rejected.

Do not assign numeric severity here.

## Stop conditions

Reject or narrow when:

- only response ordering changes while authoritative state remains correct,
- duplicate responses map to one committed effect,
- state self-corrects before any attacker advantage exists,
- the behavior is explained by documented eventual consistency,
- the result cannot be reproduced beyond ordinary network/retry noise.

## Tools on demand

Use synchronized-request tooling only after the invariant and control are clear. The exact tool or transport should be chosen to match the observed application stack; it is not part of the vulnerability definition.
