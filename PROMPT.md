# Minimal master prompt for a DeepSeek Harness target session

Use the DeepSeek Harness Web workspace picker to select either the repository root or the active target directory:

```text
<repo>/targets/<target>/
```

DSH resolves the project root from the nearest `.git` ancestor, loads root `AGENTS.md`, and discovers project skills from `<repo>/.dsh/skills/` when the session cwd is inside this repository.

If a session tries to resolve project instructions from `~/.dsh/profiles/...`, the wrong workspace is selected. Switch the Web workspace to this repository or the intended target directory before target work.

No manual target fields are required when the target folder already contains `scope.txt`/scope CSV and `rules.txt`.

Paste this once if you want to make the operating style explicit:

```text
Work autonomously from the current target workspace.

Resolve the Git repository root first. Read the root AGENTS.md and HUNT_RULES.md, then read rules.txt, the available scope file/CSV, and existing STATE/findings/recon artifacts before taking target actions. Treat target-local scope and rules as authoritative and do not ask me for facts already present there.

Choose and load the smallest relevant DSH skill yourself from the project skill catalog. Do not force a fixed mode sequence and do not load overlapping skills unless the evidence crosses domains.

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

The repository-wide operating contract is `HUNT_RULES.md`; reusable DSH skills live under `.dsh/skills/`.
