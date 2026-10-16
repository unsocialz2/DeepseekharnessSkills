# Minimal master prompt for a target session

Run OpenCode from inside:

```text
<repo>/targets/<target>/
```

No manual target fields are required if the target folder already contains `scope.txt`/scope CSV and `rules.txt`.

Paste this once if you want to make the operating style explicit:

```text
Work autonomously from the current target workspace.

Read rules.txt first, then the available scope file/CSV, then existing STATE/findings/recon artifacts before taking target actions. Treat those files as authoritative and do not ask me for facts already present there.

Choose and load the smallest relevant OpenCode skill yourself. Do not force a fixed mode sequence and do not load overlapping skills unless the evidence crosses domains.

You own execution. If another model/subagent would materially help with parallel work, context isolation, or an independent falsification pass, delegate the narrow subtask and then synthesize/check its result yourself. There are no fixed roles or required model names.

Prefer the cheapest evidence-producing next step, avoid duplicate work, keep artifacts inside this target folder, and update concise state as the hunt progresses.
```

Optional direct asks still work:

```text
Hunt this target and prioritize the highest-signal surface.
Audit the recon already here and continue from the gaps.
Test this API for authorization flaws.
Validate this candidate and try to falsify it.
Build the smallest reproducible PoC for this finding.
Write this up for HackerOne/Bugcrowd/Intigriti.
STATE
```

The repository-wide operating contract is `.opencode/rules/HUNT_RULES.md`.
