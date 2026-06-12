---
name: pr-review
description: Use when reviewing a pull request or branch diff with fixed perspectives and unified findings. Orchestrates code-reviewer, silent-failure-hunter, pr-test-analyzer, comment-analyzer, type-design-analyzer, and code-simplifier viewpoints.
---

# PR Review (Unified)

Codex 標準の `/review` の代替ではなく、**観点固定の追加レビューレイヤー**として使う統合 Skill です。PR やブランチ差分を、複数の専門観点でレビューし、重要度を揃えた Finding として出力します。

## 位置づけ

- Codex 標準レビューに加えて、特定観点（エラー処理、テスト、コメント、型設計など）を深掘りする
- 自動修正・GitHub Action・MCP 連携は行わない
- sandbox bypass や承認スキップを前提にしない

## 実行モード

Codex subagent（`.codex/agents/` のカスタムエージェント）が利用可能な場合は**並列モード**を優先する。利用できない、または spawn されない場合は**順次モード**にフォールバックする。

| モード | いつ | 動き |
|--------|------|------|
| **並列** | `.codex/agents/` に観点エージェントがあり、親が明示的に spawn できる | 適用対象の観点ごとに subagent を並列起動し、結果を統合 |
| **順次** | subagent 非対応、spawn 失敗、ユーザーが単一セッションを指定 | 1 セッション内で観点を順に適用 |

並列モードでは親セッションがオーケストレーターとして動く。subagent 間で Finding の重複排除はできないため、統合・去重は親が行う。

## レビュー手順

### 1. スコープを確定する

ユーザーから受け取るコンテキストは次のいずれか。指定がなければ `main...HEAD` を使う。

| 入力 | 取得方法 |
|------|----------|
| base branch（推奨） | `git diff <base>...HEAD` |
| ローカル未コミット変更 | `git diff` |
| PR 番号 / URL | `gh pr diff <n>`、`gh pr view <n>` |
| ファイル指定 | 指定ファイルの差分のみ |

```bash
git diff main...HEAD
git diff --name-only main...HEAD
```

レビュー開始時に、対象（base、ブランチ、変更ファイル数）を1行で明示する。プロジェクトに `AGENTS.md` 等の規約があれば先に読む。

### 2. 適用する観点を決める

ユーザーが観点を指定した場合（例: 「エラー処理とテストだけ」）はそれに従う。指定がなければ、変更内容に応じて適用する。

| 観点 | Skill | Category | 適用条件 |
|------|-------|----------|----------|
| 正しさ・セキュリティ・保守性 | `code-reviewer` | `correctness`, `security`, `maintainability` | 常に適用 |
| サイレント障害・エラー処理 | `silent-failure-hunter` | `silent-failure` | 常に適用 |
| 複雑さ・簡潔さ | `code-simplifier` | `simplicity` | 常に適用 |
| テスト網羅 | `pr-test-analyzer` | `test-coverage` | ロジック変更がある場合（テスト追加の有無に関わらず）。docs-only / metadata-only でロジック変更がなければスキップ |
| コメント・docs の事実確認 | `comment-analyzer` | `comments` | コードコメントの追加・変更、または README / docs / LICENSE / NOTICE / attribution / SECURITY / CONTRIBUTING 等のリポジトリ docs 変更時 |
| 型設計 | `type-design-analyzer` | `type-design` | 型・データモデルが追加・変更された場合。型やデータモデル変更がなければスキップ |

各 Skill の詳細手順は `skills/<name>/SKILL.md` を参照。スキップした観点は最終出力で「適用外（理由）」として1行報告する。

### 外部出典・ライセンス・attribution の確認

外部 repo、license、NOTICE、upstream path、third-party notice を主張する変更では、可能なら一次ソースで確認する。

- 確認できる場合: 矛盾があれば Finding にする
- ネットワークや権限の都合で確認できない場合: 不確実な推測を Finding にしない。Notes に「未検証: [項目]」として残す
- legal / attribution correctness を推測で断定しない

### プラグイン・Skill 変更時の検証

レビュー対象に plugin manifest、Skill、`AGENTS.md` の validation command が関係する場合、可能なら実行し、結果を Notes に出す。

```bash
python3 scripts/validate_plugin.py plugins/pr-review-toolkit
python3 scripts/generate_subagents.py --check
```

実行できない環境では、未実行であることを Notes に残す。

### 3. 観点を適用する

#### 並列モード（subagent 利用時）

Codex は「明示的に頼んだときだけ」subagent を spawn する。`pr-review` 実行時は、適用対象の各観点について**明示的に spawn を依頼する**。

1. 適用対象の観点とスコープ（base、変更ファイル一覧、差分の要約）を整理する
2. 次の形式で spawn を依頼する（例）:

