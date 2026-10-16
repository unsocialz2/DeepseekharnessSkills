---
name: platform-intel-mode
description: Bug-bounty platform and program intelligence for HackerOne, Bugcrowd, Intigriti, YesWeHack, Immunefi, target selection, triage communication, scope/policy research, and disputes. Treat repository platform notes as background and verify current platform/program facts before relying on them.
---

# PLATFORM INTEL MODE — current-policy router

This skill handles platform/program context, not vulnerability validation.

## Authority order

For any action affected by program/platform policy, use this order:

1. target-local `rules.txt` and scope export,
2. current program page/policy,
3. current official platform documentation/taxonomy,
4. repository platform notes as background heuristics only.

Static files in this repository can age. Do not present changing facts such as payout ranges, SLA timings, VRT mappings, UI fields, reputation mechanics, or platform-specific acceptance rules as current unless verified when they matter.

If current verification is unavailable, label the claim as unverified rather than turning an old heuristic into a rule.

## Deep-dive router

Load only the relevant file under `SKILL_FILES/PLATFORM_INTELLIGENCE/`:

| Need | Background file |
|---|---|
| HackerOne | `hackerone_tactics.md` |
| Bugcrowd | `bugcrowd_tactics.md` |
| Intigriti | `intigriti_tactics.md` |
| YesWeHack | `yeswehack_tactics.md` |
| Immunefi/Web3 | `immunefi_web3_tactics.md` |
| target/program selection | `program_selection_methodology.md` |
| triager communication/dispute strategy | `triage_communication_tips.md` |

These files provide starting heuristics, not authority over the current program.

## Common tasks

### Scope/policy analysis
Extract:

```text
IN-SCOPE ASSETS:
OUT-OF-SCOPE ASSETS:
ELIGIBILITY/SEVERITY CAPS:
PROHIBITED TECHNIQUES:
RATE/AUTOMATION LIMITS:
ACCOUNT/AUTH RULES:
NON-QUALIFYING CLASSES:
REPORTING REQUIREMENTS:
```

Feed the result into TARGET CONTEXT so technical skills obey it automatically.

### Program selection
Use durable factors instead of folklore:
- scope breadth and relevance to the operator's strengths,
- evidence of recent product/scope change,
- response quality observable from current public information,
- bounty/eligibility terms currently published,
- researcher density/duplicate risk where evidence exists,
- technical fit and realistic access prerequisites.

### Triage/dispute
Base arguments on reproducibility, demonstrated impact, scope language, current taxonomy/policy, and closely comparable accepted behavior. Do not argue from generic platform stereotypes.

## Boundaries

- Vulnerability testing stays in the technical domain skill.
- Numeric CVSS/final report structure belongs to `reporting-mode`.
- A platform note never expands target scope beyond the actual program files.
- Target-specific policy always wins over a generic platform convention.
