---
name: automation-mode
description: Build or improve authorized bug-bounty automation: recon pipelines, asset/scope monitoring, notifications, custom nuclei templates, ffuf workflows, Burp automation, VPS/container setup, and hunting scripts. Generated automation must inherit the active target's scope/rules and avoid hardcoded aggressive defaults.
---

# AUTOMATION MODE — workflow/tool router

Automation exists to make an already-valid hunting workflow repeatable. It does not override scope, policy, evidence requirements, or target-local state.

## First questions

Before building/running automation, determine:

```text
INPUT:       <scope asset/list/artifact>
OUTPUT:      <artifact/notification/result>
STATE:       <what persists between runs>
DELTA:       <what counts as new work>
LIMITS:      <rules.txt rate/automation restrictions>
FAILURE:     <what should stop or retry>
```

Prefer the smallest reliable mechanism that fits the workload.

## Deep-dive router

Load one relevant file at a time.

### Workflow / operations
From `SKILL_FILES/AUTOMATION/`:
- `recon_pipeline_automation.md`
- `workflow_orchestration.md`
- `monitoring_new_assets.md`
- `monitoring_scope_changes.md`
- `monitoring_new_programs.md`
- `notification_telegram_discord.md`
- `nuclei_custom_templates.md`
- `ffuf_automation.md`
- `burp_suite_automation.md`
- `custom_tool_development.md`
- `docker_hunting_environment.md`
- `vps_setup_hunting.md`

### Scripting language / utility
From `SKILL_FILES/SCRIPTING/`:
- `bash_hunting_scripts.md`
- `python_hunting_scripts.md`
- `js_node_hunting_scripts.md`
- `ruby_hunting_scripts.md`
- `go_hunting_tools.md`
- `rust_hunting_tools.md`
- `one_liners_collection.md`
- `regex_patterns_hunting.md`

`references/scripts.md` and `references/environment.md` are large compatibility/code libraries. Do not load them by default; consult only when the smaller task file lacks the required implementation detail.

## Automation design rules

Every target-facing automation should be:

- **scope-aware** — reads explicit scope/allowlists rather than deriving permission from discovered assets;
- **rules-aware** — rate/concurrency/technique limits come from target policy or explicit operator settings;
- **idempotent** — reruns do not duplicate state or destroy prior evidence;
- **delta-driven** — expensive downstream work consumes new assets/results when practical;
- **observable** — records command/stage/error/artifact information;
- **recoverable** — partial failure does not corrupt the target workspace;
- **bounded** — no hidden infinite recursion, unbounded concurrency, or surprise full-port/full-path scans;
- **secret-safe** — tokens/credentials stay out of committed files and normal logs.

Never bake high request rates, huge wordlists, all-port scans, destructive actions, or broad cloud enumeration into a generic default. Those are selected per target after rules/scope allow them.

## Target-local state

Per-target pipelines write under the active TARGET_ROOT, normally:

```text
./recon_data/
./RECON_LOG.md
./findings.md
./lessons.md
```

Global monitors that span programs must keep their own clearly separated state and must not silently treat newly discovered assets as testable; scope confirmation comes first.

## Delegation

No fixed automation agent exists. The current model may implement directly or delegate a narrow code/review task when useful, then test/sanity-check the result itself.

## Boundary

`recon-mode` decides what recon stage is needed; automation-mode makes that stage repeatable. Vulnerability interpretation remains with the relevant technical skill.
