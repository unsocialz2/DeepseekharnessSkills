# DeepseekharnessSkills — DeepSeek Harness project instructions

This repository is a native DeepSeek Harness (DSH) workspace for authorized bug-bounty research.

## Project-root rule

- The repository root is the nearest ancestor containing this repository's `.git` directory.
- Never treat `$DSH_HOME`, `~/.dsh/profiles/*`, or a DSH profile directory as this project's root unless that directory is actually the Git repository.
- If a Web session starts with a cwd outside this repository, do not hunt for this repository's `AGENTS.md`, `HUNT_RULES.md`, target files, or skills inside the profile directory. Report a workspace mismatch and stop target actions until the correct workspace is selected.
- A session may start at the repository root or inside `targets/<target>/`; DSH should still resolve the Git root as the project root.

## Always-on operating contract

Before taking any target action, read `<repo>/HUNT_RULES.md` and follow it together with the active target's `rules.txt` and scope files. Target-local policy and explicit scope override generic skill guidance.

## DSH skill layout

- Project skills live under `<repo>/.dsh/skills/*/SKILL.md`.
- These are DeepSeek Harness filesystem skills, not OpenCode assets.
- Load the smallest relevant skill and only the exact deep-dive/reference required for the current decision.
- Do not preload overlapping modes or giant reference files.

## Target workspaces

- `targets/<target>/` is the working area for one program/target and is intentionally gitignored.
- Keep target evidence, state, recon artifacts, and notes inside the active target directory.
- Never read or write a sibling target unless the operator explicitly switches targets.

## Runtime tools

MCP servers, shell tools, browser automation, and other runtime capabilities are tools, not authorization. Scope and program rules remain authoritative. Prefer evidence-producing, low-impact actions and stop at policy/authentication/WAF/state-change boundaries defined by `HUNT_RULES.md` and target rules.

## Repository map

- `AGENTS.md` — DSH standing project instructions loaded as workspace context.
- `HUNT_RULES.md` — authoritative hunting contract for target work.
- `.dsh/skills/*/SKILL.md` — DSH skill routers.
- `.dsh/skills/*/SKILL_FILES/` — domain/stage deep dives.
- `.dsh/skills/*/references/` — optional lookup material loaded lazily.
- `targets/<target>/` — target-local scope, rules, state, findings, and recon data.
- `_template/` — starter target files.
- `ENV.md` — environment notes, not permanent truth.
- `PROMPT.md` — optional DSH operator prompts.
