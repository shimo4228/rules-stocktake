Language: [English](README.md) | 日本語

# rules-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/rules-stocktake)

**常時ロードされる行動ルール**（`~/.claude/rules/`）の品質を監査する [Agent Skill](https://agentskills.io/specification)。[skill-stocktake](https://github.com/shimo4228/skill-stocktake) の姉妹スキルだが、コストモデルが反転している: skill のコストはトリガー汚染、rule のコストは**常駐** — 行動を変えるかどうかに関係なく、全行が全セッションにロードされる。

## インストール

### Claude Code

```bash
cp -r skills/rules-stocktake ~/.claude/skills/rules-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/rules-stocktake
```

## 反転したコストモデル

rules には usage 軸が存在しない —「rule の呼び出し回数」を測るものはなく、無条件ロードにその概念は成立しない。監査はこれを 2 つの静的シグナルで置き換える:

- **常駐密度** — 各行は毎セッション読ませる価値があるか。稀にしか要らない参照資料や長い手順書は、確率トリガーの skill 層へ降格する。
- **substrate 吸収** — harness（または内在化した会話パターン）が rule なしで同じことをすでにやっていないか。吸収済み rule は未使用 skill より緊急度が高い: 未使用 skill は無害に眠るが、吸収済み rule は進化する harness デフォルトを stale な内容で上書きし続ける。

## モード

| モード | トリガー | 動作 |
|------|---------|------|
| **full** | デフォルト、または `/rules-stocktake full` | 全 rule を読んで評価 |
| **changed** | `/rules-stocktake changed` | 前回実行以降に変更された rule のみ再評価し、残りは ledger から carry forward。機械的整合性チェックは常に全件実行 — 相互参照の壊れは rule の mtime に現れないため |

## 動作の仕組み

1. **Phase 1 — Inventory + 機械的整合性チェック**: Glob で `~/.claude/rules/**/*.md` を列挙し、全部を単一コンテキストに読み込み、ファイルごとの行数を計測。構造チェックは使い捨て grep で実行（enumerate/decide 分割）: `See skill:` 参照先の実在、extends リンクの解決、origin ヘッダの有無、rules README ツリーと実ファイル一覧の一致。
2. **Phase 2 — Evaluation**: 2 段 binary スクリーン — Stage 1 は rule ごとの 6 問 Yes/No チェックリスト（他 rule との重複 / skill・memory との重複 / 相互参照の解決 / 参照技術の現行性 / **substrate 未吸収** / **常駐に値する密度**）。Stage 2 は non-Keep 候補に対し、draft 判定を反証する rule 固有の質問を生成する。Dissolve 候補は吸収元の harness 機能を具体的に名指しできなければ反証される。binary 回答は holistic 判定の証拠であり、スコアには集約しない。
3. **Phase 3 — Summary**: `Rule | Lines | Verdict | Reason` テーブル。末尾に総行数と前回監査からの増減。
4. **Phase 4 — Consolidation**: 候補は **1 件ずつ**確認する — 各候補の証拠を提示してから `[y/n/skip]` を聞く。一括承認はせず、どの時点でも中断できる。承認された Improve/Update/Merge はセッション内で直接適用する（rule は小さく、rules 用の改善エンジン skill は存在しないため）。Demote の skill 作成は skill-creator にハンドオフ、Dissolve は理由の ADR 記録を提案する。

## 判定基準

| Verdict | 意味 |
|---------|------|
| **Keep** | 常駐に値する: 現行・一意・高密度 |
| **Improve** | 保持するが締める必要あり — rules では通常「短くする」 |
| **Update** | 参照技術が陳腐化 |
| **Merge into [X]** | 他 rule と実質重複 |
| **Demote to skill** | 有価値だが毎セッション常駐に値しない。確率トリガー層へ移動しポインタを残す |
| **Dissolve** | substrate（harness の native 化）または会話（内在化）に吸収済み。欠陥でなく**成功による退役** — stale shadow 化する前に削除し、理由を ADR に記録 |
| **Retire** | 欠陥ベースの削除: 低品質・陳腐・修復不能 |

## 要件

- **Glob**、**Read**、**Edit**、**Bash** ツールを持つ Claude Code（監査は単一メインコンテキストで実行 — subagent 不要）。
- オプション: changed モードのタイムスタンプ判定に `jq`。なくても gracefully degrade する。

## ハーネスからの同期

このスキルの正本は作者の生きた Claude Code harness にある。このリポジトリは一方向の公開ミラー:

```bash
scripts/sync-from-local.sh --dry-run   # 差分レポートのみ
scripts/sync-from-local.sh             # working tree に適用（commit はしない）
```

## References

2 段 binary 質問設計（screen → verdict pressure-test、holistic 判定、スコア集約なし)は checklist 分解評価の系譜に従う: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226)、CheckEval (arXiv:2403.18771)、TICK (arXiv:2410.03608)。

吸収質問と Dissolve verdict は Agent Knowledge Cycle の **Scaffold Dissolution** 概念の実装 — 2 つのベクトル（inward 内在化、downward substrate 吸収）は [docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md) に記録されている。

## このスキルについて

このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle)（Zenodo 引用可能な 6 phase 双方向成長ループ、[DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)）の **Curate** phase を rules 層へ拡張する: [skill-health](https://github.com/shimo4228/skill-health) が構造的負債、[skill-stocktake](https://github.com/shimo4228/skill-stocktake) が skill 品質、rules-stocktake が常時ロード rules を担う。監査対象の rules は [rules-distill](https://github.com/shimo4228/rules-distill)（Promote phase）が生産するもの — 同じ境界を逆方向に流れる関係にある。AKC は [@shimo4228](https://github.com/shimo4228) の 3 研究ラインの 1 つ（他は [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) と [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice)）。

## License

MIT
