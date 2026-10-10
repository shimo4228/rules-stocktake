Language: English | [日本語](README.ja.md)

# rules-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/rules-stocktake)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that audits your **always-loaded behavioral rules** (`~/.claude/rules/`) and gives each rule file a verdict, such as Keep, Improve, Demote to skill or Dissolve. It is the sibling of [skill-stocktake](https://github.com/shimo4228/skill-stocktake) with the cost model inverted: a skill's cost is trigger pollution (it can fire when it is not needed and make choosing the right skill harder), while a rule's cost is **residency**, because every line of a rule file without `paths:` frontmatter loads into every session whether or not it changes behavior.

The audit edits, demotes or deletes a rule only after you approve that rule, one at a time.

## Install

There are two routes. Cloning this repository (first block) installs rules-stocktake alone. The akc-cycle plugin (second block) also installs `skill-creator` and `adr-writer`, which the audit uses when it moves a rule's content into a new skill and when it records why a rule was deleted (see [Verdict Criteria](#verdict-criteria)). Take the plugin if you want those two steps to work without installing anything else.

```bash
git clone https://github.com/shimo4228/rules-stocktake
mkdir -p ~/.claude/skills
cp -r rules-stocktake/skills/rules-stocktake ~/.claude/skills/rules-stocktake
```

Then ask in plain words, such as "audit my rules", or type `/rules-stocktake`.

The same skill also ships in the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin, together with the other skills of the Agent Knowledge Cycle (AKC: the author's six-phase, human-gated cycle that turns a coding agent's repeated experience into skills and rules). rules-stocktake belongs to the cycle's Curate phase, and in the plugin it is called `/akc-cycle:rules-stocktake`. This repository is synced one way from the same source, so between syncs it can trail the plugin.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Requirements

