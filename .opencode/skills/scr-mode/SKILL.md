---
name: scr-mode
description: Source-code and repository security review. Use for taint/data-flow analysis, dangerous sinks, secrets, dependency risk, IaC, CI/CD, and language/framework-specific code auditing. Prefer sink-first review and load only the exact language/topic reference needed.
---

# SCR MODE — source review router

`HUNT_RULES.md` provides target scope, evidence, viability, and autonomy rules.

## Core loop

1. Map externally reachable entry points and trust sources.
2. Identify dangerous sinks or security-sensitive state transitions.
3. Trace attacker-controlled data source -> transformations/guards -> sink.
4. Record exact file:line evidence and the missing/broken control.
5. Determine runtime reachability. Unreachable code weakness = NOTE, not a validated finding.
6. If live validation is required, hand off to the owning runtime skill rather than duplicating it here.

Prefer manual data-flow reasoning over bulk tool output. Static tools generate leads; they do not establish exploitability by themselves.

## Deep-dive router

Language/topic files live under `SKILL_FILES/SOURCE_CODE_REVIEW/`:

| Need | File |
|---|---|
| general methodology | `scr_methodology_general.md` |
| dangerous functions across languages | `scr_dangerous_functions_all_langs.md` |
| JavaScript/Node | `scr_javascript_nodejs.md` |
| Python | `scr_python.md` |
| Java/Spring | `scr_java_spring.md` |
| PHP | `scr_php.md` |
| Ruby/Rails | `scr_ruby_rails.md` |
| Go | `scr_go.md` |
| .NET/C# | `scr_dotnet_csharp.md` |
| C/C++ | `scr_cpp.md` |
| Rust | `scr_rust.md` |
| Solidity/contracts | `scr_smart_contracts_solidity.md` |
| secrets | `scr_secrets_detection.md` |
| dependencies | `scr_dependency_analysis.md` |
| Terraform/Ansible/IaC | `scr_iac_terraform_ansible.md` |
| CI/CD | `scr_ci_cd_pipeline_review.md` |

`references/sinks.md` is a large sink-pattern library. Do not load it wholesale for every review. Use the language-specific file first, then consult only the relevant sink section/pattern when needed.

## Evidence shape

For a code candidate keep:

```text
SOURCE:       <file:line + attacker-controlled origin>
FLOW:         <key transformations/calls>
CONTROL:      <validation/authz/encoding/sandbox expected>
SINK:         <file:line + dangerous operation>
REACHABILITY: <route/handler/job + attacker prerequisites>
FALSIFICATION:<why guard/framework behavior does not neutralize it>
RUNTIME NEXT: <single test needed, if any>
```

## Secrets

A secret-looking string is a lead until context and, where safe/allowed, minimal validity/ownership evidence establish impact. Avoid broad use, data access, or mutation simply to prove a credential works. Target rules and provider-safe identity checks define the validation ceiling.

## Dependencies

A vulnerable version alone is not necessarily an exploitable finding. Establish whether the vulnerable code path/configuration is reachable in the target and whether a fix/backport/mitigation changes the result.

## Cross-skill boundaries

- Browser/web runtime validation -> `web-mode`.
- REST/GraphQL/gRPC runtime validation -> `api-mode`.
- Cloud/IaC runtime consequences -> `infra-mode`.
- Mobile application source -> `mobile-mode` when device/app behavior matters.
- Confirmed primitive needing compact PoC/chain -> `exploit-mode`.
- Numeric severity/report -> `reporting-mode`.

Do not assign numeric severity here by default; pass demonstrated reach and impact facts to reporting.
