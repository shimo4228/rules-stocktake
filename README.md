Language: English | [日本語](README.ja.md)

# rules-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/rules-stocktake) [![GitMCP](https://img.shields.io/endpoint?url=https://gitmcp.io/badge/shimo4228/rules-stocktake)](https://gitmcp.io/shimo4228/rules-stocktake)

An [Agent Skill](https://agentskills.io/specification) that audits your **always-loaded behavioral rules** (`~/.claude/rules/`) for quality. It is the sibling of [skill-stocktake](https://github.com/shimo4228/skill-stocktake) with the cost model inverted: a skill's cost is trigger pollution, a rule's cost is **residency** — every line loads into every session, whether or not it changes behavior.

## Install

### Claude Code

```bash
# Copy the skill into your global skills directory
cp -r skills/rules-stocktake ~/.claude/skills/rules-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/rules-stocktake
```

## The inverted cost model

Rules have no usage axis — nothing measures "rule invocations", and the concept does not apply to unconditional loading. The audit replaces it with two static signals:

- **Residency density** — is each line worth reading every session? Rarely-needed reference material and long procedures get demoted to the probabilistic skill layer.
- **Substrate absorption** — does the harness (or the internalized conversation pattern) already do this without the rule? An absorbed rule is more urgent than an unused skill: the unused skill sleeps harmlessly, the absorbed rule actively overrides evolving harness defaults as a stale shadow.

## Modes

| Mode | Trigger | What it does |
|------|---------|--------------|
| **full** | default, or `/rules-stocktake full` | Read and evaluate every rule |
| **changed** | `/rules-stocktake changed` | Re-evaluate only rules changed since the last run; carry the rest forward from the ledger. Mechanical integrity checks still run over the full set — cross-reference breakage is invisible to rule mtimes |

## How It Works

1. **Phase 1 — Inventory + mechanical integrity checks**: Glob `~/.claude/rules/**/*.md`, read everything into one context, measure per-file line counts. Structural checks run as throwaway grep (enumerate/decide split): every `See skill:` target exists, extends-links resolve, origin headers present, the rules README tree matches the actual file list.
2. **Phase 2 — Evaluation**: a two-stage binary screen — Stage 1 is a six-question Yes/No checklist per rule (overlap with other rules; overlap with skills/memory; cross-references resolve; references current; **not yet absorbed by the substrate**; **dense enough to deserve residency**). Stage 2 generates rule-specific refutation questions that pressure-test any non-Keep draft verdict; Dissolve candidates must name their absorber concretely or the claim is refuted. Binary answers are evidence for a holistic verdict, never aggregated into a score.
3. **Phase 3 — Summary**: a `Rule | Lines | Verdict | Reason` table, closing with the total line count and its delta since the previous audit.
4. **Phase 4 — Consolidation**: candidates are confirmed **one by one** — each shows its evidence, then asks `[y/n/skip]`; no bulk approval, and you can stop at any point. Approved Improve/Update/Merge edits are applied in-session (rules are small; no improvement-engine skill exists for them). Demote hands skill creation off to a skill-creator; Dissolve offers to record the why in an ADR.

## Verdict Criteria

| Verdict | Meaning |
|---------|---------|
| **Keep** | Earns its residency: current, unique, dense |
| **Improve** | Worth keeping, but needs tightening — for rules this usually means *shorten* |
| **Update** | Referenced technology is outdated |
| **Merge into [X]** | Substantial overlap with another rule |
| **Demote to skill** | Valuable content that doesn't earn per-session residency; move to the probabilistic trigger layer, leaving a pointer |
| **Dissolve** | Absorbed by the substrate (harness went native) or the conversation (internalized). Retirement by *success*, not defect — delete before it becomes a stale shadow, record the why in an ADR |
| **Retire** | Defect-based removal: low quality, stale, broken beyond repair |

## Requirements

- Claude Code with the **Glob**, **Read**, **Edit**, and **Bash** tools (the audit runs in one main context — no subagents required).
- Optional: `jq` for the changed-mode timestamp check. The skill degrades gracefully without it.

## Syncing from the harness

The canonical copy of this skill lives in the author's live Claude Code harness. This repository is a one-way publication mirror:

```bash
scripts/sync-from-local.sh --dry-run   # report differences only
scripts/sync-from-local.sh             # apply to working tree (never commits)
```

## References

The two-stage binary-question design (screen → verdict pressure-test, holistic verdict, no score aggregation) follows the checklist-decomposition evaluation line: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226), CheckEval (arXiv:2403.18771), TICK (arXiv:2410.03608) — over-decomposition degrades correlation on holistic quality, hence six questions and no score.

The absorption question and the Dissolve verdict implement the **Scaffold Dissolution** concept of the Agent Knowledge Cycle — its two vectors (inward internalization, downward substrate absorption) are documented in [docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md).

## About this skill

This skill extends the **Curate** phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) — a Zenodo-citable six-phase bidirectional growth loop ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)) — to the rules layer: [skill-health](https://github.com/shimo4228/skill-health) covers structural debt, [skill-stocktake](https://github.com/shimo4228/skill-stocktake) covers skill quality, rules-stocktake covers the always-loaded rules that [rules-distill](https://github.com/shimo4228/rules-distill) (the Promote phase) produces — the inverse direction over the same boundary. AKC is one of three research lines by [@shimo4228](https://github.com/shimo4228), alongside [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) ([DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)) and [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice) ([DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)).

## License

MIT