```text
以下の観点ごとに subagent を1つずつ並列 spawn し、すべての結果が揃うまで待ってから統合してください。
スコープ: main...HEAD（12 files）。プロジェクト規約は AGENTS.md を参照。

1. code-reviewer — 正しさ・セキュリティ・保守性
2. silent-failure-hunter — サイレント障害・エラー処理
3. code-simplifier — 複雑さ・簡潔さ
（条件付き観点があればここに追加。例: pr-test-analyzer — ロジック変更あり）
```

3. 各 subagent には同じスコープ情報を渡す。エージェント名は `.codex/agents/<skill-name>.toml` の `name` と一致する（`code-reviewer`、`silent-failure-hunter` 等）
4. 全 subagent の結果を受け取ったら、次の「統合」手順へ進む
5. spawn されなかった観点がある場合は、その観点だけ順次モードで補完する

**トークンコスト**: 6 観点すべてを常に並列起動すると高コストになる。上記の適用条件（#13）で起動数を絞ること。ユーザーが「全観点を並列で」と指定した場合のみ全件 spawn する。

**sandbox**: subagent の `sandbox_mode` は設定しない（親セッションを継承）。bypass 前提にしない。

#### 順次モード（フォールバック）

並列モードが使えない場合、または補完が必要な場合は、適用対象の各観点をこのセッション内で順に適用する。各観点の手順は `skills/<name>/SKILL.md` に従う。

#### 統合（並列・順次共通）

- 各観点の Finding を P0–P3 に揃える
- 複数観点が同じ根本原因を指摘した場合、最も重要度の高い 1 件に統合し、Category は主観点を使う
- 信頼度の低い指摘は除外する

### 4. 重要度を統一する

すべての Finding は **P0 / P1 / P2 / P3** で分類する。

| 重要度 | 意味 | 例 |
|--------|------|-----|
| **P0** | マージ前に必ず修正。データ損失、セキュリティ、サイレント障害 | 空の catch、認可バイパス |
| **P1** | 強く推奨。本番障害や回帰のリスク | 重要パスのテスト欠落、誤解を招くエラーメッセージ |
| **P2** | 改善推奨。品質・保守性の問題 | 型の不変条件が弱い、冗長な複雑さ |
| **P3** | 任意。低影響の改善 | 軽微な命名、スタイルの統一 |

**cosmetic な指摘（空白、微細なフォーマット、好みのスタイル）は出力しない。** P3 もデフォルトでは出力しない（ユーザーが求めた場合のみ）。

### 5. Finding の品質基準

各 Finding は次を満たすこと。

- **具体的**: ファイルパスと行番号を示す
- **修正可能**: 提案が実装可能な形で書かれている
- **影響が明確**: 何が壊れるか、なぜ重要かを説明する
- **重複排除**: 複数の観点が同じ根本原因を指摘した場合、最も重要度の高い1件に統合し、Category は主観点を使う

信頼度が低い指摘（推測・好み・この PR 以前から存在する既知問題）は除外する。

## 出力形式

次の形式で出力する。観点ごとの長文レポートは不要。**Finding がないのに無理に作らない。**

```markdown
## PR Review

Reviewed: main...HEAD (12 files), mode: parallel, applied: code-reviewer, silent-failure-hunter, code-simplifier, comment-analyzer; skipped: pr-test-analyzer (docs-only), type-design-analyzer (no type changes)

## Findings

### P1: [短いタイトル]

- File: `path/to/file.ts`
- Lines: 42-49
- Category: silent-failure
- Problem: [何が問題か]
- Impact: [ユーザー・運用への影響]
- Suggested fix: [具体的な修正案]

### P2: [短いタイトル]

- File: `path/to/file.test.ts`
- Lines: 10-25
- Category: test-coverage
- Problem: [何がテストされていないか]
- Impact: [どんな回帰を見逃すか]
- Suggested test: [追加すべきテストの内容]

## Notes

- [よくできている点があれば 1–3 行。なければ省略]
- Validation: `validate_plugin.py` passed; `generate_subagents.py --check` passed
- 未検証: upstream LICENSE byte-for-byte 一致（ネットワーク未確認）
```

Finding がない場合:

```markdown
## Findings

No actionable findings. The change set looks ready from the configured review perspectives.
```

## ルール

- DO: 変更されたファイルを実際に読む
- DO: 適用対象の専門観点を漏れなく適用し、スキップした観点は理由を報告する
- DO: P0/P1 を先に、同重要度内はファイル順で並べる
- DO: プロジェクトの `AGENTS.md` や既存規約があれば尊重する
- DO NOT: コードを自動修正しない（レビューのみ）
- DO NOT: cosmetic 指摘を出す
- DO NOT: 根拠のない推測を Finding にする
- DO NOT: sandbox bypass や承認スキップを前提にする
