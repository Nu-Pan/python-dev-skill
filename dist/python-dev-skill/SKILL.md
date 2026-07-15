---
name: python-dev-skill
description: Python プロジェクトの開発、修正、レビュー、開発環境整備で、既存の Python・仮想環境・依存関係管理・ツール設定を尊重し、最小変更、責務と公開契約、型・docstring、入力境界、global state・subprocess の lifecycle、隔離 test を設計したうえで Ruff、mypy、pytest-timeout、Python development mode、ResourceWarning 検査を実行する。Python コードや test を変更するとき、外部 process・並行処理・永続 state を扱うとき、品質ゲートや resource leak を調査するときに使用する。
---

# Python 開発を安全に設計・実装・検証する

プロジェクト固有の開発手順を保ちながら、変更設計、境界検証、resource lifecycle、隔離 test、静的検査、型検査、停止検知を一つの完了条件として扱う。

## 1. プロジェクトを調査する

変更前に、対象リポジトリから次を確認する。

- リポジトリ内の指示、開発ドキュメント、CI、既存の検証コマンドを読む。
- `pyproject.toml`、lockfile、requirements file、`setup.cfg`、`tox.ini` などから、Python のバージョン、仮想環境、依存関係管理方法、Ruff・mypy・test runner の設定を特定する。
- package 構成と import 境界を調べ、first-party code、変更した module、その利用側、focused test、full test の対象を決める。
- vendored code、生成 code、仮想環境、第三者 package を first-party の検査対象に混ぜない。
- 宣言済みの Python、環境、dependency manager、設定、project script を優先する。`requires-python` などが範囲なら、その範囲を満たす interpreter を選び、CI と同じ版での再現や最低対応版の互換性確認が必要かを作業内容から判断する。下限の版だけを必須 executable と解釈しない。Python のバージョンが宣言されていない場合だけ Python 3.11 以上を使用する。

対象 path や test command を決め打ちしない。複数の設定がある場合は、プロジェクトが実際に使用する設定と CI の呼び出し方を照合する。

## 2. 検証ツールを準備する

プロジェクトが採用する dependency manager と開発用 dependency group または requirements file を使用する。

- Ruff と mypy が未導入なら、既存の開発依存関係へ追加し、必要なら lockfile をプロジェクトの方法で更新する。
- pytest を使用するプロジェクトでは、pytest-timeout が未導入なら同じ開発依存関係へ追加する。pytest を使用していないプロジェクトへ、このスキルだけを理由に pytest または pytest-timeout を導入しない。
- 依存関係管理方法がない場合は、リポジトリ内の `.venv` を宣言済みの Python、または未宣言なら Python 3.11 以上で作成し、その環境へ pip で導入する。global environment を変更しない。
- 原則として選択した interpreter から `python -m ruff`、`python -m mypy`、test runner を起動し、別環境の executable を誤って使用しない。command に記載する前に interpreter が実在して宣言範囲を満たすことを `--version` で確認する。CI や最低対応版と同じ interpreter がローカルにない場合は、範囲を満たす利用可能な版で実行できる検証を行い、未確認の版を成功扱いせず別途報告する。
- 読み取り専用の依頼など、依存関係を変更する権限がない場合は変更せず、欠けているツールと実行できない検証を報告する。

既存の linter、formatter、import sorter、型検査、test runner を先に調べる。無断で置換せず、設定が競合するツールを並存させない。

## 3. 最小の変更を設計する

- リポジトリの指示、正本仕様、既存 architecture、公開契約を確認し、要求を満たす最小限の変更範囲を決める。要求外の refactor や将来を予測した抽象化を混ぜない。
- module、class、function の責務と入出力を明確にする。CLI、web、job などの entrypoint や framework adapter は引数解釈と委譲を中心とする薄い境界に保ち、domain logic と分離する。
- 複数箇所で実際に共有する処理だけを、既存の package 境界に沿った共通 module へ集約する。特定の framework、directory 名、`src` layout を新たに強制しない。
- import path、公開 symbol、CLI、設定・永続化 schema、package layout を変更するときは、利用側、後方互換性、migration の要否を確認する。
- schema、定数、型などの正本定義を test や互換 module に複製せず、参照または明示的な再公開で一貫性を保つ。

## 4. Python code を実装する

