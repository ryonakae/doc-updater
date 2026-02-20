# Plan: reflections.md 廃止 → プロンプト直接埋め込み方式

## 概要
メインエージェントがセッション振り返りをファイル(`.claude/reflections.md`)経由ではなく、サブエージェントへの呼び出し指示に直接埋め込むようにする。

## Steps

### Phase 1: settings.json の reason プロンプト修正
1. `src/.claude/settings.json` の `reason` 内のプロンプトを変更:
   - ステップ1: 振り返りを「ファイルに書き出す」→「整理だけして、次のステップでサブエージェントに直接渡す」
   - ステップ2: サブエージェントへの指示に振り返り内容を直接含めるよう指示（振り返りがない場合・ある場合の2パターン明示）
   - `.claude/reflections.md` への言及をすべて削除

新しい reason の内容:
```
実装完了を検出しました。停止前に以下を順番に実行してください。

1. セッション振り返り: このセッション中に失敗・修正・試行錯誤があった場合、以下を整理してください（ファイルには書き出さず、次のステップでサブエージェントに直接渡します）。なければスキップ。
   - 失敗パターン: エージェントが間違えた箇所、人間に修正された箇所
   - 根本原因: なぜ間違えたか
   - 防止策: エージェント指示ファイルに追記すべきルール（事実形式で記述）

2. doc-updater 呼び出し: doc-updater サブエージェント（モデルは必ず haiku）を呼び出してください。run_in_background は使わず、完了を待ってください。
   - 基本指示: 『git diff HEAD で変更内容を確認し、必要に応じてドキュメントを更新してください。』
   - ステップ1で振り返り内容がある場合: 上記の基本指示に加えて、振り返りの「防止策」をサブエージェントへの指示テキストに直接追記してください。形式例: 『...ドキュメントを更新してください。また、以下のセッション学習をエージェント指示ファイルに反映してください:\n（防止策の内容をここに埋め込む）』

3. doc-updater の完了を確認してから停止してください。
```

### Phase 2: doc-updater.md のステップ2修正
2. `src/.claude/agents/doc-updater.md` のステップ2「変更の特定」セクション:
   - 旧: 「セッション学習経由: `.claude/reflections.md` を読み、防止策を反映する。反映後は `.claude/reflections.md` を削除する」
   - 新: 「セッション学習経由: 呼び出しプロンプトにセッション学習（防止策）が含まれている場合、その内容をエージェント指示ファイルに反映する」

3. 同 ステップ3「影響判定」セクション:
   - 旧: 「`.claude/reflections.md` が存在する → 防止策の反映が必要」
   - 新: 「呼び出しプロンプトにセッション学習が含まれている → 防止策の反映が必要」

### Phase 3: README.md の更新
4. `README.md` のフロー図・説明文:
   - 「.claude/reflections.md に書き出し」→「振り返りを整理し、サブエージェントの指示に直接埋め込む」
   - 「reflections.md があれば防止策を反映し削除」→「プロンプトにセッション学習が含まれていれば防止策を反映」

## Relevant files
- `src/.claude/settings.json` — reason フィールドのプロンプト書き換え
- `src/.claude/agents/doc-updater.md` — ステップ2とステップ3の reflections.md 関連記述を変更
- `README.md` — フロー図・説明文の reflections.md 関連記述を変更

## Verification
1. settings.json が valid JSON であること（`jq . src/.claude/settings.json`）
2. reason プロンプトを展開して読み、指示が論理的に一貫していることを目視確認
3. doc-updater.md 内に `reflections.md` への言及が残っていないことを grep 確認
4. README.md 内の reflections.md フロー記述が残っていないことを確認

## Decisions
- `.claude/reflections.md` ファイル自体のテンプレートや参照はすべて削除
- pre-commit フック（`src/git-hooks/pre-commit`）は変更不要（reflections.md に関する処理がない）
