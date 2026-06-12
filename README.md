# PR Review Toolkit for Codex

Codex CLI 向けの **観点固定 PR レビュー toolkit** です。汎用レビュー（`/review`）に加えて、正しさ・エラー処理・テスト・コメント・型設計・複雑さの 6 観点を一貫した手順で適用し、Finding を **P0–P3** に揃えて出力します。

| 解決すること | この toolkit の答え |
|--------------|---------------------|
| レビュー観点がぶれやすい | 6 つの固定観点 Skill + 統合 `pr-review` |
| 重要度の表記がバラバラ | P0–P3 に統一した Finding 形式 |
| 自動修正や環境改変が怖い | レビューのみ。install script・hook・sandbox bypass なし |

**デフォルトは 1 セッション内の順次適用**です。subagent 並列は任意機能で、未対応環境では自動的に順次モードへフォールバックします。

出力例は [docs/sample-review.md](./docs/sample-review.md)。出典・ライセンスは [EXTERNAL_SOURCES.md](./EXTERNAL_SOURCES.md) と [THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md) を参照してください。

Codex 標準の `/review` の代替ではなく、**観点固定レビューの追加レイヤー**として使います。レビュー観点は [Anthropic `pr-review-toolkit`](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit) を Codex Skill 向けに適応したものです。

## 含まれる Skill

| Skill | 役割 |
|-------|------|
| `pr-review` | 統合 Skill。全観点を適用し P0–P3 で Finding を統一出力 |
| `code-reviewer` | 正しさ・セキュリティ・保守性・規約準拠 |
| `silent-failure-hunter` | サイレント障害・エラー処理 |
| `pr-test-analyzer` | テスト網羅の質 |
| `comment-analyzer` | コメント・リポジトリ docs の事実確認 |
| `type-design-analyzer` | 型設計・不変条件 |
| `code-simplifier` | 複雑さ・明瞭さ（振る舞いは変えない） |

## 構成

```text
.agents/plugins/
  marketplace.json     # ローカルマーケットプレイスエントリ
.codex/
  config.toml          # subagent 設定（max_threads 等）
  agents/              # 観点ごとの Codex カスタムエージェント（SKILL.md から生成）
plugins/
  pr-review-toolkit/
    .codex-plugin/
      plugin.json      # プラグインマニフェスト（interface メタデータ含む）
    skills/
      pr-review/
        SKILL.md
        agents/openai.yaml
      code-reviewer/
        SKILL.md
        agents/openai.yaml
      ...（他 5 Skill も同様）
scripts/
  validate_plugin.py   # プラグイン検証スクリプト
  generate_subagents.py # SKILL.md → .codex/agents/*.toml 生成
docs/
  sample-review.md     # サンプルレビュー出力
LICENSE
README.md
```

### リポジトリ構成の約束

公開リポジトリとして、次の構成ルールを守っています。

| 項目 | 状態 |
|------|------|
| `install.sh` | **なし**。README の手順のみで導入 |
| `codex-autonomous` / `codex-account` / hook runner / MCP 自動設定 | **含めない**（レビュー Skill のみ） |
| `sandbox bypass` 前提 | **なし** |
| `.agents/plugins/marketplace.json` | `plugins[].source.path` は `./plugins/pr-review-toolkit` |
| `plugins/pr-review-toolkit/.codex-plugin/plugin.json` | 公開向け `interface` メタデータ（author / homepage / defaultPrompt 等） |

**編集元と生成物（二重管理しない）**

| 編集するファイル（正） | 生成物（手編集しない） |
|------------------------|------------------------|
| `plugins/pr-review-toolkit/skills/*/SKILL.md`（6 観点） | `.codex/agents/*.toml` |
| 同上 | `.codex/config.toml` |

`code-reviewer` など 6 観点の `SKILL.md` を変更したら、必ず `python3 scripts/generate_subagents.py` を実行してください。公開前は `python3 scripts/generate_subagents.py --check` で生成物が最新か確認できます。`pr-review` は orchestrator のため `.codex/agents/` には含めません。

## クイックスタート

```bash
git clone https://github.com/kyto64/codex-pr-review-toolkit-minimal.git
cd codex-pr-review-toolkit-minimal
codex plugin marketplace add .
codex plugin add pr-review-toolkit@kyto64-codex-pr-review-toolkit-minimal
```

Codex を再起動し、`$pr-review でこのブランチの差分をレビューしてください。ベースは main です。` のように Skill を呼び出します。詳細な導入方法は次の節を参照してください。

## 導入（install script なし）

このリポジトリは **install script を提供しません**。プラグインとして導入するか、必要な Skill だけをコピーしてください。

