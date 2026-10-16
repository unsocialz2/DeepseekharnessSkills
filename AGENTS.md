# BugbountySkills — OpenCode project map

This repository is a shared OpenCode rules/skills workspace for authorized bug-bounty research.

The authoritative always-on operating contract is `.opencode/rules/HUNT_RULES.md`, which is loaded explicitly by `opencode.json`. **Do not duplicate that contract here.**

## Layout

- `.opencode/rules/HUNT_RULES.md` — global hunting contract, scope/evidence/autonomy/state rules.
- `.opencode/skills/*/SKILL.md` — small domain routers.
- `.opencode/skills/*/SKILL_FILES/` — class/stage-specific deep dives.
- `.opencode/skills/*/references/` — optional lookup/background material; load lazily.
- `targets/<target>/` — local target workspaces containing scope/rules/state/evidence; `targets/` is gitignored.
- `ENV.md` — environment notes, not permanent truth.
- `PROMPT.md` — optional operator prompt examples, not an always-on contract.

Run OpenCode from the active `targets/<target>/` directory. Let `HUNT_RULES.md` and the target-local scope/rules drive behavior, and load only the smallest skill/reference needed for the current question.
