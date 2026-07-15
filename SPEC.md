# スキルの仕様

## 概要

- Python 開発環境として有用なツールの使用を推奨する
- 変更設計、境界検証、resource lifecycle、隔離 test、機械的検証を一体として扱うことで、AI コーディングエージェントの作業品質の底上げを狙う
- プロジェクトが宣言する Python のバージョン、仮想環境、依存関係管理方法、ツール設定を優先する。Python のバージョンが宣言されていない場合は Python 3.11 以上を使用する
- Ruff、mypy、pytest-timeout が未導入の場合は、既存の開発用 dependency group や requirements file へ追加する。依存関係管理方法がない場合は、リポジトリ内の `.venv` に pip で導入し、グローバル環境を変更しない
- 作業中は必要に応じて自動修正を使用してよいが、完了前にはファイルを書き換えないモードで fresh な検証を実行する。実行できなかった検証を成功として扱わず、実行した command、結果、未実行の理由を最終報告に残す
- 対象 path や test command を固定せず、設定ファイル、package 構成、既存の開発手順を調査して決定する

## 変更設計

### goal

- リポジトリの指示、正本仕様、既存 architecture、公開契約を調べ、要求を満たす最小限の変更に留める
- module、class、function の責務と入出力を明確にし、CLI、web、job などの entrypoint や framework adapter は引数解釈と委譲を中心とする薄い境界に保つ
- 複数箇所で実際に共有される処理は、既存の package 境界に沿った共通 module へ集約する
- import path、公開 symbol、CLI、設定・永続化 schema、package layout などの公開契約を変更する場合は、利用側と互換性への影響を確認する

### non-goal

- 特定の framework、entrypoint file、共通 module 名、`src` layout を全 project へ強制すること
- 将来の再利用を予測した抽象化、要求外の大規模 refactor、無関係な code の整理を同時に行うこと
- 最小変更を理由に、判明した correctness、security、resource leak の不具合を隠すこと

## Python coding

### goal

- project 固有の style を優先し、未定義の場合は PEP 8 と Python ecosystem の標準的な命名に従う。text file は tool や既存規約に別指定がなければ UTF-8 BOM なしで扱う
- 新規・変更する公開 API と非自明な function・class には正確な型 hint を付け、非公開の module・class 識別子は既存の公開方針に反しない範囲で `_` から始める
- 公開 API と意図・副作用・失敗条件が code だけでは読み取りにくい対象には、project 既存 style の簡潔な docstring を付ける。signature から自明な情報を繰り返さない
- comment は処理の逐語説明ではなく、理由、invariant、trade-off、workaround、外部契約など、code だけでは残らない意図を説明する
- 循環 import は module 分割、依存方向、責務配置の見直しで解消する。`TYPE_CHECKING` は構造的な解消が適切でない場合に限定する

### non-goal

- relative import または absolute import の一方、特定の docstring style、comment・log の言語、`from __future__ import annotations` の使用可否を一律に強制すること
- 変更と無関係な既存 code へ型 hint、docstring、comment を一括追加すること
- 自明な code block ごとに comment を追加し、実装と同期しない説明を増やすこと

## 入力境界・global state・外部 process

### goal

- config、serialized data、外部 command の出力、file、network response などの入力境界で、型、必須 field、許容値、空値、path の所属を明示的に検証する。契約にない欠落や不正値を default で黙って補わない
- OS・library の低水準例外は、application 境界で利用者が対処できる domain error に変換し、原因の exception chain、対象 path・argv・設定などの診断情報を保持する
- cwd、環境変数、signal handler、global・context-local state、lock を一時変更する処理は context manager または `try/finally` で復元・解放する
- subprocess は原則として argv の list で起動し、Python child process には選択済み interpreter または `sys.executable` を使う。`cwd`、環境、text/binary、標準入出力、exit code、timeout の契約を明示し、失敗時に stdout・stderr を調査できるようにする
- thread、process、subprocess、process group を開始した code が lifecycle と cleanup を所有する。timeout・中断・部分初期化を含む全終了経路で、残存 process と resource を確認する
- 並行実行される state 更新、lock、path 予約には atomic・排他的な操作を使用し、競合時の一貫性を test する

### non-goal

- 広範な `except`、無言の fallback、根拠のない retry で原因を隠すこと
- 必要性を確認せず `shell=True`、process-global な cwd・環境変更、強制終了を使用すること
- timeout 値を延ばすだけで deadlock、I/O 待ち、cleanup 不備を回避すること

## test 設計

### goal

- まず project が責任を持つ決定論的な制御 logic と公開契約を test し、外部 service や生成 AI の品質そのものと分離する
- filesystem、repository、HOME、cwd、環境変数、設定を `tmp_path` などの一時領域と fixture・monkeypatch で隔離し、利用者の global state、hook、署名設定、credential、既存 file に依存または作用しないようにする
- 外部 command・service の実動作が test の目的でなければ fake・stub を使う。呼び出し境界自体が project の責務である場合は、side effect と費用を抑えた限定的な integration test も用意する
- network access、有料 API、subscription quota を消費する backend は、明示的な許可と隔離された integration test 設計がない限り自動 test で使用しない
- package・import・公開 symbol を変更した場合は、source checkout だけでなく install 後相当の layout でも import と resource 参照を検証する。schema、定数、型などの契約定義は正本を参照し、test 用 copy を別の定義として増やさない
- invalid input、境界値、error、timeout、中断、部分初期化、cleanup、並行競合を変更内容に応じて test し、exit code、stdout・stderr、永続 state、残存 resource を外部契約として確認する
- optional な外部 executable を必要とする test は、存在を検査して具体的な理由付きで skip してよい。ただし fake で検証できる必須 logic や required validation を skip で代用しない

