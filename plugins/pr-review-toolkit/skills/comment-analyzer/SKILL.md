---
name: comment-analyzer
description: Use when reviewing changed code comments and repository docs (README, license, NOTICE, attribution, security policy) for factual accuracy, completeness, and long-term maintainability.
---

# Comment Analyzer

PR で追加・変更された**コードコメント**および**リポジトリ docs**（README、LICENSE、NOTICE、attribution、SECURITY、CONTRIBUTING 等）の**事実の正確性と長期保守性**をレビューする。数ヶ月後にコンテキストなしでコードや docs を読む保守者の視点で、コメント rot や docs の陳腐化による技術的負債を防ぐ。`pr-review` の `comments` 観点として使う。

## レビュー観点

### 1. コードコメントの事実の正確性（実装と突き合わせて検証する）

- パラメータ・戻り値の記述がシグネチャと一致しているか
- 記述された振る舞いが実際のロジックと一致しているか
- 参照している型・関数・変数が実在するか
- 言及されたエッジケースが実際にコードで処理されているか
- 計算量・パフォーマンスの主張が正確か

### 2. リポジトリ docs の事実の正確性（リポジトリ実態と突き合わせて検証する）

- README のコマンド例・手順が実際の CLI やスクリプトと一致するか
- パス、ファイル名、generated/source-of-truth の説明がリポジトリ実態と一致するか
- license、NOTICE、upstream path、third-party attribution の主張が一次ソースと一致するか
- security / support / contribution policy がリポジトリの実態と矛盾しないか

### 3. 完全性

- 非自明な前提・事前条件が書かれているか
- 副作用、重要なエラー条件が説明されているか
- 複雑なアルゴリズムのアプローチ、自明でないビジネス上の理由が説明されているか

### 4. 長期価値

- 「なぜ」を説明しているか。自明な「何」の繰り返しは削除候補
- 近い将来のコード変更で陳腐化しやすい記述（一時的状態への言及など）がないか

### 5. 誤解の余地

- 複数に解釈できる曖昧な表現
- リファクタ済みコードへの古い参照、現実装と一致しない例
- すでに解決済みの TODO / FIXME

## 外部出典・ライセンス・attribution の確認

外部 repo、license、NOTICE、upstream path、third-party notice を主張する変更では、可能なら一次ソースで確認する。

- upstream の agent path が attribution table と一致するか
- upstream に NOTICE があるか／ないかの主張が実態と一致するか
- local LICENSE が upstream plugin LICENSE と byte-for-byte で一致するか（主張がある場合）

**確認できない場合**: 不確実な推測を Finding にしない。Notes に「未検証: [項目]」として残す。legal / attribution correctness を推測で断定しない。

## 重要度の目安

| 状況 | 重要度 |
|------|--------|
| 誤ったコメントや docs が誤った修正・運用を誘発する | P1 |
| 重要な前提が欠けている、陳腐化しやすい記述 | P2 |
| 削除推奨の無価値コメント | P3（pr-review では通常除外） |

## 出力

Finding がある場合、または Finding はないが未検証の外部出典など Notes を残す必要がある場合は、次の形式を使う。観点ごとの長文レポートは不要。

```markdown
## Findings

### P1: [タイトル]
- File: `path`
- Lines: N-M
- Category: comments
- Problem: [不正確・誤解を招く点。コードや docs のどこと矛盾するか]
- Impact: [将来の保守者への影響]
- Suggested fix: [具体的な書き換え案または削除理由]
```

Finding がなく Notes のみある場合:

```markdown
## Notes

- 未検証: [項目]（確認不能な理由）
```

Finding と Notes の両方がある場合は Findings を先に、続けて Notes を返す。

Finding も Notes もない場合は `No findings from comment-analyzer perspective.` と1行で返す。

**レビューのみ**。コメントや docs の直接編集は行わない。

## ルール

- DO: コメントや docs の主張を実装・リポジトリ実態・一次ソースと突き合わせてから指摘する
- DO: 具体的な書き換え案を示す
- DO: 良い手本になるコメントや docs は短く肯定的に言及してよい
- DO: 外部確認不能な項目は Notes に未検証として残す
- DO NOT: 自明なコメント削除を大量に列挙する
- DO NOT: 自明な文章校正や好みの表現を指摘する
- DO NOT: 変更スコープ外の docs 全体監査に広げる
- DO NOT: 確認不能な外部出典・ライセンス主張を推測で Finding にする
