---
title: "GitHub Actions のみで、actions/cache も使わない最軽量の VRT"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [githubactions, playwright, vrt]
published: true
---

Web アプリケーション開発での VRT 導入は、ちゃんと運用するとなると以下のような多くの検討事項を伴います。

- Storybook のストーリーベースで比較するか？それとも実アプリケーションの URL ベースで比較するか？
- CI 上でアプリケーションをビルドして dev server を立ち上げるか、それともデプロイ先のアプリケーションにアクセスするか？
- スクリーンショットの比較はどのように行うか？比較時の閾値はどのように設定するか？
- 比較元のスクリーンショットはどのように用意するか？例えば Amazon S3 などのストレージ や GitHub Actions の actions/cache を使用する場合など
  - コミットハッシュを用いて比較元のスクリーンショットを特定する場合、マージ先のコミットハッシュに紐づくスクリーンショットが存在しない時の対応は？
- VRT の結果で差分が出たが、それが期待通りの変更であった場合、ベースラインをどのように更新するか？
- 実行時間の増加や flaky な実行結果をどのように抑制するか？

とても大変です。とはいえ大規模なプロダクトではこれらの点を一つ一つ慎重に検討する必要があります。しかし小規模なプロダクトだったり、とにかくまずは小さく VRT を始めたい！という場合もあるでしょう。検討事項をできるだけ減らし、最も軽量な VRT 導入方法を検討します。

## 基本方針

最軽量な VRT を導入するため、今回は以下の方針で実装を進めます。

- **Playwright を使用したスクリーンショットの撮影・比較**：Playwright はヘッドレスブラウザを操作し、簡単にスクリーンショットを撮影できます。またピクセル単位での差分を検出する機能も備えており、VRT を容易に実現できます。
- **GitHub Actions だけでの完結**：外部サービスを使用せず、GitHub Actions のみで全ての VRT プロセスを管理します。これにより、セットアップが簡単となり、追加コストや管理負荷を最小限に抑えます。
- **スクリーンショットの都度撮影と比較**：過去のスクリーンショットをキャッシュや外部ストレージに保存せず、毎回比較元・比較先両方のスクリーンショットを撮影し比較します。これにより、比較元のスクリーンショットが存在しないという問題が解消されますが、実行時間の増加はトレードオフとなります。
- **Preview Environment での VRT 実施**：Pull Request に紐づく Preview Environment へのデプロイをトリガーとして VRT を実施します。今回は Vercel を使用する場合を想定します。これにより CI 上でのビルドや dev server の設定を考慮する必要がありません。

最軽量を目指しているため、いくつかのトレードオフが存在します。今回の例では Vercel へのデプロイ・Playwright や GitHub Actions の使用という前提条件があり、すべてのプロダクトに適用可能なわけではありません。しかし、他のツールを使用場合であっても同様のアプローチを実現できるかと思います。

## 実装

### Playwright の設定

https://playwright.dev/docs/intro

上記を参考に Playwright を設定します。インストール時にはサンプルファイルも生成されますが、今回は不要なため削除します。

```bash
pnpm create playwright
```

以下はこのプロジェクト用の Playwright の設定です：

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./src/__vrt__", // テストファイルを格納するディレクトリ
  testMatch: "vrt.test.ts", // 今回設定するテストファイルの名前
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: process.env.BASE_URL, // ここは環境変数で指定するようにする
    trace: "on-first-retry",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
    { name: "Mobile Chrome", use: { ...devices["Pixel 5"] } },
    { name: "Mobile Safari", use: { ...devices["iPhone 12"] } },
  ],
});
```

次に、テストファイル `./src/__vrt__/vrt.test.ts` を作成します：

```ts:./src/__vrt__/vrt.test.ts
import type { Page } from '@playwright/test'
import { test, expect } from '@playwright/test'

const screenshot = async (page: Page, targetPage: TargetPage) => {
  await page.goto(targetPage.path)
  // Playwright では、toHaveScreenshot でスクリーンショットによるスナップショットができる
  await expect(page).toHaveScreenshot({ fullPage: true })
}

