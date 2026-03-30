# doc-updater

コード変更時にプロジェクトドキュメント（AGENTS.md, CLAUDE.md, README.md 等）を自動更新する構成一式のテンプレートリポジトリ。`src/` 配下のファイルを他のプロジェクトに導入して使う。

## プロジェクト概要

### プロジェクト構成

```text
doc-updater/
├── AGENTS.md
├── CLAUDE.md -> AGENTS.md
├── GEMINI.md -> AGENTS.md
├── README.md -> AGENTS.md
└── src/
    ├── .agents/
    │   └── skills/
    │       ├── commit-push/
    │       │   ├── SKILL.md
    │       │   └── references/
    │       │       └── commit-message-profiles.md
    │       └── doc-updater/
    │           └── SKILL.md
    └── git-hooks/
        └── pre-commit
```

### テックスタック

- Agent Skills（`src/.agents/skills`）
- Claude CLI を使う `git pre-commit` フック
- Codex / Gemini CLI / GitHub Copilot 等から参照する共通 skill 構成
- シェルスクリプト（`src/git-hooks/pre-commit`）

## 設計方針

### 2つの入口

| 入口 | 役割 |
|------|------|
| `commit-push` スキル | 手動で commit / push したいときに、必要なら先に doc-updater を実行する |
| `git pre-commit` フック | 手動コミット時に staged 差分を見てドキュメント更新を差し込む |

`doc-updater` はドキュメント同期の source of truth、`commit-push` はその呼び出しと commit / push の orchestration を担う。runtime が別 skill の明示呼び出しをサポートしていれば `doc-updater` を直接呼び、サポートしていなければ skill ファイルを読んで同じ手順を inline で実行する。

### テンプレートの責務

- 共通 skill は `.agents/skills/` 配下に置く
- `pre-commit` は `doc-updater` skill ファイルを source of truth として読む
- Claude Code 固有の `.claude/` 設定や stop hook はこのテンプレートには含めない

### symlink戦略

AGENTS.md を実ファイルとして管理し、CLAUDE.md / GEMINI.md / README.md はすべて AGENTS.md への symlink とする。1つのファイルを編集するだけで全 AI エージェントと GitHub の README に反映される。

## 開発・検証

```bash
# pre-commit フックの構文チェック
bash -n src/git-hooks/pre-commit

# ルート symlink の整合性確認
ls -la CLAUDE.md GEMINI.md README.md

# 主要ファイルの参照確認
rg -n "doc-updater|commit-push|SKIP_DOC_UPDATER" AGENTS.md src
```

## 作業ルール

- **`src/` 配下**: 配布用テンプレート。他プロジェクトに導入される内容
- **ルート直下**: このリポジトリ自体の管理。AGENTS.md のみ実ファイルとして編集する
- **symlink（CLAUDE.md, GEMINI.md, README.md）は直接編集しない**。AGENTS.md への変更が自動反映される

各ファイルの役割:

| ファイル | 役割 |
|---------|------|
| `src/.agents/skills/doc-updater/SKILL.md` | `git diff --cached` を対象にドキュメント更新要否と更新手順を定義する source of truth |
| `src/.agents/skills/commit-push/SKILL.md` | ステージ確認、doc-updater 呼び出し、profile 選択、commit / push を orchestrate する |
| `src/.agents/skills/commit-push/references/commit-message-profiles.md` | runtime ごとの commit trailer と補助ルールを管理する |
| `src/git-hooks/pre-commit` | `doc-updater` skill を読んで実行する `git pre-commit` フック |

## 検出対象の変更

doc-updater は以下の変更を検出し、ドキュメントへの反映が必要か判断する:

- **コード変更**: 機能追加、API 変更、エージェント・スキル・フックの変更
- **依存関係の変更**: package.json, requirements.txt, Cargo.toml 等のライブラリ追加・削除・変更
- **ファイルのリネーム・削除**: ドキュメント内のパス参照を自動更新・除去
- **コマンド・スクリプトの変更**: Makefile, justfile, npm scripts 等のビルド・テスト・実行コマンド変更
- **設定ファイルの変更**: .env.example, docker-compose.yml, tsconfig.json 等の開発環境設定

## 処理の流れ

### commit-push

```text
ステージ済み差分を確認
  → 必要なら doc-updater を呼ぶ
  → commit message profile を選ぶ
  → commit
  → push
```

### git pre-commit

```text
ステージ済み差分を確認
  → Claude CLI が doc-updater skill ファイルを読む
  → git diff --cached を分析
  → 必要ならドキュメントを更新して git add
```

## セットアップ（利用者向け）

### 1. ファイルを配置

```bash
mkdir -p /path/to/your-project/.agents/skills
cp -r src/.agents/skills/commit-push /path/to/your-project/.agents/skills/
cp -r src/.agents/skills/doc-updater /path/to/your-project/.agents/skills/
```

### 2. git pre-commitフックを設定

```bash
cp src/git-hooks/pre-commit /path/to/your-project/.git/hooks/pre-commit
chmod +x /path/to/your-project/.git/hooks/pre-commit
```

### 3. プロジェクト固有の調整

`src/.agents/skills/doc-updater/SKILL.md` と `src/.agents/skills/commit-push/references/commit-message-profiles.md` はテンプレート。プロジェクトに導入したら以下を必要に応じて調整する:

- README.md のセクション構成
- AGENTS.md / CLAUDE.md / GEMINI.md の既存構造
- docs/ 配下のドキュメント体系
- runtime ごとの commit trailer や補助ルール
- doc-updater をどの入口からどう呼ぶか

## 参考ドキュメント

| ドキュメント | URL |
|-------------|-----|
| Codex skills | https://developers.openai.com/codex/skills |
| GitHub Copilot skills | https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills |
| Agent Skills specification | https://agentskills.io/specification |

## 注意点

- **pre-commit の実行時間**: Claude CLI の起動と差分分析で数十秒かかる。`--no-verify` でスキップ可能
- **Claude CLI が無い環境**: `pre-commit` はそのままスキップする
- **再帰防止**: 環境変数 `SKIP_DOC_UPDATER` でガードしている
- **Claude Code 固有設定**: `.claude/` や stop hook はテンプレートに含めない。必要なら導入先で別管理する