- project 固有の style を優先し、未定義なら PEP 8 と Python ecosystem の標準的な命名に従う。text file は別指定がなければ UTF-8 BOM なしで扱う。
- 新規・変更する公開 API と非自明な function・class に正確な型 hint を付ける。非公開の module・class 識別子は既存の公開方針に反しない範囲で `_` から始める。
- 公開 API と、意図・副作用・失敗条件が code だけでは読み取りにくい対象に、既存 style の簡潔な docstring を付ける。signature の情報を冗長に繰り返さない。
- comment は処理の逐語説明ではなく、理由、invariant、trade-off、workaround、外部契約を説明する。自明な block ごとの comment を強制しない。
- 循環 import は module 分割、依存方向、責務配置の見直しで解消する。`TYPE_CHECKING` は構造的な解消が適切でない場合に限定する。
- relative/absolute import、docstring style、comment・log の言語、`from __future__ import annotations` の方針は project の既存規約に従い、一律の好みを持ち込まない。

## 5. 入力境界と lifecycle を守る

- config、serialized data、外部 command 出力、file、network response の境界で、型、必須 field、許容値、空値、path の所属を検証する。契約にない欠落や不正値を default で黙って補わない。
- OS・library の低水準例外は application 境界で対処可能な domain error に変換し、exception chain と path・argv・設定などの診断情報を保持する。広範な `except` や無言の fallback で原因を隠さない。
- cwd、環境変数、signal handler、global・context-local state、lock の一時変更は context manager または `try/finally` で復元・解放する。
- subprocess は原則 argv の list で起動し、Python child process には選択済み interpreter または `sys.executable` を使う。`cwd`、環境、text/binary、標準入出力、exit code、timeout の契約を明示する。
- thread、process、subprocess、process group を開始した code が lifecycle を所有し、正常終了、error、timeout、中断、部分初期化の全経路で cleanup する。強制終了後は descendant と残存 resource を確認する。
- 並行実行される state 更新、lock、path 予約には atomic・排他的な操作を使い、競合時にも一貫性を保つ。

## 6. 決定論的で隔離された test を作る

- project が責任を持つ決定論的 logic と公開契約を先に test し、外部 service、第三者 CLI、生成 AI の品質そのものと分離する。
- filesystem、repository、HOME、cwd、環境変数、設定を `tmp_path` などの一時領域と fixture・monkeypatch で隔離する。利用者の global state、hook、署名設定、credential、既存 file に依存または作用させない。
- 外部 command・service の実動作が目的でなければ fake・stub を使う。呼び出し境界自体が project の責務なら、side effect と費用を抑えた限定的な integration test も用意する。
- network、有料 API、subscription quota を消費する backend は、利用者の明示的な許可と隔離された integration test 設計がない限り自動 test で使わない。
- package・import・公開 symbol を変更したら install 後相当の layout でも検証する。schema、定数、型などの契約定義は正本を参照し、test 用の独立した copy を増やさない。
- invalid input、境界値、error、timeout、中断、部分初期化、cleanup、並行競合を変更内容に応じて test し、exit code、stdout・stderr、永続 state、残存 resource を確認する。
- optional な外部 executable がない test は具体的な理由付きで skip してよいが、fake で検証できる必須 logic や required validation を skip で代用しない。

## 7. Ruff を使用する

プロジェクトの Ruff 設定を使用し、lint、未使用 import、import 順序、format を検査する。

- 作業中は変更した first-party path を中心に `python -m ruff check ...` と `python -m ruff format --check ...` を実行する。
- 必要に応じて限定した対象へ自動修正または format を適用してよい。適用後は diff を確認し、挙動変更や無関係な一括整形がないことを確かめる。
- 設定なしで Ruff を導入する場合は、明白な構文・静的解析違反、未使用 import、import 順序、基本的な style を扱う小さく説明可能な rule set から始める。全 rule、preview rule、厳格な docstring rule を一律に有効化しない。
- 指摘は code または根拠のある設定変更で解消する。抑制が不可避なら最小の行または対象に限定し、理由を近傍へ残す。広範な `noqa` や file-level ignore で通過させない。

完了前は first-party Python code 全体に対して、少なくとも次に相当する書き換えなしの検査を fresh に実行する。

```bash
python -m ruff check <first-party targets>
python -m ruff format --check <first-party targets>
```

## 8. mypy を使用する

プロジェクトの mypy 設定と package 構成から検査対象を決め、型の不整合、到達不能な前提、不適切な `Any` の流出、無効になった ignore を検出する。

- 作業中は変更した module と、その型契約を利用する側を検査する。狭すぎる単一 file の成功だけで package 全体の整合性を判断しない。
- 設定なしで導入する場合は first-party code を blocking な対象とし、既存 code の型付け状況に合わせて段階的に厳格化する。最初から strict mode 全体や全関数への注釈を強制しない。
- 到達不能 code と無効な ignore の検出が既存設定で無効なら、baseline を確認しながら `warn_unreachable` と `warn_unused_ignores` に相当する設定を有効にする。`Any` は first-party の境界と流入元を追跡し、必要な注釈や stub を追加する。
- 型エラーは実装または正確な型注釈で解消する。対象全体の除外、`ignore_errors`、広範な `type: ignore`、根拠のない `Any` や `cast` で隠さない。
- vendored code、生成 code、仮想環境、第三者 package を無条件に検査対象へ含めない。
- mypy の成功を runtime test の代わりにしない。