interface TargetPage {
  name: string
  path: string
}

const targetPages: TargetPage[] = [
  {
    name: 'home',
    path: '/',
  },
]

for (const targetPage of targetPages) {
  test(targetPage.name, async ({ page }) => {
    await screenshot(page, targetPage)
  })
}
```

Playwright の `toHaveScreenshot()` はスクリーンショットを撮影しつつピクセル差分を検出するため、VRT を簡単に実現できます。詳しくは以下のドキュメントを参照してください。

https://playwright.dev/docs/test-snapshots

最後に、package.json に以下のスクリプトを追加します。

```json:package.json
{
  "scripts": {
    "test:vrt:screenshots": "playwright test --update-snapshots",
    "test:vrt:compare": "playwright test"
  }
}
```

`--update-snapshots` オプションは、新しいスナップショットの結果を正として更新するためのものです。  
`test:vrt:screenshots` ではスクリーンショットを撮影し、`test:vrt:compare` では撮影したスクリーンショットと比較することを想定しています。

### GitHub Actions の設定

続いて GitHub Actions の設定を記載します。以下のワークフローファイルを追加します:

```yaml:.github/workflows/vrt.yml
name: VRT

on:
  # デプロイが成功した時に実行する。これにより、Preview Environment の URL に対して VRT を実行することができる
  deployment_status:

jobs:
  screenshots:
    # Preview Environment へのデプロイが成功した時に実行する
    if: github.event_name == 'deployment_status' && github.event.deployment_status.state == 'success' && github.event.deployment_status.environment == 'Preview'
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read

    # Playwright が提供する Docker イメージを使用することで、セットアップを簡単にする
    container:
      image: mcr.microsoft.com/playwright:v1.40.1-focal

    steps:
      - uses: actions/checkout@v4

      # 以下はpnpmや依存のセットアップなので説明は省略
      - uses: actions/setup-node@v3.5.1
        with:
          node-version-file: .node-version
      - uses: pnpm/action-setup@v2
        with:
          version: 8.9.2
          run_install: false
      - name: Get pnpm store directory
        id: pnpm-cache
        run: echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT
      - uses: actions/cache@v3
        name: Setup pnpm cache
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}-setup-pnpm
          restore-keys: |
            ${{ runner.os }}-pnpm-store-setup-pnpm
      - run: pnpm install
        shell: bash

      # 本番環境に対してスクリーンショットを撮る。このスクリーンショットが比較元となる
      - run: pnpm test:vrt:screenshots
        env:
          BASE_URL: https://example.com/

      # デプロイが成功した Preview 環境に対してスクリーンショットを撮り、比較する
      - run: pnpm test:vrt:compare
        env:
          BASE_URL: ${{ github.event.deployment_status.environment_url }}

      # 失敗時に理由がわかるよう、失敗したスクリーンショットをアップロードする
      - name: Upload failed screenshots
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: vrt-failed-screenshots-${{ github.sha }}
          path: test-results
```

重要な点をピックアップして紹介していきます。

#### `on: deployment_status`

```yaml
on:
  deployment_status:
```

通常の CI では push や pull_request イベントで実行することが多いですが、今回は deployment_status イベントをトリガーとして VRT を実行します。Vercel へのデプロイ時には GitHub の Deployments API を叩いてくれるため、このイベントからデプロイ先の URL を取得することができます。

https://vercel.com/guides/how-can-i-run-end-to-end-tests-after-my-vercel-preview-deployment

#### if 条件

```yaml
if: github.event_name == 'deployment_status' && github.event.deployment_status.state == 'success' && github.event.deployment_status.environment == 'Preview'
```

deployment_status イベントのうち、プレビュー環境のデプロイが成功した時のみ実行します。

#### Playwright の Docker イメージ

```yaml
container:
  image: mcr.microsoft.com/playwright:v1.40.1-focal
