# SQL Injection — differential reasoning guide

Use this file when observed behavior suggests user-controlled data may alter a database query or query-construction path. This is a methodology guide, not a payload library.

## Goal

Determine whether attacker-controlled input changes the **structure or semantics of a backend query** in a way the application did not intend.

Do not infer SQL injection from one error, one slow response, or the presence of a database-looking parameter.

## Model the query boundary

For the candidate input, identify what you can infer about:

- input type and parser before the database layer,
- whether the value is likely data, identifier, sort/order expression, filter fragment, search expression, or raw query text,
- framework/ORM/query-builder behavior,
- transformations or normalization before query construction,
- database family only when evidence supports it,
- whether the result is reflected, boolean-differential, timing-only, error-based, or out-of-band.

The useful question is: **does changing this input produce a controlled database-dependent semantic difference that cannot be explained by normal application logic?**

## Minimal hunt loop

1. Establish a stable baseline request and response/timing distribution.
2. Change one input in a way that tests query interpretation without destructive side effects.
3. Compare a matched positive/control pair rather than a single anomalous response.
4. Repeat enough times to distinguish deterministic behavior from network/WAF/backend jitter.
5. Identify the narrowest supported technique class: boolean/content differential, controlled error, timing, or another query-dependent effect.
6. Falsify alternate explanations such as validation branches, search semantics, cache variance, rate limiting, WAF delay, or backend instability.
7. Only after injection is established, determine the minimum additional evidence needed to demonstrate impact under program rules.

## High-signal contexts

Pay attention when the application exposes query-construction escape hatches such as:

- sort/order/group fields,
- search/filter builders,
- reporting/analytics expressions,
- raw ORM fragments,
- dynamic table/column selection,
- stored values later reused in query construction,
- alternate parsers for JSON/form/query inputs.

These are hypotheses, not assumptions of vulnerability.

## Timing evidence

A timing candidate requires controlled repetition and a matched control. Record baseline variance and show that the delay tracks the input condition rather than infrastructure noise.

Do not treat one slow response as SQL injection.

## Error evidence

Database-looking errors are leads. Confirm that attacker input influences the query path and that the behavior is reproducible. Generic 500s, parser errors, and WAF blocks do not establish SQL injection.

## Second-order behavior

When input is stored and later processed, preserve the exact storage-to-query path. Prove that the later query consumes the attacker-controlled value and that the effect is not merely unsafe storage or logging.

## Impact reasoning

After confirmation, separate:

- query manipulation only,
- limited data inference/read capability,
- broader read/write capability,
- access to database features outside normal application data access.

Do not assume full database compromise or code execution from the injection primitive alone.

## Evidence contract

A strong finding should preserve:

1. exact input and endpoint,
2. baseline/control behavior,
3. matched test behavior,
4. repeated evidence showing deterministic query-dependent effect,
5. inferred query/context and confidence level,
6. strongest benign explanation tested and rejected,
7. minimum demonstrated impact.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Park or reject when:

- differences are not repeatable,
- timing remains inside normal variance,
- only generic validation/WAF/backend errors occur,
- application semantics explain the response difference without query manipulation,
- the candidate cannot be isolated to attacker-controlled input.

## References on demand

Use the `payloads` skill only after the query context or database family is understood. Canonical runtime material lives in `payloads/references/2-sqli-by-db.md`. Load only the small subset that answers the current question; do not copy the whole payload sheet into the hunt context.
