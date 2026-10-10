Language: [English](README.md) | 日本語

# rules-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/rules-stocktake)

**常時ロードされる行動ルール**（`~/.claude/rules/`）を監査し、ルールのファイルごとに Keep・Improve・Demote to skill・Dissolve などの判定を出す Claude Code 向けの [Agent Skill](https://agentskills.io/specification)（英語）です。[skill-stocktake](https://github.com/shimo4228/skill-stocktake) の姉妹スキルですが、コストモデルが反転しています。スキルのコストはトリガー汚染（必要のない場面で呼ばれ、正しいスキルを選びにくくすること）ですが、ルールのコストは**常駐**です。`paths:` frontmatter を持たないルールファイルは、行動を変えるかどうかに関係なく、全行が全セッションにロードされるからです。

監査がルールを編集・降格・削除するのは、あなたがそのルールを 1 件ずつ承認した後だけです。

## インストール

インストールの方法は 2 つあります。このリポジトリを clone する方法（1 つ目のブロック）では rules-stocktake だけが入ります。akc-cycle プラグイン（2 つ目のブロック）では、ルールの内容を新しいスキルへ移すときに使う `skill-creator` と、ルールを削除した理由を記録するときに使う `adr-writer` も一緒に入ります（[判定基準](#判定基準)を参照）。この 2 つの手順をほかに何も入れずに動かしたいときは、プラグインを選んでください。

```bash
git clone https://github.com/shimo4228/rules-stocktake
mkdir -p ~/.claude/skills
cp -r rules-stocktake/skills/rules-stocktake ~/.claude/skills/rules-stocktake
```

そのあとは「ルールを棚卸しして」のように普通の言葉で頼むか、`/rules-stocktake` と入力します。

同じスキルは、Agent Knowledge Cycle（AKC: コーディングエージェントが繰り返した経験をスキルとルールに変える、人が承認する著者の 6 フェーズのサイクル）のほかのスキルと一緒に、Claude Code プラグイン [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）にも入っています。rules-stocktake はこのサイクルの Curate（整理）フェーズに属し、プラグインでは `/akc-cycle:rules-stocktake` という名前で呼びます。このリポジトリは同じ元から一方向に同期しているので、同期と同期の間はプラグインより古いことがあります。

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## 要件

- **Glob**、**Read**、**Edit**、**Write**、**Bash** ツールを持つ Claude Code（監査は単一のメインコンテキストで実行し、サブエージェントは不要です）。
- changed モードのタイムスタンプ判定に `jq`。
- Update の判定に使う現行性の確認（[判定基準](#判定基準)を参照）では、参照しているツールやフラグがまだ現行かを確かめるために Web を検索することがあります。
- Phase 1 の整合性チェックは著者の規約（1 行目の `origin` ヘッダ、`rules/common/` のファイルの `rationale:` と `review-when:` のコメント、`rules/README.md` の表）を前提にしているので、それらを持たないルールファイルは指摘として報告されます。この規約を使っていない環境では全ファイルに指摘が出て、Improve の候補になることがありますが、`n` で断れます。`rationale:` / `review-when:` のチェックは著者のハーネスにある `~/.claude/scripts/hooks/harness_lint.py` を実行しますが、このスクリプトはこのリポジトリには入っていません。スクリプトがない環境での代わりの動作は SKILL.md に定められていません。このスクリプトが確かめる条件は、両方のコメントがファイルの先頭 10 行以内にあることです。
- clone する方法では、Demote に `skill-creator` という名前のスキルのインストールが必要です（著者版は akc-cycle プラグインに入っています）。`adr-writer` は任意で、Dissolve の理由を ADR（アーキテクチャ決定記録）に記録するときだけ使います。

## 反転したコストモデル

ルールには使用回数の軸がありません。「ルールの呼び出し回数」を測るものはなく、`paths:` frontmatter を持たないルールファイルの無条件ロードにはその概念が成り立たないからです。監査はこれを 2 つの静的シグナルで置き換えます:

- **常駐密度**: 各行は毎セッション読ませる価値があるか。稀にしか要らない参照資料や長い手順書は、トリガーされたときだけ読み込まれるスキルへ降格します。
- **基盤による吸収**: 基盤、つまりルールが乗っているハーネス（システムプロンプト、ツールの説明、組み込みの計画・レビューの仕組み）が、ルールなしで同じことをすでにやっていないか、または原則が身についていて、ルールがなくても行動が変わらないか。吸収済みのルールは未使用のスキルより緊急度が高いです。未使用のスキルのコストは先に述べたトリガー汚染という受け身のものですが、吸収済みのルールはハーネスの新しいデフォルトを古い指示で能動的に上書きし続けます。

## モード

| モード | トリガー | 動作 |
|------|---------|------|
| **full** | デフォルト、または `/rules-stocktake full` | 全ルールを読んで評価 |
| **changed** | `/rules-stocktake changed` | 前回実行以降に変更されたルールのみ再評価し、残りは台帳（毎回の実行が書く `results.json`、Phase 4 を参照）から引き継ぐ。台帳は clone する方法では `~/.claude/skills/rules-stocktake/results.json`、プラグイン版は `${CLAUDE_PLUGIN_ROOT}/skills/rules-stocktake/results.json` を読む。相互参照の壊れはルールの mtime に現れないため、機械的整合性チェックは常に全件実行 |

## 動作の仕組み

1. **Phase 1 — Inventory + mechanical integrity checks**: Glob で `~/.claude/rules/**/*.md` を列挙し、全部を単一コンテキストに読み込み、ファイルごとの行数を計測します。構造チェックは使い捨て grep で実行し、モデルは結果の意味の判断だけを受け持ちます: ``skill: `name` `` ポインタが `~/.claude/skills/` の下のフォルダを指すか（プラグインで入れたスキルへのポインタは未解決として報告されます）、ルールファイル間の相対リンクが解決するか、全ルールファイルに `origin` ヘッダがあるか、`rules/common/` の全ファイルに `rationale:` と `review-when:` のコメントがあるか、ルールの README の表と実ファイル一覧が一致するか。
2. **Phase 2 — Evaluation**: Yes/No の質問による 2 段の選別です。Stage 1 はルールごとの 6 問の Yes/No チェックリスト（他ルールとの重複 / スキル・memory との重複 / 相互参照の解決 / 参照技術の現行性 / **基盤にまだ吸収されていない** / **常駐に値する密度**）です。Stage 2 は Keep 以外の暫定判定を、ルール固有の反証の質問で検証します（詳細は [SKILL.md](skills/rules-stocktake/SKILL.md)（英語））。
3. **Phase 3 — Summary**: `Rule | Lines | Verdict | Reason` の表を出し、末尾に総行数と前回監査からの増減を書きます。理由はそれだけで判断できるものでなければならず、スキルが手本として示す Demote の理由は次のとおりです（英語）: "109 lines of pytest fixture recipes; only the 80%-coverage principle changes per-session behavior. Keep 3 lines + pointer, move recipes to the language-specific skill."
4. **Phase 4 — Consolidation**: 候補は **1 件ずつ**確認します。各候補の証拠を提示してから `[y/n/skip]` を聞き、一括承認はせず、どの時点でも中断できます。承認された Improve/Update/Merge はセッション内で直接適用します。ルールファイルは直接編集できるほど短いからです。Demote は新しいスキルの作成を `skill-creator` という名前のスキルに引き渡し、ルールには短いポインタを残します。Dissolve は `adr-writer` で理由を ADR に記録することを提案します。ファイルを追加・改名・削除したときは、ルールの README の表をそれに合わせて更新します。どの実行も、スキップしたものを含む判定を台帳に書きます。

## 判定基準

| Verdict | 意味 |
|---------|------|
| **Keep** | 常駐に値する: 現行・一意・高密度 |
| **Improve** | 保持するが締める必要あり。ルールでは通常「短くする」 |
| **Update** | 参照技術が陳腐化 |
| **Merge into [X]** | 他ルールと実質重複 |
| **Demote to skill** | 価値はあるが毎セッションの常駐に値しない。スキルへ移し、ルールにはポインタを残す |
| **Dissolve** | 基盤（ハーネスが標準機能として取り込んだ）に吸収済み、または原則が身につき、ルールがなくても行動が変わらない。欠陥でなく**成功による退役**で、ハーネスの新しいデフォルトを上書きし始める前に削除し、理由を ADR に記録 |
| **Retire** | 欠陥ベースの削除: 低品質・陳腐・修復不能 |

## 参考研究

Yes/No の質問による 2 段の設計は、チェックリストによる評価の研究に基づいています（いずれも英語）: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226)、CheckEval (arXiv:2403.18771)、TICK (arXiv:2410.03608)。吸収の質問と Dissolve の判定は、Agent Knowledge Cycle の **Scaffold Dissolution** 概念（ルールやスキルは削除できたときに成功したとみなす考え方）の実装で、その説明は [docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md)（英語）にあります。

## 著者のほかの仕事

- **[Opus 5 世代でルールの書き方は公式に変わった——自作ルールの棚卸し手順](https://zenn.dev/shimo4228/articles/claude5-rules-official-shift-audit)**（[English](https://dev.to/shimo4228/opus-5-changed-how-rules-should-be-written-audit-yours-4fb4)）: 著者が常駐ルールを 1 つずつ新しいモデルに実際に読み込まれる指示と照らし、残す・直す・退役させるを決めた手順です。
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: このスキルをサイクルのほかのスキルと一緒に、1 つの Claude Code プラグインとして入れます（英語）。
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: Curate を含むサイクルの各フェーズがなぜあるのかを、日付付きの設計判断として記録しています。
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: インストール済みのスキルについての同種の監査で、古さ・矛盾・重複を見つけてスキルごとに判定を出します。
- **[agent-stocktake](https://github.com/shimo4228/agent-stocktake)**: サブエージェント定義についての同種の監査です。description は毎セッション常駐し、本文は呼ばれたときだけ読み込まれます。
- **[rules-distill](https://github.com/shimo4228/rules-distill)**: 逆向きの流れです。複数のスキルに繰り返し出てくる原則を見つけ、1 件ずつあなたの確認を取って常駐ルールにします（英語）。
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: 著者の拠点リポジトリです。AKC をほかの長期プロジェクトとその DOI と並べています。

## ライセンス

MIT

<details>
<summary>ツールと AI アシスタント向けの資料</summary>

rules-stocktake は、`~/.claude/rules/` の全ルールファイルを 1 つのコンテキストで読み、ファイルごとに判定を出す Claude Code 向けの Agent Skill です。自分の常駐ルールを保守していて、短く、現行で、モデルやハーネスがすでに自前でやっていることを含まない状態に保ちたい人のためのものです。判定は Keep・Improve・Update・Merge into [X]・Demote to skill・Dissolve・Retire の 7 つで、1 件ずつの `[y/n/skip]` の確認なしにルールファイルを編集・降格・削除することはありません。判定の台帳は毎回の実行で書きます。

存在する理由は、`paths:` frontmatter を持たないルールの代金が毎セッション払われることです。その 1 行ごとに毎セッションのトークンコストと指示の希釈が生じ、全体が長いほど個々のルールの効き目が弱まるので、総行数が増えるほど Keep の基準は上がります。`paths:` frontmatter でスコープされたルールファイルは、Claude が該当するファイルを扱うときだけロードされます。SKILL.md は `~/.claude/rules/` 以下の `.md` ファイルをすべて列挙して他と同じく行数を数え、パス指定のファイルを別扱いにはしません。ルールには使用回数のシグナルがないため、監査は代わりに 2 つの静的な問いを立てます。各行は常駐に値する密度か、そして基盤（ハーネス）がそのルールをすでに吸収していないか、あるいは原則が身についていてルールがなくても行動が変わらないか、です。吸収済みのルールは新しいデフォルトを上書きするので、必ず Dissolve の候補として挙げます。吸収元を具体的に名指しできない Dissolve 候補は反証されます。ルールごとの Yes/No チェックリストの回答は 1 つの総合判定の証拠であり、スコアには集約しません。これは AKC の Scaffold Dissolution のルール層での形で、ルールの成功とは削除できることです。

基本的な事実: MIT ライセンス。スキル本体（`skills/rules-stocktake/`）はスクリプトのない `SKILL.md` 1 枚です。著者 1 人（@shimo4228）が保守しています。状態: 稼働中で、著者の Claude Code ハーネスから `scripts/sync-from-local.sh`（`--dry-run` は差分の報告だけ。commit はしません）で一方向に同期しており、akc-cycle プラグインにも `/akc-cycle:rules-stocktake` として入っているため、同期と同期の間はこのリポジトリがプラグインより古いことがあります。要件: Glob・Read・Edit・Write・Bash を使える Claude Code、`changed` モード用の `jq`。有料の鍵は不要です。整合性チェックの 1 つ（`rationale:` / `review-when:` のコメント）は著者のハーネスにある `~/.claude/scripts/hooks/harness_lint.py` を実行しますが、このスクリプトはこのリポジトリには入っておらず、ない場合の代わりの動作は SKILL.md に定められていません。承認された Improve・Update・Merge の編集は自分で適用し、Demote は `skill-creator` という名前のスキルに引き渡し、Dissolve には `adr-writer` を提案し、ルールごとの行数と全体の合計を含む台帳 `results.json` を書きます（clone する方法では `~/.claude/skills/rules-stocktake/results.json`、プラグイン版は `${CLAUDE_PLUGIN_ROOT}/skills/rules-stocktake/` から読みます）。

例: 実行はまず見つかったファイル数、総行数、整合性の失敗を述べ、`Rule | Lines | Verdict | Reason` の表を出し、総行数と前回監査からの増減で締めます。Demote の理由は例えば "109 lines of pytest fixture recipes; only the 80%-coverage principle changes per-session behavior. Keep 3 lines + pointer, move recipes to the language-specific skill." のように書かれます。

リンク: [skills/rules-stocktake/SKILL.md](skills/rules-stocktake/SKILL.md)（英語）がスキル本体、[llms.txt](llms.txt) と [llms-full.txt](llms-full.txt)（英語）が機械可読の要約と参照資料、AKC の [docs/scaffold-dissolution.md](https://github.com/shimo4228/agent-knowledge-cycle/blob/main/docs/scaffold-dissolution.md)（英語）が吸収の 2 つのベクトルの説明です。このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の Curate フェーズをルール層へ広げるもので、AKC の concept DOI（常に最新版へつながる代表 DOI）は [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726) です。引用はこの DOI で行ってください。サイクル全体をインストールできる形は [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）です。

</details>
