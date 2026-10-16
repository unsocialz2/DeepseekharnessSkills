# API Fuzzing — hypothesis-driven differential testing guide

Use this file when the API surface is understood well enough that targeted input variation can answer a concrete parser, validation, authorization, or state question.

## Goal

Use **small, discriminating variations** to discover where equivalent inputs are parsed, validated, authorized, or normalized differently.

Do not inject every vulnerability-class payload into every parameter.

## Start from a question

Before fuzzing, state what you are trying to distinguish, for example:

- does this field accept a different type than the schema/UI suggests,
- do JSON/form/XML paths reach different validators,
- do two HTTP methods hit the same business action with different middleware,
- does an undocumented parameter change ownership/role/state,
- does parser normalization create a security-relevant discrepancy?

## Minimal loop

1. Establish a stable control request.
2. Change one dimension only: value, type, field presence, nesting, content type, method, header, or path representation.
3. Compare semantic result and authoritative state.
4. Repeat enough to rule out cache/jitter/random backend errors.
5. Escalate to a vulnerability-specific skill only when the differential supports that hypothesis.
6. Stop broadening when additional variations no longer change the security question.

## Useful variation dimensions

Select only those justified by observed API design:

- type: string/integer/boolean/null/array/object,
- presence: omitted/empty/default/duplicate field,
- nesting/parent-child representation,
- JSON vs form/XML/multipart when the API actually accepts them,
- method variants that plausibly map to the same action,
- version/route aliases,
- documented vs undocumented fields,
- boundary/length/enum values,
- header-driven behavior where the gateway/framework exposes such controls.

## Auth and state before payloads

Prioritize identity, authorization, writable-state, version drift, and parser differences before generic injection payloads. Once a concrete sink/engine is suspected, hand off to the owning vulnerability skill or `payloads` for exact probes.

## Evidence contract

Record:

1. control request,
2. single changed dimension,
3. semantic/state difference,
4. why that difference matters to a security boundary,
5. repeatability,
6. strongest benign explanation tested.

## Stop conditions

Stop or narrow when:

- changes only affect validation/error formatting,
- no authoritative state/capability difference exists,
- behavior is documented schema coercion,
- broad variation is producing noise without a concrete hypothesis.

## Tools

Choose schema-aware/manual tooling that preserves clean controls and low-noise differentials. Tool choice is secondary to the experiment design.
