---
name: reporting-mode
description: Final bug-bounty report writing, CVSS/severity assessment, impact wording, remediation, platform-specific templates, and dispute/escalation communication. Use only after evidence is sufficient to reproduce and support the claimed impact.
---

# REPORTING MODE — final writeup / scoring router

This skill is the **single default owner of numeric severity/CVSS** across the project. Other hunting skills should pass demonstrated facts here rather than embedding their own score tables.

## Entry gate

Do not write a final report from a lead. Require:

```text
BUG CLASS:
AFFECTED ASSET/ENDPOINT:
ATTACKER PRECONDITION:
MINIMAL REPRO:
EXPECTED/CONTROL BEHAVIOR:
ACTUAL RESULT:
DEMONSTRATED IMPACT:
SCOPE/POLICY BASIS:
FALSIFICATION RESULT:
```

If a required fact is missing, identify the exact evidence gap and return to validation rather than filling it with assumptions.

## Deep-dive router

Load only the piece needed from `SKILL_FILES/REPORTING/`:

| Need | File |
|---|---|
| overall structure | `report_writing_methodology.md` |
| HackerOne report | `hackerone_report_template.md` |
| Bugcrowd report | `bugcrowd_report_template.md` |
| Intigriti report | `intigriti_report_template.md` |
| email disclosure | `email_disclosure_template.md` |
| impact wording | `impact_statement_writing.md` |
| PoC presentation | `poc_documentation.md` |
| remediation | `remediation_advice.md` |
| CVSS/severity | `severity_rating_cvss.md` |
| dispute/escalation | `escalation_dispute_templates.md` |

## Report workflow

1. Verify the evidence packet against the entry gate.
2. Identify the platform/program from target files when available.
3. Load the one platform template needed plus only any supporting reporting file required.
4. Write steps that reproduce the actual observed behavior, not a generalized exploit story.
5. Map each impact sentence to evidence.
6. Score only the demonstrated final capability.
7. Keep remediation specific to the broken trust boundary.

## Severity/CVSS rules

- Never inherit a numeric score from another skill without recomputing it from the actual finding.
- Never score a lead or hypothetical chain.
- Show the full vector when giving CVSS.
- If the platform uses a taxonomy/rating system (for example VRT), verify the **current** official taxonomy when it materially affects the report; do not rely on stale static mappings.
- If uncertainty remains in one metric, state the assumption and provide the conservative interpretation.
- Program-specific severity caps/eligibility in scope/rules override generic expectations.

## Freshness rule

Platform UI, VRT categories, SLAs, payout behavior, report fields, and policy conventions can change. Current `rules.txt`, scope exports, and official platform documentation outrank repository heuristics.

Use `platform-intel-mode` when current platform behavior/policy research is needed. Static platform files are background, not authority.

## Output standard

Default report shape:

```markdown
# <specific title: class + location + demonstrated impact>

## Summary
<2-3 sentences>

## Steps to reproduce
1. ...

## Proof / Evidence
<minimal request/response, PoC, screenshots/video references>

## Impact
<demonstrated attacker capability and affected trust boundary>

## Severity
<platform rating and/or CVSS vector with justification>

## Remediation
<specific control/root-cause fix>
```

Platform templates may adjust fields/order, but evidence remains primary.

## Writing rules

- No hype, no speculative mass impact unless scale was established.
- Redact credentials/PII while preserving reproducibility.
- Separate observed facts from assumptions.
- Do not bury prerequisites.
- Prefer one minimal PoC over multiple noisy variants.
- A report should be easier for a triager to reproduce than the original hunt was.
