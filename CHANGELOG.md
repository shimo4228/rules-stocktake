# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-07-03

Initial release as a standalone skill repository.

### Added

- `skills/rules-stocktake/SKILL.md` — quality audit for always-loaded behavioral rules (`~/.claude/rules/`): full/changed modes, Phase 1 mechanical integrity checks (See-skill pointers, extends links, origin headers, README tree), a six-question binary screen with refutation pressure-test, and seven verdicts (Keep / Improve / Update / Merge / Demote-to-skill / Dissolve / Retire) under an inverted, residency-based cost model.
- `scripts/sync-from-local.sh` — one-way export from the live Claude Code harness; the harness copy is canonical, this repository is the publication mirror.
- `.claude-plugin/marketplace.json`, `llms.txt`, `llms-full.txt`, bilingual README (en/ja).
