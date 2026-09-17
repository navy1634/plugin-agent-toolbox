# pytest

## 基本方針

Pythonのテスト実行基盤とテストフレームワークは pytest に統一します。
`unittest`、`unittest.mock`、`unittest.TestCase`、`unittest` の `patch`・`Mock`・`AsyncMock` などを、テストコード、fixture、設定、サンプルから一律に禁止します。
pytest-mockが内部で利用する実装はこの禁止対象ではありませんが、テスト側から直接 import してはいけません。

テストは `tests/unit`、`tests/integration`、`tests/e2e` の責務単位で配置し、fixtureのscopeを必要最小限にします。
テスト名は検証する振る舞いを表し、Arrange、Act、Assertを分けます。テスト間で状態を共有せず、実装詳細ではなく公開結果を検証します。

振る舞い変更では、実装前にテストが期待どおり失敗するREDを確認します。失敗理由が契約の未実装部分を示していることを記録してから実装担当者へ渡し、実装担当者はテストコードを変更して通過させません。

pytestのテスト、fixture、mockの期待値は仕様と受入条件から作成します。実装を読んで現在の挙動に合わせたテストを作成したり、テストを通すために期待値やケースを実装へ合わせて変更したりしません。仕様と実装が食い違う場合は、仕様に基づくREDを維持して差分を報告します。

## docstring とコメント

pytestのテスト、fixture、helper、Page Objectは、テストコードであってもプロダクションコードと同じように読み手が保守します。Pythonの一般規約は `coding-standards` の定義を使い、pytest固有のdocstringは次の方針で書きます。

- docstring、コメント、テスト内の説明は日本語で書き、Google Styleの構造を使います。モジュール先頭のdocstringは置きません。
- fixture、再利用するhelper、Page Objectのクラスと公開メソッドには、呼び出し側が必要とする概要、前提、ライフサイクル、観測結果を記載します。テスト名とコードから明らかな手順を重ねて説明しません。
- fixtureが値を返す場合は `Yields:` または `Returns:`、例外を契約とする場合は `Raises:` を使います。型は注釈へ記載し、docstringでは値の意味・制約・後処理を説明します。
- テスト関数のdocstringは、テスト名とArrange・Act・Assertから前提や期待結果が分からない場合だけ追加します。追加する場合も、検証する利用者の振る舞いを短く示し、実装手順を列挙しません。
- コメントはコードから読み取れない制約や、待機・mock・fixtureの選択理由だけを書き、コードの言い換え、実行順の実況、却下した案は残しません。

次のように、型は注釈に置き、fixtureが提供する値と後処理だけを `Yields:` へ記載します。

```python
from collections.abc import Iterator

import pytest


@pytest.fixture
def test_resource() -> Iterator[dict[str, str]]:
    """テストごとに独立したリソースを準備して後片付けする

    Yields:
        初期化済みのテスト用リソース
    """
    resource = {"name": ""}
    yield resource
    resource.clear()
```

## pytest拡張

対象の事象に対応するpytest拡張がある場合は、独自fixtureや標準ライブラリによる個別実装より先に原則として採用します。導入しない場合は、既存依存との互換性、安全性、実行環境などの理由を記録します。プロジェクトの依存管理、バージョン、既存設定を正本とし、必要な拡張を導入する場合はそのプロジェクトのテスト実行経路へ追加します。

### mock と非同期処理

mock、patch、spy、stubは `pytest-mock` の `mocker` fixtureを使います。`mocker.patch`、`mocker.patch.object`、`mocker.spy`、`mocker.stub` を基本とし、契約へ結び付ける必要がある場合は `spec` または `autospec` を指定します。非同期のdoubleには `mocker.AsyncMock` を使い、`unittest.mock` から直接 import しません。pytest-mockのfixtureはテスト終了時に自動で復元されます。

### AWS SDK

AWS（boto3）の境界テストでは、`mocker.Mock` や自作fakeで成功レスポンスだけを返す方法を禁止します。`moto.mock_aws` を開始してから実際のboto3 clientまたはresourceを生成し、bucket、table、queueなどの前提リソースを作成したうえで、AWS APIの入力検証、状態遷移、例外、永続化結果を検証します。`mock_aws` の外で生成済みのclientを使うと実AWSへ接続する可能性があるため、mockを有効にしてからclientを生成します。

```python
import boto3
from moto import mock_aws


@mock_aws
def test_stores_object_in_s3() -> None:
    client = boto3.client("s3", region_name="ap-northeast-1")
    client.create_bucket(
        Bucket="test-bucket",
        CreateBucketConfiguration={"LocationConstraint": "ap-northeast-1"},
    )

    client.put_object(Bucket="test-bucket", Key="example.txt", Body=b"")

    response = client.get_object(Bucket="test-bucket", Key="example.txt")
    assert response["Body"].read() == b""
```

