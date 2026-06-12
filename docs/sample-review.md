# サンプル PR レビュー出力

`$pr-review` の出力イメージ。以下のサンプル diff（意図的に問題を含む）に対して、各 SKILL.md の手順に従って生成したもの。

## サンプル diff

```diff
--- a/src/user-service.ts
+++ b/src/user-service.ts
@@ -10,6 +10,22 @@
+// Returns the user's display name, falling back to email
+export async function getDisplayName(userId: string): Promise<string> {
+  try {
+    const user = await fetchUser(userId);
+    return user.profile.displayName;
+  } catch (e) {
+    return "";
+  }
+}
+
+export type UserProfile = {
+  displayName: string;
+  email: string;
+  // must be set before displayName is read
+  initialized: boolean;
+};
```

テスト追加なし。

## レビュー出力

```markdown
## PR Review

Reviewed: main...HEAD (1 file), perspectives: all (comment-analyzer applied — comments changed)

## Findings

### P0: Silent failure swallows all errors in getDisplayName

- File: `src/user-service.ts`
- Lines: 12-18
- Category: silent-failure
- Problem: catch がすべての例外を握りつぶし、空文字を返して続行する。ログも通知もない。
- Hidden errors: ネットワーク障害、認証失効、`user.profile` が undefined の TypeError、fetchUser のバグ — すべて同じ空文字に潰れる。
- Impact: ユーザー名が無言で空表示になり、原因（一時障害か恒久バグか）の切り分けが不可能になる。
- Suggested fix: 想定するエラー型のみ catch してログ＋上位へ伝播するか、`Result` を返して呼び出し側に失敗を見せる。空文字フォールバックを残すなら、その仕様根拠と warn ログを必須にする。

### P1: New error path and fallback have no tests

- File: `src/user-service.ts` (no corresponding test file)
- Lines: 12-18
- Category: test-coverage
- Problem: fetchUser 失敗時・profile 欠落時の挙動がテストされていない。
- Impact: 将来 catch の挙動を変えた際、フォールバックの回帰（例: 例外が漏れて画面クラッシュ）を検出できない。
- Suggested test: fetchUser をエラーにし、(1) 期待するフォールバック値が返ること、(2) エラーがログ／伝播されることを検証する。

### P1: Comment contradicts implementation

- File: `src/user-service.ts`
- Lines: 11
- Category: comments
- Problem: 「falling back to email」とあるが、実装は email ではなく空文字を返す。
- Impact: 保守者がコメントを信じて「email が出るはず」と誤った修正・デバッグをする。
- Suggested fix: 実装に合わせて書き換えるか、コメント通り email フォールバックを実装する。

### P2: UserProfile invariant enforced only by comment

- File: `src/user-service.ts`
- Lines: 21-26
- Category: type-design
- Invariants: `initialized` が true になるまで `displayName` を読んではならない。
- Problem: 不変条件（強制: 弱い）がコメントのみで、型としては未初期化のまま displayName を読める。
- Impact: 初期化忘れがコンパイル時にも実行時にも検出されない。
- Suggested fix: `UninitializedProfile` / `UserProfile` の2型に分けるか、初期化を保証するコンストラクタ関数経由でのみ生成できるようにする。

## Notes

- 変更が1関数＋1型に収まっており、レビューしやすい粒度。
```
