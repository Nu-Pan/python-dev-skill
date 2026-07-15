# python-dev-skill

Python プロジェクトが宣言する Python、仮想環境、依存関係管理方法、ツール設定を優先し、構成に合わせて検査対象とコマンドを決定する Codex スキルです。pytest fixture による隔離と install 後相当の package test、Ruff、mypy、pytest-timeout、Python development mode、`ResourceWarning` 検査を、変更中の focused check と fresh な完了ゲートに分けて適用します。

## インストール

配布物は [`dist/python-dev-skill`](dist/python-dev-skill) にあります。このリポジトリのルートで、対象リポジトリの絶対 path を指定して次を実行します。

```bash
TARGET_REPO=/absolute/path/to/repository
mkdir -p "$TARGET_REPO/.agents/skills/python-dev-skill"
cp -R dist/python-dev-skill/. "$TARGET_REPO/.agents/skills/python-dev-skill/"
```

これにより、`SKILL.md` と `agents/openai.yaml` が `<target-repository>/.agents/skills/python-dev-skill/` に配置されます。インストール後に Codex のセッションを開始し、明示的に使用する場合はプロンプトで `$python-dev-skill` を指定します。

```text
$python-dev-skill を使って、このリポジトリの変更を実装し、適切な Python 品質ゲートを実行してください。
```
