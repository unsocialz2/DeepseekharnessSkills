---
name: llm-mode
description: Security testing for LLM/GenAI systems including direct/indirect prompt injection, RAG poisoning, tool/plugin abuse, data exfiltration, model extraction, supply-chain issues, and resource exhaustion. Use when untrusted input reaches a model or a model can invoke tools/data sources.
---

# LLM MODE — AI trust-boundary router

Target scope/rules and the global evidence contract come from `HUNT_RULES.md`.

## Boundary

Use this skill to analyze **how untrusted input influences model behavior or how model output/tool access crosses trust boundaries**.

Do not treat "the model ignored an instruction" or a generic jailbreak as a security finding by itself. A candidate needs a concrete security consequence: unauthorized data exposure, action execution, privilege crossing, tenant isolation failure, sensitive tool use, or material availability/cost impact.

## Attack-surface map

Map four things before testing deeply:

```text
INPUTS:       user text / files / images / retrieved content / messages
CONTEXT:      system/developer prompts / RAG / memory / hidden metadata
CAPABILITIES: tools / browser / email / HTTP / DB / files / code / payments
OUTPUT SINKS: rendered HTML / markdown / downstream agents / stored content / actions
```

Then choose one deep-dive file.

## Deep-dive router

All canonical files are under `SKILL_FILES/VULNERABILITIES/LLM_AI/`:

| Need | File |
|---|---|
| overall test methodology | `llm_testing_methodology.md` |
| direct prompt injection | `prompt_injection_direct.md` |
| indirect prompt injection | `prompt_injection_indirect.md` |
| jailbreak techniques | `jailbreaking_techniques.md` |
| RAG poisoning | `llm_rag_poisoning.md` |
| tool/plugin/function-call abuse | `llm_plugin_attacks.md` |
| data exfiltration | `llm_data_exfiltration.md` |
| model extraction | `ai_model_extraction.md` |
| supply-chain/model artifact risk | `llm_supply_chain.md` |
| resource/token/cost exhaustion | `llm_dos_attacks.md` |

Load only the file matching the current trust boundary.

## Workflow

1. Identify the attacker-controlled source and the protected boundary.
2. Prove that the malicious instruction/data reaches the model path under test.
3. Prove the model can cause a security-relevant sink/action beyond merely generating text.
4. Use the smallest payload that distinguishes control from coincidence.
5. Falsify benign explanations such as normal model variability, hallucination, public data, or user-authorized tool access.
6. Reproduce enough times to establish the behavior when model nondeterminism matters.
7. Record exact prompt/input, relevant context, model/tool state, response/action evidence, and attacker prerequisites.

Do not assign generic numeric severity from bug class alone. Final scoring belongs to `reporting-mode` and must reflect the demonstrated downstream capability.

## Cross-domain handoff

- Prompt injection causing HTTP fetch/SSRF -> prove the LLM/control path here, then use `web-mode` or `infra-mode` for downstream reach/impact.
- Tool call hitting REST/GraphQL/gRPC -> use `api-mode` for API authorization behavior.
- Generated output leading to XSS/rendering bugs -> use `web-mode` for the sink.
- Source/RAG pipeline code review -> `scr-mode`.
- Compact confirmed chain -> `exploit-mode`.
- Platform-ready severity/report -> `reporting-mode`.

## Evidence shape

```text
ATTACKER INPUT:
MODEL/ROUTE:
TRUSTED CONTEXT TO PROTECT:
CAPABILITY/SINK:
MINIMAL REPRO:
OBSERVED ACTION/LEAK:
CONTROL/FALSIFICATION:
REAL IMPACT:
```
