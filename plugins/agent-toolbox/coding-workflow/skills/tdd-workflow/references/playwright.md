# Playwright E2E

## 適用範囲と構成

Playwrightは、ブラウザを入口に利用者の主要なjourneyを検証するときに使います。Pythonプロジェクトでは、既存のpytest、`pytest-playwright`、必要に応じて `pytest-asyncio` の実行経路へ統合し、テストコードの規約は `coding-standards` と [pytest](pytest.md) に従います。

プロジェクトの規模と既存構成に合わせますが、テスト対象、ページ操作、共有fixtureを分ける構成にします。

```text
tests/
├── e2e/
│   ├── auth/
│   │   └── test_login.py
│   ├── resources/
│   │   ├── test_browse.py
│   │   └── test_create.py
│   └── conftest.py
├── pages/
│   ├── auth_page.py
│   └── resources_page.py
pyproject.toml
```

固定のディレクトリ名を導入すること自体を目的にせず、既存のテスト探索設定と所有単位を優先します。Page Objectはセレクターと利用者の操作・観測をまとめ、業務ルールや受入条件を隠しません。

## シナリオの作り方

各journeyを、正常系、空状態、入力境界、入力エラー、権限不足、依存先障害などの独立した振る舞いへ分解します。認証、データ作成、検索、更新、削除、決済など、失敗時の影響が大きい経路から優先します。

テストはArrange、Act、Assertを保ち、各テストが自分のデータと認証状態を準備します。テスト間でPage、BrowserContext、ユーザー、DB状態を共有せず、実行順を変えても同じ結果になるようにします。

## locator と待機

利用者が認識できる role、label、名称、placeholderを優先し、画面とテストの契約として安定した `data-testid` を使います。偶然のCSS class、深いDOM階層、位置番号だけのselectorは避けます。

画面の状態、URL、レスポンス、要素の可視性など、目的の結果を表す条件を待ちます。`wait_for_timeout` の固定sleepや、完了を確認しない `networkidle` 依存を標準手段にしません。レスポンスを待つ場合は、操作と対象URL・status・schemaの条件を結び付けます。

## Page Object の例

```python
from playwright.async_api import Locator, Page


class ResourcesPage:
    def __init__(self, page: Page) -> None:
        self.page = page
        self.search_input: Locator = page.get_by_role("textbox", name="Search")
        self.resource_cards: Locator = page.get_by_test_id("resource-card")

    async def goto(self) -> None:
        await self.page.goto("/resources")

    async def search(self, query: str) -> None:
        async with self.page.expect_response(
            lambda response: "/api/resources/search" in response.url
            and response.status == 200
        ):
            await self.search_input.fill(query)

    async def resource_count(self) -> int:
        return await self.resource_cards.count()
```

Page Objectは利用者の操作と観測可能な結果を公開し、内部locatorの組み立てやHTTP実装をテスト本文へ漏らしません。テスト本文では、結果の意味を `expect` や公開値で検証します。

```python
import pytest
from playwright.async_api import Page, expect
from pages.resources_page import ResourcesPage


@pytest.mark.asyncio
async def test_user_can_search_resources(page: Page) -> None:
    resources = ResourcesPage(page)

    await resources.goto()
    await resources.search("example")

    await expect(resources.resource_cards.first).to_be_visible()
    assert await resources.resource_count() > 0
```

実際のプロジェクトで `page` fixture、非同期marker、base URLの指定方法が異なる場合は、その設定を正本とします。例の文字列や件数を実環境のデータへ依存させません。

## 環境と実行

base URL、browser、headless、timeout、slow motion、traceの設定は、プロジェクトのpytest設定またはテスト環境設定から読み込みます。認証情報、実データ、秘密鍵をソースや `.env` のコミット対象へ置きません。

開発中は対象ファイルやheaded mode、inspectorを使って調査できます。完了判定では、プロジェクトのタスクランナーが定義する全E2E範囲をheadlessで実行し、ブラウザ種類を増やす場合は受入条件とCI設定に記録します。

```bash
# 開発中の個別確認
uv run pytest tests/e2e/test_<journey>.py -k <case> -q

# プロジェクトにタスクランナーがない場合の全体例
uv run pytest tests/e2e
```

## flaky test とアーティファクト

失敗したテストは、まず待機条件、競合、データ分離、環境、依存先の遅延を調査します。リトライは原因調査の補助に限り、CIの失敗を無期限に隠しません。隔離する場合は、flakyである根拠、再現条件、修正issue、解除条件、CIから除外する範囲を記録します。

失敗時は、スクリーンショット、trace、必要なvideo、browser／server log、入力と環境を保存します。全ステップの画像やvideoを無条件に残すのではなく、原因の再現とレビューに必要な範囲を選びます。アーティファクトの保持期間とアップロードはCI設定の正本に従います。

HTML reportやJUnit XMLを出力する場合は、実行日時、対象suite、総数、成功・失敗・skip・flaky、失敗箇所、再現手順、アーティファクトの場所、未実行範囲を報告します。ローカル・emulator・staging・実環境・CIの結果を混ぜません。