完了前はプロジェクトが定める first-party の全対象に対して、次に相当する検査を fresh に実行する。

```bash
python -m mypy <first-party targets or packages>
```

## 9. pytest の停止を検知する

pytest を使用する場合は、停止、deadlock、終了しない外部 process を検出するため、pytest-timeout を focused test と full test の両方で有効にする。

- pytest-timeout が実際に読み込まれ、global timeout が適用されることを pytest の設定と plugin 情報で確認する。
- 正常時の focused test と full test の実測時間、および CI・実行環境の揺らぎを踏まえ、保守的な global timeout を設定する。導入時に暫定値が必要なら十分に余裕のある値で計測し、根拠のない数値を最終設定として残さない。実測せずに正常な test まで不安定にする短い値を一律設定せず、timeout を性能要件や benchmark として扱わない。
- 正当に長い test だけに test 単位の timeout を設定し、長くなる理由を近傍へ残す。
- timeout が発生したら stack、process、thread、I/O 待ち、fixture teardown を調査して原因を解消する。調査なしに test を disable・skip したり、timeout を過度に延ばしたりしない。
- timeout による強制終了後は cleanup が完了していない可能性を考慮し、残った child process、一時 file、socket、その他の resource を確認して片付ける。

## 10. development mode で resource leak を検査する

完了前の full test を、Python development mode と `ResourceWarning` のエラー化を有効にして実行する。pytest の標準的な例は次のとおりとし、project 固有の runner がある場合は、その runner が起動する Python process に同等の環境を伝播させる。

```bash
PYTHONDEVMODE=1 PYTHONWARNINGS="error::ResourceWarning" python -m pytest <project full-test arguments>
```

- file、socket、subprocess、async task、その他の resource leak は、所有者の lifecycle、context manager、fixture teardown、または cleanup を修正して解消する。
- 第三者 library だけが発生させる warning を除外する前に、実際の traceback と再現結果から発生元を切り分け、project code の誤った lifecycle や API 利用が原因でないことを確認する。除外が必要なら category、module、message を使って最小範囲に限定し、理由を記録する。
- project code の leak を warning filter、広範な pytest 設定、環境変数の解除で隠さない。`ResourceWarning` 以外の全 warning を、このスキルだけを理由に一律でエラー化しない。
- development mode と `ResourceWarning` 検査だけですべての resource leak を検出できるとは保証しない。

pytest を使用しない場合も full test の実行を省略せず、採用されている runner へ同等の development mode と warning 設定を適用する。test suite 自体が存在しない場合は、その事実と確認方法を報告する。

## 11. 完了ゲートを実行する

変更作業が終わったら、過去の出力や focused check を使い回さず、現在の worktree で次を順に実行する。

1. diff を要求と照合し、無関係な変更、正本の複製、意図しない公開契約・package layout の変更がないことを確認する。
2. 入力境界、error 経路、global state、resource cleanup、並行競合と、それらに対応する隔離 test を見直す。first-party code 全体、型検査対象、full test command が project の実態と一致することも確認する。
3. Ruff の lint と format check を、自動修正を伴わない mode で全対象へ実行する。
4. mypy を全対象へ実行する。
5. pytest を使う場合は pytest-timeout が有効な状態で、focused test ではなく full test を development mode と `ResourceWarning` のエラー化付きで実行する。別 runner の場合も同等に実行する。
6. 検証中に file を変更した場合は diff を再確認し、影響する検証だけでなく完了ゲート全体をもう一度 fresh に実行する。

自動修正用の `--fix` や formatter の書き換え mode を完了ゲートに混ぜない。失敗した検証や実行できなかった検証を成功として扱わない。

## 12. 結果を報告する

最終報告に次を明記する。

- 実行した command と対象範囲
- 各 command の成功、失敗、timeout、または未実行という結果
- 未実行の検証がある場合は、その具体的な理由と残る影響
- timeout や resource warning があった場合は、原因、修正、残存 process・resource の確認結果
- skip した optional integration test、使用した fake・実 service、network・有料 backend を使用しなかった／許可を得て使用した範囲
- 追加または変更した dependency、設定、最小限の抑制とその理由

検証できていない範囲に成功表現を使わない。focused test の成功、mypy の成功、過去の CI 結果のいずれか一つだけで、full runtime test や全体の完了を代用しない。