```

Playwright の Docker イメージを使っています。GitHub Actions デフォルトの ubuntu ランナーでもインストールは可能ですが、 `pnpm dlx playwright install --with-deps` でインストールしたバイナリや依存のキャッシュを考えると少々複雑です。Playwright の Docker イメージを使うことで、セットアップを簡単にすることができます。

#### 本番環境のスクリーンショット撮影

```yaml
- run: pnpm test:vrt:screenshots
  env:
    BASE_URL: https://example.com/
```

まず本番環境のスクリーンショットを撮影します。先ほど package.json に設定した `test:vrt:screenshots` の中身は `playwright test --update-snapshots` です。これはスクリーンショットが撮影できたら成功となります。  
この指定の方法では、例えば Feature Branch A から Feature Branch B へのマージをする PR などのようなブランチ戦略には対応できず、常に本番環境との比較になります。ただ今回は最軽量を目指すため複雑な仕組みは入れません。

#### Preview 環境のスクリーンショット撮影

```yaml
- run: pnpm test:vrt:compare
  env:
    BASE_URL: ${{ github.event.deployment_status.environment_url }}
```

次に Preview 環境のスクリーンショットを撮影します。`test:vrt:compare` の中身は `playwright test` です。差分があった場合は失敗します。

#### 失敗時のスクリーンショットアップロード

```yaml
- name: Upload failed screenshots
  if: failure()
  uses: actions/upload-artifact@v3
  with:
    name: vrt-failed-screenshots-${{ github.sha }}
    path: test-results
```

失敗時のデバッグができるよう、テスト結果を GitHub Actions の Artifacts にアップロードします。`test-results`には Playwright が結果を格納してくれています。  
この形だと GitHub Actions の Summary から Artifacts の結果をダウンロードし、手元で zip ファイルを解凍して確認することになります。別の VRT ツールである reg-suit や Chromatic では PR 上から遷移可能な URL から差分を確認することができますが、今回は最軽量を目指すためそのようなリッチな機能は実装しません。

### 運用

Pull Request を作成し、Vercel bot による Preview Environment へのデプロイが完了したら VRT のワークフローが実行されます。成功・失敗で以下のように対応していきます。

#### 成功時

UI のデグレードはないですね！ということで、PR のレビュープロセスやマージに進みましょう。

#### 失敗時

UI の差分が発生しているので、以下の項目をチェックします。

- UI の変更が意図したものでない: UI のデグレードが検知できましたね！開発に戻り実装を修正しましょう。
- UI の変更が意図したものである: CI は落ちたままですが、そのまま PR のレビュープロセスに進みましょう。
  - Chromatic などでは[ベースライン](https://www.chromatic.com/docs/branching-and-baselines/)という概念があり、変更に対して Approve することで、その状態が正であるとしてベースラインを更新します。
  - 今回はそもそもキャッシュを保存していないため、ベースラインを更新する必要がありません。CI が落ちているのは気になりますが、運用上特に問題はありません。

## まとめ

この記事では、GitHub Actions を使用して最軽量なビジュアルリグレッションテスト（VRT）を実現する方法を紹介しました。小規模なプロダクトを想定し、ストレージや比較対象のスクリーンショットの管理を不要にすることができましたが、以下のトレードオフが存在します。

- 本番環境との比較しかできない
- 毎回 2 回スクリーンショットを撮影するため、実行時間が増加する
- 失敗時の結果は Artifacts からダウンロードすることになるので少々面倒

これらの問題に対応するためには比較元のスクリーンショットを外部に保存しジョブ上で復元する必要があります。その際の選択肢や実装方法はこちらが参考になりました。

https://zenn.dev/cybozu_ept/articles/practice-vrt-using-github-actions-cache

また、[Chromatic](https://www.chromatic.com/)や[percy](https://percy.io/)などの SaaS や、[reg-suit](https://reg-viz.github.io/reg-suit/)などの OSS を使うのも選択肢の一つかと思います。プロダクトの状況に合わせて、最適な VRT の導入方法を検討してみてください。
