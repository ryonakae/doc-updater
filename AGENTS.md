# doc-updater

コード変更時にプロジェクトドキュメント（AGENTS.md, CLAUDE.md, README.md等）を自動更新する構成一式のテンプレートリポジトリ。`src/` 配下のファイルを他のプロジェクトに導入して使う。

## プロジェクト概要

### プロジェクト構成

```
doc-updater/                              # リポジトリルート
├── AGENTS.md                             # 実ファイル（本ドキュメント）
├── CLAUDE.md -> AGENTS.md                # symlink
├── GEMINI.md -> AGENTS.md                # symlink
├── README.md -> AGENTS.md                # symlink
└── src/                                  # 配布用ファイル群（他プロジェクトに導入する）
    ├── .claude/
    │   ├── agents/
    │   │   └── doc-updater.md            # ドキュメント更新サブエージェント
    │   ├── settings.json                 # Stopフック（agent型ゲート付き）
    │   └── skills/
    │       └── commit-push/
    │           └── SKILL.md              # コミット&pushスキル
    └── git-hooks/
        └── pre-commit                    # git pre-commitフック
```

### テックスタック

- Claude Code（Hooks / Sub-agents / Skills）
- シェルスクリプト（`src/git-hooks/pre-commit`）
- LLMモデル: claude-sonnet-4-6（Stopフック判定用）、haiku（doc-updater用）

## 設計方針

### 二重トリガー（Stopフック + pre-commit）

| トリガー | カバーする対象 |
|----------|--------------|
| Stopフック | エージェントによる実装完了時。セッション文脈（失敗・試行錯誤の学習）も活用できる |
| pre-commitフック | 人間の手動変更、エージェント生成コードの修正後コミット |

両方あることで「いつコードが変更されても必ずドキュメントが追従する」状態を保証する。

### LLM判定ゲート

Stopフック判定にagent型ゲートを使い、会話的なやり取りや調査報告では発火しない。以下の2条件を両方満たす場合のみ続行指示:
1. 実装完了：コード変更を伴うタスクが完了し、成果が報告されている
2. コード変更あり：ファイルの作成・編集・削除が行われている

**無限ループ防止**: `stop_hook_active` は同一 Stop イベント内のリエントランス防止フラグであり、次のターンの Stop イベントではリセットされる。そのため、doc-updater 実行後の2回目の Stop イベントでも `stop_hook_active: false` になり再発火しうる。これを防ぐため、`last_assistant_message` に doc-updater の実行・完了に関する言及（「doc-updater」「ドキュメント更新」「AGENTS.md」等）がある場合を例外として ok=true を返す。

git操作のみ・相談調査・途中経過・判断がつかない場合はすべてok=true（発動しない）。タイムアウトは180秒で設定。

### モデル選定

Stopフック判定には claude-sonnet-4-6 を使い、セッション履歴の分析精度を確保する。doc-updater サブエージェントは「差分の分析 + 必要最小限の編集」であり高度な推論は不要なため haiku を使い、コストを最小化する。

### symlink戦略

AGENTS.md を実ファイルとして管理し、CLAUDE.md / GEMINI.md / README.md はすべてAGENTS.mdへのsymlinkとする。1つのファイルを編集するだけで全AIエージェントとGitHubのREADMEに反映される。

## 開発・検証

```bash
# settings.json のJSONバリデーション
jq . src/.claude/settings.json

# pre-commit フックの構文チェック
bash -n src/git-hooks/pre-commit

# symlink の整合性確認
ls -la CLAUDE.md GEMINI.md README.md
```

## 作業ルール

- **`src/` 配下**: 配布用テンプレート。他プロジェクトに導入される内容
- **ルート直下**: このリポジトリ自体の管理。AGENTS.md のみ実ファイルとして編集する
- **symlink（CLAUDE.md, GEMINI.md, README.md）は直接編集しない**。AGENTS.md への変更が自動反映される

各ファイルの役割:

