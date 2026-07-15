# python-dev-skill

Python プロジェクトの開発時に、静的解析、型検査、テストの停止検知、resource leak 検査を組み合わせた品質ゲートを適用する Codex スキルです。

対象プロジェクトが宣言する Python のバージョン、仮想環境、依存関係管理方法、既存のツール設定を優先し、プロジェクトの構成に合わせて検査対象とコマンドを決定します。

## 主な機能

- Ruff による lint、import、format の検査
- mypy による first-party code の型検査
- pytest-timeout による停止、deadlock、終了しない外部 process の検知
- Python development mode と `ResourceWarning` のエラー化を使用した resource leak 検査
- focused check と fresh な完了前検証の使い分け、および未実行・失敗を含む検証結果の報告

## インストール

配布物は [`dist/python-dev-skill`](dist/python-dev-skill) にあります。このリポジトリのルートで、対象リポジトリの絶対 path を指定して次を実行します。

```bash
TARGET_REPO=/absolute/path/to/repository
mkdir -p "$TARGET_REPO/.agents/skills/python-dev-skill"
cp -R dist/python-dev-skill/. "$TARGET_REPO/.agents/skills/python-dev-skill/"
```

インストール後の構造は次のようになります。

```text
<target-repository>/
└── .agents/
    └── skills/
        └── python-dev-skill/
            ├── SKILL.md
            └── agents/
                └── openai.yaml
```

インストール後に Codex のセッションを開始し、明示的に使用する場合はプロンプトで `$python-dev-skill` を指定します。

```text
$python-dev-skill を使って、このリポジトリの変更を実装し、適切な Python 品質ゲートを実行してください。
```
