---
name: mobile-mode
description: Android/iOS security testing for authorized mobile targets. Use for policy triage, APK/IPA acquisition, static analysis, intents/deep links, WebViews, dynamic instrumentation, pinning, and mobile-only backend discovery. Read target rules first and verify the actual device/tool environment at runtime.
---

# MOBILE MODE — policy-first mobile router

Target scope/rules come from `HUNT_RULES.md`. Mobile programs often exclude entire classes based on rooting, ADB, physical access, pinning, WebViews, or malware-app prerequisites, so policy triage is the first technical step.

## Start here

1. Read target `rules.txt` and scope.
2. Load `SKILL_FILES/01_policy_triage_gate.md` if mobile exclusions/qualifiers are not already clear.
3. Check `ENV.md` for local host/device notes, but treat them as dated environment evidence rather than permanent truth.
4. Verify actual installed/runtime versions when the exact Frida/ADB/device/tool version matters.
5. Choose one mobile workflow below; do not load every file.

## Deep-dive router

| Need | Canonical file |
|---|---|
| host/device/ABI/tool preflight | `SKILL_FILES/00_environment_preflight.md` |
| program mobile exclusions and reportability gate | `SKILL_FILES/01_policy_triage_gate.md` |
| scenario/impact examples | `SKILL_FILES/02_real_world_scenarios.md` |
| intents, exported components, deep links, URL schemes | `SKILL_FILES/03_intent_deeplink_audit.md` |
| APK/IPA static workflow, manifest, strings, endpoints | `SKILL_FILES/04_static_workflow.md` |
| Frida/dynamic instrumentation/pinning runtime work | `SKILL_FILES/05_frida_dynamic.md` |

The top-level skill intentionally does **not** pin Frida/tool versions. Environment-specific pins belong in `ENV.md` or the dynamic reference and must be checked against the actual machine before use.

## Workflow

1. Policy gate.
2. Cheapest static evidence first: package metadata, manifest/entitlements, exported components, URLs, hardcoded config, backend map.
3. Device/environment preflight only when runtime testing is needed.
4. Dynamic instrumentation only when static evidence cannot settle the question.
5. For each candidate, prove a realistic attacker path under the program's allowed device assumptions.
6. Falsify rooted/emulator/sideload-only artifacts when the real target population would not expose them.
7. Record demonstrated impact; numeric severity belongs to `reporting-mode`.

## Boundary rules

- Lack of obfuscation, lack of pinning, root detection behavior, backup flags, and similar best-practice issues are not findings unless target policy accepts them and concrete impact is demonstrated.
- A rooted/emulated lab may be useful for analysis but does not automatically make rooted-only impact reportable.
- APK-derived backend endpoints hand off to `api-mode`/`web-mode` for server-side validation; do not duplicate those methodologies here.
- Deep links/WebViews stay here until the exploit crosses into a backend/API boundary.
- Tool/version troubleshooting should consult `ENV.md` and the exact deep-dive file, not be embedded permanently in this router.

## Evidence shape

```text
APP/PACKAGE:
DEVICE STATE: <stock/rooted/emulator/physical + relevant policy>
ENTRY VECTOR: <intent/deeplink/webview/local storage/etc>
PREREQUISITE:
MINIMAL REPRO:
RESULT:
REAL-USER IMPACT:
POLICY FIT:
```

Use `exploit-mode` only after the mobile primitive is concrete and a compact PoC/chain is needed. Use `reporting-mode` for final severity/writeup.
