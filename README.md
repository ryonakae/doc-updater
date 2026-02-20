# doc-updater セットアップガイド

コード変更時にプロジェクトドキュメント（AGENTS.md, CLAUDE.md, README.md等）を
自動更新するための構成一式です。

## 構成

```
.claude/
├── agents/
│   └── doc-updater.md                  # ドキュメント更新サブエージェント
└── settings.json                       # Stopフック（prompt型ゲート付き）

git-hooks/
└── pre-commit                          # git pre-commitフック
```

## トリガーの使い分け

| トリガー | 対象 | タイミング |
|----------|------|-----------|
| Claude Code Stopフック | エージェントによる実装完了時 + セッション中の失敗・試行錯誤からの学習 | Claudeが実装完了を報告するたび（自動） |
| git pre-commitフック | 人間の手動コミット / エージェントコードの修正後コミット | `git commit` 実行時（自動） |

### 検出対象の変更

doc-updaterは以下の変更を検出し、ドキュメントへの反映が必要か判断します:

- **コード変更**: 機能追加、API変更、エージェント・スキル・フックの変更
- **依存関係の変更**: package.json, requirements.txt, Cargo.toml 等のライブラリ追加・削除・変更
- **ファイルのリネーム・削除**: ドキュメント内のパス参照を自動更新・除去
- **コマンド・スクリプトの変更**: Makefile, justfile, npm scripts 等のビルド・テスト・実行コマンド変更
- **設定ファイルの変更**: .env.example, docker-compose.yml, tsconfig.json 等の開発環境設定

これにより以下のすべてのケースに対応できます:
- Claude Codeが実装を完了した場合（コード変更の反映 + セッション中の失敗・試行錯誤の学習）
- 人間が手でコードを書いてコミットする場合
- エージェント生成コードを人間が修正してからコミットする場合

## 処理の流れ

### Stopフック（実装完了 → セッション振り返り + ドキュメント更新）

```
メインClaude が実装完了を報告 → 停止しようとする
    │
    ▼
Stop フック（prompt型ゲート）が判定
    │
    ├─ 実装完了でない場合 → そのまま停止
    └─ 実装完了の場合 → {ok: false} で続行指示
           │
           ▼
       メインClaude（セッション履歴が見える）
           │
           ├─ 1. セッション振り返り
           │     失敗・修正・試行錯誤があれば
           │     内容を整理し、サブエージェントの指示に埋め込む
           │
           ├─ 2. doc-updater サブエージェント呼び出し（完了を待つ）
           │     ├─ エージェント指示ファイルの構成を検出
           │     ├─ git diff で変更内容を分析
           │     ├─ 必要に応じてドキュメントを更新
           │     ├─ プロンプトにセッション学習が含まれていれば防止策を反映
           │     └─ サマリーを返す
           │
           └─ 3. 停止
```

### git pre-commit（コード変更 → ドキュメント更新）

```
ステージされたファイルの変更を検出
    │
    ▼
doc-updater サブエージェント（別コンテキスト）
    │
    ├─ エージェント指示ファイルの構成を検出（実ファイル / symlink）
    ├─ git diff --cached で変更内容を分析
    ├─ 影響判定（ドキュメント更新が必要か LLM が判断）
    ├─ 必要に応じてドキュメントを更新
    └─ サマリーを返す
```

## セットアップ

### 1. ファイルを配置

```bash
cp -r .claude/ /path/to/your-project/.claude/
# 既存の .claude/settings.json がある場合は hooks セクションだけマージ
```

### 2. git pre-commitフックを設定

```bash
cp git-hooks/pre-commit /path/to/your-project/.git/hooks/pre-commit
chmod +x /path/to/your-project/.git/hooks/pre-commit
```

### 3. プロジェクト固有の調整

doc-updater.md 内の記述規約セクションはテンプレートです。
プロジェクトのドキュメント構造に合わせてカスタマイズしてください:

- README.mdのセクション構成
- AGENTS.md / CLAUDE.mdの既存構造
- docs/ 配下のドキュメント体系

## 注意点

- **Stopフックのコスト**: prompt型ゲートは毎回LLM呼び出しが発生します。
  コストが気になる場合はgit pre-commitフックのみの運用も可能です。
- **pre-commitの実行時間**: claude CLIの起動＋分析で数十秒かかります。
  `--no-verify` でスキップ可能です。
- **再帰防止**: 環境変数 `DOC_UPDATER_RUNNING` でガードしています。
