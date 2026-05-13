# Playwright のお試し

## First touch

最初の一歩は [とほほのPlaywright入門](https://www.tohoho-web.com/ex/playwright.html) がやりやすい。

Windows では MSYS2 + UCRT64 で nodejs を入れてしまうのが楽かも。

    > pacman -S mingw-w64-ucrt-x86_64-nodejs

あとはほぼ「とほほ」通り。ディレクトリ名は変えている。

```console
$ mkdir playwright-playwright
$ cd playwright-playwright

$ npm init playwright@latest
# 全ての質問にはEnterでデフォルト選択した
# 特に3ブラウザのインストールはYesとした方が良い

$ npx playwright test
# 2つのテストケースが3つのブラウザで、計6ケースが実行される

$ npx playwright show-report
# テストの実行結果が内蔵ブラウザとHTMLベースで表示される
```

テストの実行は基本ヘッドレス。

ただしオプションで実行用のUIを表示したり、ブラウザの画面を表示したり、ブラウザを限定して実行したりできる。

```console
$ npx playwright test --ui

$ npx playwright test --headed

$ npx playwright test --headed --project=chromium
$ npx playwright test --headed --project=firefox
$ npx playwright test --headed --project=webkit
```

テストスクリプトは以下に示すようにBDD風味で、 `await` が手動で実行する際の感覚にあっているためか、読みやすい。

<details>
<summary>example.spec.ts (initで例として生成されるスクリプト)</summary>

```typescript
import { test, expect } from '@playwright/test';

test('has title', async ({ page }) => {
  await page.goto('https://playwright.dev/');

  // Expect a title "to contain" a substring.
  await expect(page).toHaveTitle(/Playwright/);
});

test('get started link', async ({ page }) => {
  await page.goto('https://playwright.dev/');

  // Click the get started link.
  await page.getByRole('link', { name: 'Get started' }).click();

  // Expects page to have a heading with the name of Installation.
  await expect(page.getByRole('heading', { name: 'Installation' })).toBeVisible();
});
```

</details>

MSの過去のテスト用製品の知見が濃縮されたような感じ。

## First touch で気になった件の調査

実際に触ってみて、いくつかの気になる点があった。

-   `page` に対して行える操作にはどんなものがある?
-   `expect` のように、振る舞いの記述に使える動詞は? 
-   テストケースの構造化、再利用は?

### `page` に対して行える操作

[Page class](https://playwright.dev/docs/api/class-page) に記載されている。
ブラウザのタブに相当。

いくつか気になるところ抜粋:

-   `get*` 系
    -   `getByAltText` - 画像の alt 属性でマッチ
    -   `getByLabel` - input 要素に対応する label 要素のテキストでマッチ。もしくは input の [aria-label 属性](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label)。
    -   `getByPlaceholder` - input 要素の placeholder 属性でマッチ
    -   `getByRole` - [ARIA ロール、ARIA 属性](https://www.w3.org/TR/wai-aria-1.2/) および[アクセシビリティ名](https://w3c.github.io/accname/#dfn-accessible-name)でのマッチ
    -   `getByTestId` - `data-testid` 属性でのマッチ
    -   `getByText` - `innerText` プロパティでのマッチ
    -   `getByTitle` - [`title` 属性](https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Global_attributes/title) でのマッチ
    -   `locator` - セレクタ文字列でのマッチ。 [`Locator` クラス](https://playwright.dev/docs/api/class-locator) を返す。おそらく要素を集合として操作できる
-   `wait*` 系
    -   `waitForEvent` - 指定したイベントを待つ
    -   `waitForFunction` - 指定した関数が true を返すまで待つ。任意の条件で待つことができる
    -   `waitForLoadState` - `load` イベント等を待つ。厳密には名前どおりに特定の load state になるまで待つ
    -   `waitForRequest` - ブラウザが発する特定の HTTP リクエストを待つ
    -   `waitForResponse` - ブラウザが受け取る特定の HTTP レスポンスを待つ
    -   `waitForURL` - ページが特定のURLへ遷移するのを待つ
-   その他
    -   `screenshot()` - スクリーンショットのキャプチャ・保存
    -   `video()` - 録画された動画にアクセスする

### 振る舞いの記述に使える動詞・その他

気になったところを抜粋:

-   [Test class](https://playwright.dev/docs/api/class-test) - `@playwright/test` から提供されるシンボル一覧。 `expect` も含まれる
    -   `expect` テスト対象となるオブジェクトを引数
-   [Assertions](https://playwright.dev/docs/test-assertions) - 振る舞いの記述に使える動詞(≒ Assertions)がまとまってる。動詞は大きく以下のように分離されてる
    -   [Auto-retrying assertions](https://playwright.dev/docs/test-assertions#auto-retrying-assertions) - 特定の条件を満たすまで待つ
    -   [No-retrying assertions](https://playwright.dev/docs/test-assertions#non-retrying-assertions) - ワンショットで条件を満たしているか
    -   [Asymmetric matchers](https://playwright.dev/docs/test-assertions#asymmetric-matchers) - 他の assertions 無いで利用できる条件
    -   [Negative matchers](https://playwright.dev/docs/test-assertions#negating-matchers) - 条件を反転する条件
    -   [Soft assertions](https://playwright.dev/docs/test-assertions#soft-assertions) - 条件未達時に即時停止するのではなく、失敗として記録して実行を続ける仕組み
    -   [expect.configure](https://playwright.dev/docs/test-assertions#expectconfigure) - タイムアウトなど、グローバルな設定を変更する
    -   [テストにおける Assertion の書き方](https://playwright.dev/docs/writing-tests#assertions)

-   [Fixture](https://playwright.dev/docs/api/class-fixtures) - `browser`, `browserName`, `context`, `page`, and `request`
    
    各テストケースの名前付き引数として解決される。

### テストケースの構造化、再利用

-   [テストの基本的な書き方](https://playwright.dev/docs/writing-tests)
-   [テストの自動記録](https://playwright.dev/docs/codegen-intro)

構造化に対する支援は今のところ見つけられてない。
TypeScript (ホスト言語)の仕組みを使えば良い、という話ではある。

サブテストは定義できないが [`test.step`](https://playwright.dev/docs/api/class-test#test-step) で、ケース内部を分割管理できる。

## AI (LLM)との協調

AI (LLM) と Playwright との協調方法を調べる。

-   LLMによる直接操作 (MCP Server や browser-use)

    Playwright MCP Server では自然言語でブラウザ操作を指示できる。
    DOM構造を解釈して自律的に実行。
    Chat AIから利用可能。

    browser-use でできることの例:

    - 情報の調査: 特定のトピックについて複数のサイトを巡回し、スプレッドシートにまとめる
    - フォーム入力: CSVデータをもとに、複雑な社内システムのフォームに一件ずつ入力・送信する
    - E2Eテスト: 「会員登録から商品購入まで」といった大まかな指示で、シナリオを自律実行させる

-   テスト開発の自動化
-   AIによる検証 (expect)
-   codegen (テストの自動記録) + AIによる modification - 記録したコードのリファクタリングや Assertion の追加

## ここまでのまとめ#1

-   セットアップが極めて簡単
    -   複数ブラウザ対応
    -   CI (GitHub Actions) の設定
-   テストシナリオ・コードの見た目が自然
    -   await がマッチする
    -   BDDとassertionのハイブリッドっぽい
-   テストのメンテ・管理が楽
    -   UIでスクショを取りつつ実行など
-   AIとの協調を前程にしている

## Second Touch

### Playwright自身が提供するAI連携

第一歩として [公式サイト](https://playwright.dev/) から、公式が提供しているAI機能を確認。

-   [MCP](https://playwright.dev/mcp/introduction) -
    Playwright が提供するブラウザ自動化機能をAIに対してい提供するアダプター。
    視覚モデルは無くても良い。
    エンジニアと対話、伴走する利用方法を想定 (Headed)。
    自立エージェントは CLI を使う。
-   [CLI](https://playwright.dev/agent-cli/introduction) - 
    コーディングエージェント向けにブラウザ自動化機能を提供する。
    より大規模向け。
    自立型で Headless 動作。
    トークン効率が良い。
    エージェントにとっての「スキル」となる。
-   [Playwright Test Agents](https://playwright.dev/docs/test-agents) -
    MCPを使って、テストの計画、実装、メンテナンス(修正)を担う3種のエージェント: planner, generator, and healer が提供される。

MCPは Playwright Test Agents以外だと、CursorやVS Code (GitHub Copilot)との直接連携で使われる。
他にも諸々を自作する場合に採用できる。

高いけど、賢いMCP。安いけど、賢くはないCLI。AgentsはMCPを使って、テストの全工程を自動化、もしくはAIによる補助をする。

### 他のブラウザ操作系AIエージェントとの比較

[Computer-useとBrowser-useとPlaywright-MCPを比較](https://zenn.dev/headwaters/articles/7f0717b61848c3) -
テストとは違った観点で、「AIによるブラウザ操作の自動化」を検討。利用モデルはGPT 4.1。
トークン量の少なさから browser-use が使いやすそう。
裏ではどれもPlaywrightを使っているように見えるが、トークン量にこれだけの差が生じる理由が不明。
画像を読んでるか否からしい。
画像≒ブラウザのスクリーンショットを見て、操作を適用させようとすると、どうしても高くつくと言うこと。
browser-use は DOM だけ見てる感じ。

[AIエージェントによるブラウザ自動化ツール3種比較](https://www.ytyng.com/blog/ai-browser-automation-tools-comparison-2026) -
AIによる比較レポートw AIがAIエージェントに指示を出し、動作を比較している。
トークン効率から agent-browser (Vecel Labs) を推奨とし、
不安定さから Claude in Chrome を非推奨としてる。

Playwright Test Agents はトークン使用量が増える傾向にある。
増大の理由は、DOMの大きさ(長さ)、スクリーンショットの利用、試行錯誤。
Test Agents にプロンプトにてPlaywright CLIを使わせる手も有効そう。

### ここまでのまとめ #2

-   Playwright 自身はコードによるブラウザ操作の自動化フレームワークである
-   Playwright を AI に操作させる口として MCP と CLI が用意されている
    -   これらとAIエージェントを連携させることで、AIにブラウザ操作を任せることができる
    -   一長一短だが、AIにより柔軟に判断させたい場合はMCPが、大量に操作したい場合はCLIが向く
-   Playwright Test Agents はMCPもしくはCLIを用いて、テスト工程全体の補助をする
    -   テスト全体を設計するプランナー
    -   テストコードを生成するジェネレーター
    -   テストコードをメンテ・修正するのヘルパー

    利用できるモデルは Claude (Anthropic), VS Code Copilot, OpenCode に限定されている。
    OpenCode はローカルを含めいろんなプロバイダを使えるので、それで良いとなりそう。