- Claude Code with the **Glob**, **Read**, **Edit**, **Write**, and **Bash** tools (the audit runs in one main context; no subagents required).
- `jq` for the changed-mode timestamp check.
- The currency check behind the Update verdict (see [Verdict Criteria](#verdict-criteria)) may search the web to confirm that a referenced tool or flag is still current.
- The Phase 1 integrity checks assume the author's conventions (an `origin` header on line 1, `rationale:` and `review-when:` comments in `rules/common/` files, a `rules/README.md` table), so rule files without them are reported as findings: on a setup without these conventions, expect a finding on every file, which can feed an Improve candidate that you decline with `n`. The `rationale:` / `review-when:` check runs `~/.claude/scripts/hooks/harness_lint.py` from the author's harness, which this repository does not ship; SKILL.md defines no fallback for a machine without it. The condition that script checks is that both comments sit within a file's first 10 lines.
- On the clone route, Demote needs a skill named `skill-creator` installed (the author's version ships in the akc-cycle plugin); `adr-writer` is optional and only records the why of a Dissolve in an ADR (architecture decision record).

## The inverted cost model

Rules have no usage axis: nothing measures "rule invocations", and the concept does not apply to the unconditional loading of rule files without `paths:` frontmatter. The audit replaces it with two static signals:

- **Residency density**: is each line worth reading every session? Rarely-needed reference material and long procedures get demoted to a skill, which loads only when it is triggered.
- **Substrate absorption**: does the substrate, meaning the harness your rules sit on (system prompt, tool descriptions, built-in plan and review machinery), already do this without the rule, or is the principle so internalized that behavior no longer depends on the rule? An absorbed rule is more urgent than an unused skill: the unused skill's cost is the passive trigger pollution described above, while the absorbed rule actively keeps overriding the harness's newer defaults with older instructions.

## Modes

| Mode | Trigger | What it does |
|------|---------|--------------|
| **full** | default, or `/rules-stocktake full` | Read and evaluate every rule |
| **changed** | `/rules-stocktake changed` | Re-evaluate only rules changed since the last run; carry the rest forward from the ledger, `results.json`, which every run writes (Phase 4). On the clone route it is `~/.claude/skills/rules-stocktake/results.json`; the plugin copy reads `${CLAUDE_PLUGIN_ROOT}/skills/rules-stocktake/results.json`. Mechanical integrity checks still run over the full set, because cross-reference breakage is invisible to rule mtimes |

## How It Works

1. **Phase 1 — Inventory + mechanical integrity checks**: Glob `~/.claude/rules/**/*.md`, read everything into one context, measure per-file line counts. Structural checks run as throwaway grep, and the model judges only what the findings mean: every ``skill: `name` `` pointer resolves to a folder under `~/.claude/skills/` (pointers to plugin-installed skills come back unresolved), relative links between rule files resolve, every rule file carries an `origin` header, every `rules/common/` file carries `rationale:` and `review-when:` comments, and the rules README table matches the actual file list.
2. **Phase 2 — Evaluation**: a two-stage binary screen. Stage 1 is a six-question Yes/No checklist per rule (overlap with other rules; overlap with skills/memory; cross-references resolve; references current; **not yet absorbed by the substrate**; **dense enough to deserve residency**). Stage 2 pressure-tests every non-Keep draft verdict with rule-specific refutation questions (details in [SKILL.md](skills/rules-stocktake/SKILL.md)).
3. **Phase 3 — Summary**: a `Rule | Lines | Verdict | Reason` table, closing with the total line count and its delta since the previous audit. Each reason has to stand on its own; the skill's model Demote reason reads: "109 lines of pytest fixture recipes; only the 80%-coverage principle changes per-session behavior. Keep 3 lines + pointer, move recipes to the language-specific skill."
4. **Phase 4 — Consolidation**: candidates are confirmed **one by one**: each shows its evidence, then asks `[y/n/skip]`; no bulk approval, and you can stop at any point. Approved Improve/Update/Merge edits are applied in-session, because rule files are short enough to edit directly. Demote hands skill creation off to a skill named `skill-creator` and leaves a short pointer in the rule; Dissolve offers to record the why in an ADR through `adr-writer`. When a file is added, renamed or removed, the rules README table is updated to match. Every run also writes its verdicts, skipped ones included, to the ledger.

## Verdict Criteria

| Verdict | Meaning |
|---------|---------|
| **Keep** | Earns its residency: current, unique, dense |
| **Improve** | Worth keeping, but needs tightening; for rules this usually means *shorten* |
| **Update** | Referenced technology is outdated |
| **Merge into [X]** | Substantial overlap with another rule |
| **Demote to skill** | Valuable content that doesn't earn per-session residency; move it to a skill, leaving a pointer in the rule |
| **Dissolve** | Absorbed by the substrate (harness went native), or internalized so that behavior no longer depends on the rule. Retirement by *success*, not defect: delete it before it starts overriding newer harness defaults, and record the why in an ADR |
| **Retire** | Defect-based removal: low quality, stale, broken beyond repair |

## References

The two-stage Yes/No checklist draws on checklist-based evaluation research: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226), CheckEval (arXiv:2403.18771), TICK (arXiv:2410.03608). The absorption question and the Dissolve verdict implement the **Scaffold Dissolution** concept of the Agent Knowledge Cycle, in which a rule or skill succeeds when it can be deleted ([docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md)).

## More from the author

- **[Opus 5 Changed How Rules Should Be Written — Audit Yours](https://dev.to/shimo4228/opus-5-changed-how-rules-should-be-written-audit-yours-4fb4)** ([日本語](https://zenn.dev/shimo4228/articles/claude5-rules-official-shift-audit)): how the author checked each always-loaded rule against the instructions the new model actually loads and decided to keep, fix or retire it.
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: installs this skill together with the rest of the cycle as one Claude Code plugin.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Curate among them, recorded as dated design decisions.
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: the same kind of audit for your installed skills, finding staleness, conflicts and redundancy with a verdict per skill.
- **[agent-stocktake](https://github.com/shimo4228/agent-stocktake)**: the same kind of audit for subagent definitions, whose descriptions ride in every session while their bodies load only when called.
- **[rules-distill](https://github.com/shimo4228/rules-distill)**: the opposite direction; it finds principles that recur across skills and drafts them as always-loaded rules, with your confirmation for each.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

rules-stocktake is an Agent Skill for Claude Code that reads every rule file under `~/.claude/rules/` in one context and gives each a verdict, for people who maintain their own always-loaded rules and want them kept short, current and free of what the model or harness now does natively. The seven verdicts are Keep, Improve, Update, Merge into [X], Demote to skill, Dissolve and Retire; no rule file is edited, demoted or deleted without a one-at-a-time `[y/n/skip]` confirmation, while the ledger of verdicts is written on every run.

It exists because rules without `paths:` frontmatter are paid for in every session: each of their lines is a per-session token cost plus instruction dilution, and the longer the corpus, the weaker each rule's pull, so the Keep bar rises with total line count. Rule files scoped with `paths:` frontmatter load only when Claude works on matching files; SKILL.md enumerates every `.md` file under `~/.claude/rules/` and counts their lines with the rest, without treating path-scoped files separately. Rules have no usage signal, so the audit asks two static questions instead: is each line dense enough to deserve residency, and has the substrate (the harness) already absorbed the rule, or is its principle so internalized that behavior no longer depends on it? An absorbed rule must be surfaced as a Dissolve candidate, because it overrides newer defaults, and a Dissolve candidate that cannot name its absorber concretely is refuted. The answers to its per-rule Yes/No checklist are evidence for one overall verdict and are never aggregated into a score. This is the rules-layer form of AKC's Scaffold Dissolution, where a rule's success is that it can be deleted.

Canonical facts: MIT license; the skill payload (`skills/rules-stocktake/`) is a single `SKILL.md` with no scripts; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (`--dry-run` reports differences only; it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:rules-stocktake`, so this repository can trail the plugin between syncs. Requirements: Claude Code with Glob, Read, Edit, Write and Bash; `jq` for `changed` mode; no paid key. One integrity check (the `rationale:` / `review-when:` comments) runs `~/.claude/scripts/hooks/harness_lint.py` from the author's harness, which this repository does not ship, and SKILL.md defines no fallback without it. It applies approved Improve, Update and Merge edits itself, hands Demote to a skill named `skill-creator`, offers `adr-writer` for Dissolve, and writes a ledger, `results.json`, with per-rule line counts and the corpus total (`~/.claude/skills/rules-stocktake/results.json` on the clone route; the plugin copy reads it from `${CLAUDE_PLUGIN_ROOT}/skills/rules-stocktake/`).

Example: a run states the files found, total lines and integrity failures, then renders `Rule | Lines | Verdict | Reason` and closes with the total line count and its delta since the last audit. A Demote reason reads: "109 lines of pytest fixture recipes; only the 80%-coverage principle changes per-session behavior. Keep 3 lines + pointer, move recipes to the language-specific skill."

Links: [skills/rules-stocktake/SKILL.md](skills/rules-stocktake/SKILL.md) is the skill itself; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference; [docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md) in AKC explains the two absorption vectors. The skill extends the Curate phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) to the rules layer, concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