`moto`で再現できないAWS機能だけは、対応するSDK stubや別のサービスエミュレーターを選び、再現できない範囲を検証報告へ記載します。自作mockへ置き換える場合も、実態と一致しないリスク、対象API、代替手段を記録します。motoの成功は実AWSやCIの証拠と混同しません。

### DB repository と Testcontainers

DBを扱うrepository自身のテストは、原則としてTestcontainersで本番と同じDBエンジン・互換性のあるバージョンを起動し、実際のドライバーとmigrationを使ってテストします。in-memory SQLite、辞書だけの自作repository、戻り値を固定したDB client mockでrepositoryの挙動を代替しません。本番DBがin-memoryまたはSQLiteそのものであるなど、代替が実態と一致する場合だけ例外とし、理由を記録します。

Testcontainersのcontainer lifecycleはpytest fixtureで管理し、テストごとにtransaction、schema、またはデータを初期化します。containerをsession scopeで共有する場合も、各テストの状態が分離される根拠を示します。Dockerが使えない環境ではテストを黙ってskipせず、未実行範囲として報告します。

```python
from collections.abc import Iterator

import pytest
from testcontainers.postgres import PostgresContainer


@pytest.fixture
def database_url() -> Iterator[str]:
    """本番と同じDBエンジンの接続先を準備して破棄する

    Yields:
        Testcontainersが起動したDBの接続URL
    """
    with PostgresContainer("postgres:16") as container:
        yield container.get_connection_url()
```

本番DBがPostgreSQL以外の場合は、対象エンジンのcontainerを使います。Testcontainersのimage、DB初期化、migration、ドライバー、接続URLの設定はプロジェクトの `pyproject.toml` と既存のfixture構成を正本にし、検証報告へimageとDBバージョンを記載します。

### 環境変数

テストスイート全体で固定する環境変数は `pytest-env` で `pyproject.toml` の設定へ定義します。pytest本体と拡張の設定も `pyproject.toml` にまとめ、別のpytest設定ファイルを追加しません。秘密値や実環境の認証情報は設定へ書き込まず、実行環境から安全に注入します。テストケースごとに一時的な差分を作る場合はpytestの `monkeypatch` を使い、終了後に元へ戻します。

### 時刻

現在時刻、日付境界、タイムゾーン、期限切れを検証する場合は、`pytest-freezegun` 系拡張の `freezer` fixtureまたはfreeze用markを使います。datetimeを手作業でpatchしたり、実時間のsleepで境界を待ったりしません。時刻を進める必要がある場合も、fixtureが提供する制御方法で明示します。

### その他の拡張

非同期テストには `pytest-asyncio`、coverageには `pytest-cov` など、プロジェクトの実行基盤と互換性のあるpytest拡張を優先します。HTTP、DB、スナップショット、並列実行なども、既存の品質・安全要件を満たす公式またはプロジェクト承認済みの拡張がある場合は同じ方針で選びます。拡張を形式的に増やすことは目的にしません。

## fixture とmockの値

fixtureのmockは公開契約へ結び付け、文字列は空文字、数値は0、boolはFalse、list・dictは空、nullableはNoneを基本値にします。必要な入力制約がある場合だけ最小の有効値を使い、実装を推測してAPIを増やしません。DB、cache、外部APIはclientの境界をmockまたはemulatorで分離し、実装内部のprivate stateを直接差し替えません。詳細なdoubleの選択は [Test doubles](test-doubles.md) を参照します。

## 配置と実行

プロジェクトの既存構成に合わせて、unit、integration、E2Eのテストを分けます。たとえば次のように、機能名や境界を表すディレクトリを選びます。

```text
tests/
├── unit/
│   └── test_<component>.py
├── integration/
│   └── test_<boundary>.py
├── e2e/
│   └── test_<journey>.py
└── conftest.py
```

プロジェクトにcoverage閾値がある場合は、設定の `--cov-fail-under` を正本にします。設定された範囲に対して `pytest-cov` を実行し、`term-missing` は欠落箇所の調査、HTML reportは詳細調査に使います。80%などの数値を、このskillから一律に要求しません。

DoDはタスクランナーの `test`、`lint`、`format`、`type-check`、必要なE2Eタスクを使います。開発中に単一テストをpytestへ直接渡すことは許可しますが、タスクランナーがあるプロジェクトでruffやmypyを直接呼びません。
理由のない `pytest.skip`、失敗を隠す `xfail`、無期限のretryは使いません。
