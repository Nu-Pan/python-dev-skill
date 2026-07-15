# リポジトリの仕様

## 目的

- `SPEC.md` の内容を正本として、Python 開発を安全に設計・実装し、機械的に検証するスキルの実装を生成する

## 配布物

- `.agents/skills/python-dev-skill` に配置するべき配布物は、`dist/python-dev-skill` 内にのみ生成する
- `dist/python-dev-skill` を対象リポジトリの `.agents/skills` へコピーすることで、`.agents/skills/python-dev-skill` としてインストールできる構造にする
- 配布物には `SKILL.md`、`agents/openai.yaml`、およびスキルの実行時に必要なリソースだけを含める
- リポジトリ直下の `AGENTS.md`、`SPEC.md`、`README.md`、`LICENSE` は配布物に含めない

## README

- リポジトリ直下の `README.md` に、このスキルの説明とインストール方法を記載する
- インストール方法では、`dist/python-dev-skill` を対象リポジトリの `.agents/skills/python-dev-skill` に配置する手順を示す
