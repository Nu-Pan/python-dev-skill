
# 概要

- Python 開発環境として有用なツールの使用を推奨するスキル
- 機械的検証を手厚くサポートすることで、AI コーディングエージェントの作業品質の底上げを狙う
- このスキルを使用したいリポジトリに `.agents/skills/python-dev-skill` として配置される想定
- この AGENTS.md の内容を正本として、スキルの実装が生成されるものとする
- プロジェクトが宣言する Python のバージョン、仮想環境、依存関係管理方法、ツール設定を優先する。Python のバージョンが宣言されていない場合は Python 3.11 以上を使用する
- Ruff、mypy、pytest-timeout が未導入の場合は、既存の開発用 dependency group や requirements file へ追加する。依存関係管理方法がない場合は、リポジトリ内の `.venv` に pip で導入し、グローバル環境を変更しない
- 作業中は必要に応じて自動修正を使用してよいが、完了前にはファイルを書き換えないモードで fresh な検証を実行する。実行できなかった検証を成功として扱わず、実行した command、結果、未実行の理由を最終報告に残す
- 対象 path や test command を固定せず、設定ファイル、package 構成、既存の開発手順を調査して決定する

# Ruff

## goal

- プロジェクトの設定を使用して lint、import の整理状態、format を検査し、構文上・静的解析上の明白な不具合、未使用の import、import 順序、基本的な style 違反を機械的に検出する
- 変更中は変更箇所に絞った検査を行い、完了前には first-party の Python code 全体に対して、少なくとも以下に相当する read-only な検査を実行する

```bash
python -m ruff check <対象 path>
python -m ruff format --check <対象 path>
```

- 設定がない状態で Ruff を導入する場合は、小さく説明可能な rule set から始める。指摘は原則として code または正当な設定変更によって解消し、抑制が不可避な場合は最小範囲に限定して理由を近傍へ残す

## non-goal

- Ruff の全 rule、preview rule、厳格な docstring rule を最初から一律に有効化すること
- Ruff を通すことだけを目的に、無関係な code を一括整形したり、挙動を変えたり、広範な `noqa` や file-level ignore を追加したりすること
- 既存の linter、formatter、import sorter を調査せずに置換すること、または互いに矛盾する formatter 設定を並存させること
- 自動修正後の diff を確認せず、そのまま完了とすること

# mypy

## goal

- プロジェクトの設定と package 構成から検査対象を決め、first-party の Python code に対して型の不整合、到達不能な前提、不適切な `Any` の流出、無効になった ignore を検出する
- 変更中は変更した module とその利用側を検査し、完了前にはプロジェクトが定める全対象に対して、以下に相当する fresh な検査を実行する

```bash
python -m mypy <対象 path または package>
```

- 設定がない状態で mypy を導入する場合は first-party code を blocking な対象とし、既存 code の型付け状況に合わせて段階的に厳格化する。型エラーは原則として実装または型注釈を修正して解消する

## non-goal

- 既存 code の状態を調査せず、最初から strict mode 全体や全関数への型注釈を強制すること
- error を隠すために、対象全体の除外、`ignore_errors`、広範な `type: ignore`、根拠のない `Any` や `cast` を追加すること
- vendored code、生成 code、仮想環境、第三者 package まで無条件に型検査の対象とすること
- mypy の成功を runtime test の代わりにすること

# pytest-timeout

## goal

- pytest を使用するプロジェクトでは、停止、deadlock、終了しない外部 process を検出するため、pytest-timeout を開発依存関係へ追加して全体に保守的な timeout を設定する
- timeout 値は正常時の実測時間と実行環境の揺らぎを考慮して決める。正当に長い test には、理由を残したうえで test 単位の timeout を設定する
- 変更中の focused test と完了前の full test の両方で timeout を有効にし、timeout が発生した場合は stack、process、thread、I/O 待ち、teardown を調査して原因を解消する
- 強制終了では fixture の teardown や cleanup が完了しない可能性を考慮し、残った process や一時 resource の有無も確認する

## non-goal

- timeout を性能要件や benchmark として使用すること
- 正常な test でも不安定になるほど短い timeout を、実測せず一律に設定すること
- timeout した test を、原因を調査せず無効化、skip、または過度に長い timeout に変更すること
- pytest を使用していないプロジェクトへ pytest または pytest-timeout を強制すること

# Python development mode と ResourceWarning

## goal

- 完了前の full test を、Python development mode と `ResourceWarning` のエラー化を有効にした以下相当の環境で実行する

```bash
PYTHONDEVMODE=1 PYTHONWARNINGS="error::ResourceWarning" python -m pytest
```

- project 固有の test runner を使用する場合も、その runner が起動する Python process へ同等の設定を適用する
- 検出された file、socket、subprocess、async task、その他の resource の解放漏れは、原則として lifecycle や cleanup を修正して解消する
- 第三者 library だけが発生させる warning を除外する必要がある場合は、実際の出力を根拠に module、message、warning category を用いて最小範囲に限定し、理由を記録する

## non-goal

- `ResourceWarning` 以外を含む全 warning を、この Skill だけを根拠として一律にエラー化すること
- project code による resource leak を warning filter、広範な pytest 設定、環境変数の解除によって隠すこと
- 第三者 library の warning を、project code に原因があるか調査せず修正対象または除外対象と決めること
- focused test の成功や過去の実行結果だけで、development mode を使用した full test が成功したと報告すること
- development mode と `ResourceWarning` 検査だけで、すべての resource leak を検出できると保証すること
