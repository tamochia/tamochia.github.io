---
layout: post
title: "(続)GitHub Actionsを利用してJekyll v4.4でビルドしたGitHub Pagesを公開する"
date: 2026-08-15
description: "GitHub Actions を利用した Jekyll 4.4 による GitHub Pages のビルドとデプロイについての備忘録です。"
categories: 
  - "Jekyll"
tags:
  - "Ruby"
  - "GitHub"
---

GitHub Actionsにて次のような警告が出るようになった。

```
Warning: Node.js 20 is deprecated. The following actions target Node.js 20 but are being forced
to run on Node.js 24: actions/checkout@v4, actions/upload-artifact@v4. For more information
see: https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/
```

これは、Node 20 が 2026年4月にサポート終了（EOL）となっていることから、GitHub Actionsにおいても段階的廃止しますよ。っていうことみたいです。2026年秋には、すべてのアクションをNode24で実行できるよう移行する予定のようです。

[Deprecation of Node 20 on GitHub Actions runners - GitHub Changelog](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/)

現在、この tamo's archives も GitHub Pages を使用していますが、Jekyll 4.4 でビルドしていることから、クラッシックモードいわゆるブランチによるデプロイではなく、GitHub Actions を使用してビルドとデプロイを行っています。

[GitHub Actionsを利用したJekyll 4.4によるGitHub Pages構築](https://tamochia.github.io/article/building-github-pages-with-jekyll-4.4-using-github-actions.html)

ここで示したワークフローを約1年間使用してきましたが、特にメンテナンスはしていませんでした。

調べてみると警告の対象は次のActionでした。

- actions/checkout@v4
- actions/upload-artifact@v4
- actions/download-artifact@v4
- actions/configure-pages@v4
- actions/deploy-pages@v4

これらは内部的に Node.js 20 を使用しているバージョンですので、この度、Node.js 24 対応版へ更新することにしました。

現在のワークフローは build / deploy の2つのジョブで構成し、それぞれの steps にて次のようなアクションを実行しています。

- build ジョブ
	- actions/checkout@v4
	- ruby/setup-ruby@v1
	- run: `bundle exec jekyll build`
	- actions/upload-artifact@v4
- deploy ジョブ
	- actions/download-artifact@v4
	- actions/configure-pages@v4
	- actions/upload-pages-artifact@v3
	- actions/deploy-pages@v4

このうち、「actions/upload-pages-artifact」は内部的に artifact (成果物) を管理することが分かったため、build ジョブから deploy ジョブへの成果物の橋渡しの部分、

```
actions/upload-artifact
↓
actions/download-artifact
```

は簡略化できそうです。つまり無くても良さそうということです。

(参考)[GitHub Pages公開フローを5年ぶりに刷新した #Security - Qiita](https://qiita.com/Adacchi3/items/bb8c862af0b2dadddf2c)

新しくしたワークフローは build / deploy の2つのジョブ構成は変わらず、次のような流れに変えました。

- build ジョブ
	- actions/checkout@v6
	- actions/configure-pages@v5
	- ruby/setup-ruby@v1
	- run: `bundle exec jekyll build`
	- actions/upload-pages-artifact@v4
- deploy ジョブ
	- actions/deploy-pages@v5

1. リポジトリのデフォルトブランチへのプッシュがトリガーとなりワークフローが実行
2. `actions/checkout` アクションを使用し、リポジトリの内容をチェックアウト
3. 静的サイト用ファイルのビルド
4. `actions/upload-pages-artifact` アクションを使用し、静的ファイルを成果物(artifact)としてアップロード
5. `actions/deploy-pages` アクションを使用し、成果物をデプロイ
    - ワークフローがpull requestによってトリガーされた場合は、このステップはスキップされる

(参考)[GitHub ページでのカスタム ワークフローの使用 - GitHubドキュメント](https://docs.github.com/ja/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages#linking-separate-build-and-deploy-jobs)

上記参考のサイトでは、Jekyllでのビルドの部分を次のように記載されています。

```yaml
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
        with:
          source: ./
          destination: ./_site
```

実際に、このビルド方法で試してみたところ、スタイルシートが反映されていないページが公開されてしまいました。
このワークフローの実行ログを確認したところ、次のような警告が出ていました。

```
Warning: The github-pages gem can't satisfy your Gemfile's dependencies. If you want to use 
a different Jekyll version or need additional dependencies, consider building Jekyll site 
with GitHub Actions: https://jekyllrb.com/docs/continuous-integration/github-actions/
```

`bundle install` も失敗しており、生成されたHTMLソースをみると Jekyll v3.10.0 となっていました。

```
<meta name="generator" content="Jekyll v3.10.0" />
```

やはり、Jekyll v4 で動かそうとすれば、「ruby/setup-ruby@v1」アクションにて Ruby 3.4 のコンテナを起動させ、その上で「bundle install」そして「bundle exec jekyll build」を実行するひと手間が必要なのです。以下、今回更新したワークフローです。

```yaml
name: Build and Deploy Jekyll site to GitHub Pages

# mainブランチへのpush時にトリガー
on:
  push:
    branches:
      - main
  workflow_dispatch:   # 手動実行も可能にする

# ワークフロー全体では権限なし（ジョブごとに明示）
permissions: {}

jobs:
  build:
    name: Build Jekyll Site
    runs-on: ubuntu-latest

    # buildジョブに必要な最低限の権限
    permissions:
      contents: read

    steps:
      # ソースコードをチェックアウト
      - name: Checkout source code
        uses: actions/checkout@v6

      # GitHub Pages の設定
      - name: Setup Pages
        uses: actions/configure-pages@v5

      # Ruby環境をセットアップ（Jekyll実行用）
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.4'  # 必要に応じて変更
          bundler-cache: true   # gemsのインストールとキャッシュを自動化 

      # Jekyllでサイトをビルド
      - name: Build the site with Jekyll
        run: bundle exec jekyll build
        env:
          JEKYLL_ENV: production

      # ビルド成果物（_site）をアーティファクトとして保存
      - name: Upload build artifact
        uses: actions/upload-pages-artifact@v4
        
  deploy:
    name: Deploy to GitHub Pages
    needs: build  # buildジョブが完了してから実行
    runs-on: ubuntu-latest

    # デプロイに必要な権限のみを指定
    permissions:
      pages: write
      id-token: write

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      # GitHub Pagesにアーティファクトをデプロイ
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```