| ファイル | 役割 |
|---------|------|
| `src/.claude/agents/doc-updater.md` | doc-updaterサブエージェントの定義。記述規約の詳細を含む |
| `src/.claude/settings.json` | Stopフック設定。変更後は `jq .` でバリデーション |
| `src/.claude/skills/commit-push/SKILL.md` | Conventional Commitsメッセージ自動生成 + doc-updater連携スキル |
| `src/git-hooks/pre-commit` | bashスクリプト。変更後は `bash -n` で構文チェック |

## トリガーの使い分け

| トリガー | 対象 | タイミング |
|----------|------|-----------|
| Claude Code Stopフック | エージェントによる実装完了時 + セッション中の失敗・試行錯誤からの学習 | Claudeが実装完了を報告するたび（自動） |
| git pre-commitフック | 人間の手動コミット / エージェントコードの修正後コミット | `git commit` 実行時（自動） |

### 検出対象の変更

doc-updaterは以下の変更を検出し、ドキュメントへの反映が必要か判断する:

- **コード変更**: 機能追加、API変更、エージェント・スキル・フックの変更
- **依存関係の変更**: package.json, requirements.txt, Cargo.toml 等のライブラリ追加・削除・変更
- **ファイルのリネーム・削除**: ドキュメント内のパス参照を自動更新・除去
- **コマンド・スクリプトの変更**: Makefile, justfile, npm scripts 等のビルド・テスト・実行コマンド変更
- **設定ファイルの変更**: .env.example, docker-compose.yml, tsconfig.json 等の開発環境設定

## 処理の流れ

### Stopフック（実装完了 → ドキュメント更新）

```
メインClaude が実装完了を報告 → 停止しようとする
    │
    ▼
Stop フック（agent型ゲート）が判定
    │
    ├─ 実装完了 かつ コード変更あり → {ok: false} で続行指示
    │        │
    │        ▼
    │    メインClaude（セッション履歴が見える）
    │        │
    │        └─ doc-updater サブエージェントを実行
    │              ├─ エージェント指示ファイルの構成を検出
    │              ├─ 対象ファイル群の変更内容を分析
    │              ├─ 必要に応じてドキュメントを更新
    │              ├─ セッション中の失敗・修正から防止策があれば反映
    │              └─ サマリーを返す
    │
    └─ それ以外（git操作のみ・相談調査・途中経過等）→ そのまま停止
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

## セットアップ（利用者向け）

### 1. ファイルを配置

```bash
cp -r src/.claude/ /path/to/your-project/.claude/
# 既存の .claude/settings.json がある場合は hooks セクションだけマージ
```

### 2. git pre-commitフックを設定

```bash
cp src/git-hooks/pre-commit /path/to/your-project/.git/hooks/pre-commit
chmod +x /path/to/your-project/.git/hooks/pre-commit
```

### 3. プロジェクト固有の調整

`src/.claude/agents/doc-updater.md` 内の記述規約セクションはテンプレート。プロジェクトのドキュメント構造に合わせてカスタマイズする:

- README.mdのセクション構成
- AGENTS.md / CLAUDE.mdの既存構造
- docs/ 配下のドキュメント体系

## 参考ドキュメント

| ドキュメント | URL |
|-------------|-----|
| Hooks reference | https://code.claude.com/docs/en/hooks.md |
| Automate workflows with hooks | https://code.claude.com/docs/en/hooks-guide.md |
| Create custom subagents | https://code.claude.com/docs/en/sub-agents.md |
| Best Practices for Claude Code | https://code.claude.com/docs/en/best-practices.md |

## 注意点

- **Stopフックのコスト**: LLM判定ゲートは毎回LLM呼び出しが発生する。コストが気になる場合はgit pre-commitフックのみの運用も可能
- **Stopフックのタイムアウト**: 180秒に設定。セッション履歴が大きい場合は調整が必要
- **pre-commitの実行時間**: claude CLIの起動＋分析で数十秒かかる。`--no-verify` でスキップ可能
- **再帰防止**: 環境変数 `DOC_UPDATER_RUNNING` でガードしている