**前提:** [Codex CLI](https://developers.openai.com/codex) がインストール済みであること。PR 番号指定のレビューには `gh` CLI が必要です。

### 方法 A: プラグインとして使う（推奨）

リポジトリを clone し、マーケットプレイス経由でインストールする。

`codex plugin marketplace add` に渡すのは `marketplace.json` ファイルではなく、**marketplace root directory** です。このリポジトリでは clone 先のルート（`.`）を marketplace root として登録します。`.agents/plugins/marketplace.json` の `plugins[].source.path`（現在は `"./plugins/pr-review-toolkit"`）は、この marketplace root からの相対パスです。Codex は marketplace root 直下を plugin パスに指定できないため、plugin 本体は `plugins/pr-review-toolkit/` に配置しています。

```bash
git clone https://github.com/kyto64/codex-pr-review-toolkit-minimal.git
cd codex-pr-review-toolkit-minimal

# マーケットプレイスを登録（リポジトリルートで実行）
codex plugin marketplace add .

# 登録確認（kyto64-codex-pr-review-toolkit-minimal が表示されれば OK）
codex plugin marketplace list

# プラグインをインストール
codex plugin add pr-review-toolkit@kyto64-codex-pr-review-toolkit-minimal

# インストール確認（installed, enabled と表示されれば OK）
codex plugin list
```

`codex plugin list` で次のように表示されれば導入成功です。

```text
pr-review-toolkit@kyto64-codex-pr-review-toolkit-minimal  installed, enabled  0.1.0  .../plugins/pr-review-toolkit
```

Codex を再起動し、CLI では `/plugins` から `pr-review-toolkit` が表示されることを確認してください。

**統合レビュー Skill は `$pr-review` です。** プラグイン名は `pr-review-toolkit` であり、Skill 名とは異なります。初回利用時は `$pr-review` を明示してください。

Codex App では Composer のスタータープロンプトから `$pr-review` 等を選べます。`plugins/pr-review-toolkit/.codex-plugin/plugin.json` の `interface.defaultPrompt` に初期プロンプトが定義されています。

### 方法 B: Skill を個別にコピー

使いたい Skill だけ `$CODEX_HOME/skills`（通常 `~/.codex/skills`）へコピーする。

```bash
# 例: 統合レビューだけ使う
cp -r plugins/pr-review-toolkit/skills/pr-review ~/.codex/skills/

# 例: 全 Skill を入れる
cp -r plugins/pr-review-toolkit/skills/* ~/.codex/skills/
```

並列モードを使う場合のみ（任意）、`.codex/agents/` をプロジェクトルートに置くか `~/.codex/agents/` へコピーする（このリポジトリの `.codex/agents/` をそのまま使える）。`.codex/agents/` がなくても Skill 単体は順次モードで動作する。

### 方法 C: プロジェクトローカルに置く

他プロジェクトのリポジトリ内で Skill を参照する場合、Codex の標準パス `.agents/skills` に symlink する。

```bash
REPO=/path/to/codex-pr-review-toolkit-minimal
mkdir -p .agents/skills
for name in pr-review code-reviewer silent-failure-hunter pr-test-analyzer comment-analyzer type-design-analyzer code-simplifier; do
  ln -sfn "$REPO/plugins/pr-review-toolkit/skills/$name" ".agents/skills/$name"
done

# 並列モード用（任意）: リポジトリの .codex をそのまま使うか agents だけコピー
ln -sfn "$REPO/.codex/agents" ".codex/agents"
cp "$REPO/.codex/config.toml" ".codex/config.toml"
```

## 検証

プラグイン構成と subagent 定義が最新か確認する。

```bash
python3 scripts/validate_plugin.py plugins/pr-review-toolkit
python3 scripts/generate_subagents.py --check
```

`SKILL.md` を編集したあとは `python3 scripts/generate_subagents.py` で `.codex/agents/*.toml` を再生成する。

## 使い方

Codex CLI セッションで、レビューしたい PR やブランチのコンテキストを渡し、Skill を明示する。

### 入力として渡せるコンテキスト

| 入力 | 例 |
|------|-----|
| base branch（推奨） | 「ベースは main です」 |
| ローカル未コミット変更 | 「未コミットの変更をレビューして」 |
| PR 番号 / URL | 「PR #42 をレビューして」（`gh` CLI が必要） |
| ファイル指定 | 「`src/auth/` の変更だけ見て」 |

指定がない場合は `main...HEAD` の差分が対象になる。

### 統合レビュー（1コマンドで全観点）

統合レビューは **`$pr-review`** を使います（プラグイン名 `pr-review-toolkit` とは別です）。

ブランチ差分:

```
$pr-review でこのブランチの差分をレビューしてください。ベースは main です。
```

PR URL 指定（`gh` CLI が必要）:

```
$pr-review で https://github.com/OWNER/REPO/pull/NN をレビューしてください。
```

観点を絞る場合:

```
$pr-review ベースは main。エラー処理とテストの観点だけ見てください。
```

### 並列モード（任意 / Codex subagent）

**任意機能です。** デフォルトの `pr-review` は 1 セッション内で観点を順に適用します。並列化が必要なときだけ、次の条件を満たす環境で明示的に依頼してください。

- Codex の [subagent](https://developers.openai.com/codex/subagents) 機能が有効
- `.codex/agents/` に 6 観点の定義がある（このリポジトリを clone したプロジェクトルート、または `.codex/agents/` をコピーした先）

subagent が spawn されない・利用できない場合は、**自動的に順次モードへフォールバック**します。並列を使わない限り `.codex/agents/` の配置は不要です。

明示的に並列を頼む例:

```
$pr-review で main...HEAD をレビューしてください。適用対象の観点ごとに subagent を並列 spawn し、結果を P0–P3 に統合してください。
```

**トークンコストに注意**: 6 観点 × 独立コンテキストは単一セッションより高コスト。`pr-review` は変更内容に応じて起動する観点を絞る（ロジック変更がなければ `pr-test-analyzer` をスキップする等）。全観点の並列起動が必要なときだけ明示する。

README・LICENSE・NOTICE・attribution など docs-only の PR では、`pr-review` が `comment-analyzer` を docs factual review として適用し、ロジック変更がなければ `pr-test-analyzer` と `type-design-analyzer` をスキップしやすくなります。

### 個別観点だけ使う

```
$silent-failure-hunter で main...HEAD のエラー処理を重点的に見てください。

$pr-test-analyzer で PR #42 のテストが behavioral coverage として十分か見てください。

$type-design-analyzer でこの PR で追加した型の不変条件を見てください。
```

### 出力形式

`pr-review` は次の形式で Finding を出します（cosmetic 指摘は抑制）。

```markdown
## Findings

### P1: Silent failure in error handling

- File: `src/foo.ts`
- Lines: 42-49
- Category: silent-failure
- Problem:
- Impact:
- Suggested fix:
```

重要度:

- **P0**: マージ前に必須修正
- **P1**: 強く推奨
- **P2**: 改善推奨
- **P3**: 任意（通常は出力しない）

サンプル diff に対する出力例は [docs/sample-review.md](./docs/sample-review.md) を参照。

## Claude 版との差分・割り切り

原典（Claude Code の [`pr-review-toolkit`](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit)）と比べた、Codex Skill 版としての意図的な違い。

| 項目 | Claude 版（原典） | 本リポジトリ（Codex 版） |
|------|-------------------|--------------------------|
| 実行形態 | Claude agent / command | Codex Skill（`$pr-review` 等） |
| 並列レビュー | subagent を前提に設計 | **任意**。デフォルトは順次。未対応時はフォールバック |
| コード修正 | `code-simplifier` が直接書き換え可能 | レビュー専用（提案のみ） |
| モデル | agent ごとに opus / inherit 指定 | 呼び出し側の Codex セッションに従う |
| プロジェクト規約 | CLAUDE.md 固有ルールあり | 汎用化し `AGENTS.md` を参照 |
| 重要度表現 | 観点ごとに数値レーティング等 | P0–P3 に統一 |
| 導入 | Claude plugin として提供 | Codex plugin / Skill コピー。install script なし |
| 安全方針 | Claude 環境に依存 | sandbox bypass・hook 自動設定・MCP 自動設定を意図的に含めない |

並列モード利用時は、Finding の重複統合を親セッションが行います（subagent 間では去重できないため）。

## 意図的に含めないもの

PR レビュー用途に不要、またはリスク・運用負荷が高いため **削除済み** です。

- `codex-autonomous` などの autonomous 実行系
- `codex-account` などの auth 補助
- hook runner / autonomous subagent ランナー
- MCP 自動設定
- `install.sh` による広範囲な環境変更
- `--dangerously-bypass-approvals-and-sandbox` 前提の手順

## 出典

`code-reviewer`、`code-simplifier`、`comment-analyzer`、`pr-test-analyzer`、`silent-failure-hunter`、`type-design-analyzer` は [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) の `pr-review-toolkit` を Codex Skill 向けに適応したものです。`pr-review` およびプラグイン梱包・検証スクリプト等は本リポジトリ独自です。詳細は [EXTERNAL_SOURCES.md](./EXTERNAL_SOURCES.md) と [THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md) を参照してください。

## ライセンス

[Apache License 2.0](./LICENSE)。上記 6 観点 Skill は [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) の `pr-review-toolkit` を由来とする派生作品です。upstream に `NOTICE` はなく、本リポジトリの `LICENSE` は upstream と同一の Apache-2.0 全文です。帰属の区分は [THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md) を参照してください。

## コミュニティ

- 改善提案・不具合報告: [GitHub Issues](https://github.com/kyto64/codex-pr-review-toolkit-minimal/issues)
- 貢献手順: [CONTRIBUTING.md](./CONTRIBUTING.md)
- セキュリティ報告: [SECURITY.md](./SECURITY.md)

リリース履歴は GitHub Releases で管理します（初回公開タグは `v0.1.0` を予定）。