### non-goal

- 外部 service、LLM、第三者 CLI 自体の品質・安定性を project の自動 test で保証すること
- fake だけで実 integration 契約を検証済みとすること、または全 test で実 service を起動すること
- test の順序、利用者環境、外部 network、過去の生成物に依存する test を残すこと
- skip された optional integration test を実行成功として扱うこと

## Ruff

### goal

- プロジェクトの設定を使用して lint、import の整理状態、format を検査し、構文上・静的解析上の明白な不具合、未使用の import、import 順序、基本的な style 違反を機械的に検出する
- 変更中は変更箇所に絞った検査を行い、完了前には first-party の Python code 全体に対して、少なくとも以下に相当する read-only な検査を実行する

```bash
python -m ruff check <対象 path>
python -m ruff format --check <対象 path>
```

- 設定がない状態で Ruff を導入する場合は、小さく説明可能な rule set から始める。指摘は原則として code または正当な設定変更によって解消し、抑制が不可避な場合は最小範囲に限定して理由を近傍へ残す

### non-goal

- Ruff の全 rule、preview rule、厳格な docstring rule を最初から一律に有効化すること
- Ruff を通すことだけを目的に、無関係な code を一括整形したり、挙動を変えたり、広範な `noqa` や file-level ignore を追加したりすること
- 既存の linter、formatter、import sorter を調査せずに置換すること、または互いに矛盾する formatter 設定を並存させること
- 自動修正後の diff を確認せず、そのまま完了とすること

## mypy

### goal

- プロジェクトの設定と package 構成から検査対象を決め、first-party の Python code に対して型の不整合、到達不能な前提、不適切な `Any` の流出、無効になった ignore を検出する
- 変更中は変更した module とその利用側を検査し、完了前にはプロジェクトが定める全対象に対して、以下に相当する fresh な検査を実行する

```bash
python -m mypy <対象 path または package>
```

- 設定がない状態で mypy を導入する場合は first-party code を blocking な対象とし、既存 code の型付け状況に合わせて段階的に厳格化する。型エラーは原則として実装または型注釈を修正して解消する

### non-goal

- 既存 code の状態を調査せず、最初から strict mode 全体や全関数への型注釈を強制すること
- error を隠すために、対象全体の除外、`ignore_errors`、広範な `type: ignore`、根拠のない `Any` や `cast` を追加すること
- vendored code、生成 code、仮想環境、第三者 package まで無条件に型検査の対象とすること
- mypy の成功を runtime test の代わりにすること

## pytest-timeout

### goal

- pytest を使用するプロジェクトでは、停止、deadlock、終了しない外部 process を検出するため、pytest-timeout を開発依存関係へ追加して全体に保守的な timeout を設定する
- timeout 値は正常時の実測時間と実行環境の揺らぎを考慮して決める。正当に長い test には、理由を残したうえで test 単位の timeout を設定する
- 変更中の focused test と完了前の full test の両方で timeout を有効にし、timeout が発生した場合は stack、process、thread、I/O 待ち、teardown を調査して原因を解消する
- 強制終了では fixture の teardown や cleanup が完了しない可能性を考慮し、残った process や一時 resource の有無も確認する

### non-goal

- timeout を性能要件や benchmark として使用すること
- 正常な test でも不安定になるほど短い timeout を、実測せず一律に設定すること
- timeout した test を、原因を調査せず無効化、skip、または過度に長い timeout に変更すること
- pytest を使用していないプロジェクトへ pytest または pytest-timeout を強制すること

## Python development mode と ResourceWarning

### goal

- 完了前の full test を、Python development mode と `ResourceWarning` のエラー化を有効にした以下相当の環境で実行する

```bash
PYTHONDEVMODE=1 PYTHONWARNINGS="error::ResourceWarning" python -m pytest
```

- project 固有の test runner を使用する場合も、その runner が起動する Python process へ同等の設定を適用する
- 検出された file、socket、subprocess、async task、その他の resource の解放漏れは、原則として lifecycle や cleanup を修正して解消する
- 第三者 library だけが発生させる warning を除外する必要がある場合は、実際の出力を根拠に module、message、warning category を用いて最小範囲に限定し、理由を記録する

### non-goal

- `ResourceWarning` 以外を含む全 warning を、この Skill だけを根拠として一律にエラー化すること
- project code による resource leak を warning filter、広範な pytest 設定、環境変数の解除によって隠すこと
- 第三者 library の warning を、project code に原因があるか調査せず修正対象または除外対象と決めること
- focused test の成功や過去の実行結果だけで、development mode を使用した full test が成功したと報告すること
- development mode と `ResourceWarning` 検査だけで、すべての resource leak を検出できると保証すること
