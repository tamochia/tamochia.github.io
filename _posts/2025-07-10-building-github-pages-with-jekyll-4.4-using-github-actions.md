---
title: "GitHub Actionsを利用したJekyll 4.4によるGitHub Pages構築"
date: 2025-07-10
categories: 
  - "Jekyll"
tags:
  - "Ruby"
  - "GitHub"
---

# はじめに

GitHub Pagesへのビルドとデプロイについて、**クラッシックモード** と **GitHub Actions** を使用した方法があります。

- クラシックモードには静的HTMLの配信と、Jekyllという静的ジェネレータを使用することが可能
- 何も設定しなければ、自動的にJekyllが採用される
- クラシックモードでJekyllを使用しない場合は、リポジトリのルートに「`.nojekyll`」を置く
	- GitHub Actions を使用する場合は、「`.nojekyll`」は不要
- クラシックモードでのJekyllはバージョン3.xで古い
- Jekyll 4.xを使用したいなら、GitHub Actions を使ってビルド＆デプロイする必要がある

今回は、Jekyll 4.4 を使用したいので、GitHub Actions を使用します。

まずは、[GitHub](https://github.com) にログインして、［New repository］で GitHub Pages の公開用（Public）リポジトリを作成しておきます。
その際に、リポジトリ名を「`○○○○○○.github.io`」（`○○○○○○` はユーザ名）に設定する必要があります。

「Settings」＞「Pages」の「Build and deployment」の「Source」を「**GitHub Actions**」に変更します。
なお、従来のクラシックモードを使用する場合は「Deploy from a branch」にて、「main」など適当なブランチを指定します。

![](/assets/images/20250710a01.png){:width="500px"}


# ローカルリポジトリの作成

適当な作業用ディレクトリ配下にディレクトリ「`○○○○○○.github.io`」を作成し、そのディレクトリに移動しておきます。
（私は、リポジトリ名と同じ名前のディレクトリを作るようにしています。）

```console
$ mkdir -p ~/適当/○○○○○○.github.io
$ cd ~/適当/○○○○○○.github.io
```

以降は、ディレクトリ「`○○○○○○.github.io`」をカレントディレクトリとして話を進めていきます。

ここをgitのローカルリポジトリとし、GitHubのリポジトリと関連付けます。

```console
$ git init
$ git remote add origin git@github.com:○○○○○○/○○○○○○.github.io.git
$ git remote -v
```

適当なファイル `README.md` を作成してコミット及びリモートのmainブランチにプッシュしておきます。

```console
$ touch README.md
$ git add .
$ git commit -m "Initial commit"
$ git push origin main
```

# シンプルなJekyllのサイトを作成

まずは Bundler の設定から、Jekyll は bundle で動かし、インストールされる gems は `vendor/bundle` 配下に置くようにします。

```console
$ bundle init
$ bundle config set --local path 'vendor/bundle'
$ bundle config set --local without 'development test'
```

上記コマンドの実行で `./Gemfile` と、`.bundle/config` が作成されます。

**.bundle/config**
```ruby
---
BUNDLE_PATH: "vendor/bundle"
BUNDLE_WITHOUT: "development:test"
```

`Gemfile` は次のように修正します。
なお、Ruby Ver.3.5.0以降の場合は、gem `logger` が標準で組み込まれていないのでインストールしておかないと、`jekyll new` したときに警告が出るようです。

**Gemfile**
```ruby
# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll"
gem "logger"

group :jekyll_plugins do
  gem 'jekyll-paginate'
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
end
```

これらgemsをローカルにインストールします。

```console
$ bundle install
```

`jekyll new` コマンドにて「`--blank`」を指定して、空のJekyllサイトを作成します。

```console
$ bundle exec jekyll new . --blank --force
New jekyll site installed in /home/○○○○○○/適当/○○○○○○.github.io.
```

こんな感じのディレクトリ・ファイル構成になるはずです。

```console
$ tree -L 3
.
├── Gemfile
├── Gemfile.lock
├── README.md
├── _config.yml
├── _data
├── _drafts
├── _includes
├── _layouts
│   └── default.html
├── _posts
├── _sass
│   └── base.scss
├── assets
│   └── css
│       └── main.scss
├── index.md
└── vendor
    └── bundle
        └── ruby
```

SASSがDart Sass 3.0.0 なので、`assets/css/main.scss` で設定されている `import` が非推奨ということで警告が出ます。

**assets/css/main.scss**
```scss
---
---

@import "base";
```

なので、 `@import` を `@use` に変更し、ついでに、`_sass/base.scss` を `_sass/_base.scss` にファイル名を変更しておきます。

とりあえず、これでJekyllは動くので、確認してみます。

```console
$ bundle exec jekyll serve
```

# GitHub Actions ワークフロー

ローカルリポジトリにて、「`.github/workflows/`」という名前のディレクトリを作成し、その中にワークフローファイル「`○○○○.yml`」を作成します。

まずは、今回作成したワークフローファイル（YAML形式）のファイルの完成版を掲載します。

{% raw %}
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
        uses: actions/checkout@v4

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
        uses: actions/upload-artifact@v4
        with:
          name: jekyll-site
          path: ./_site
          retention-days: 1

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
      # ビルドアーティファクトをダウンロード
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: jekyll-site
          path: ./_site

      # GitHub Pages の設定
      - name: Setup Pages
        uses: actions/configure-pages@v4

      # ビルドした静的ファイルをアップロード
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./_site

      # GitHub Pagesにアーティファクトをデプロイ
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
{% endraw %}

**特徴**
- ジョブ（`jpbs:`）を、ビルド（`build:`）とデプロイ（`deploy:`）に分離
- デプロイに `needs: build` を明記し、依存関係を明確に定義
- パーミッション設定として
	- ビルドジョブは、 `contents: read` のみ
	- デプロイジョブは、 `pages: write` と `id-token: write`のみ
- デプロイジョブの環境変数の設定にて、GitHub Pages環境を指定、URL出力の設定

{% raw %}
```yaml
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
```
{% endraw %}

- `vendor/bundle` のGemsのキャッシュ
	- 従来の手動で行う方法（`actions/cache@v3`）で行う方法と、`ruby/setup` アクションの `bundler-cache: true` で行う方法がある
	- 後者の方法は、以下を自動的にやってくれる
		- 参考 [【GitHub Actions】キャッシュ #初心者 - Qiita](https://qiita.com/Taira0222/items/9f65f3f87c8039a28916)
		- `bundle config` の設定
		- `bundle install` の実行
		- `vendor/bundle` のキャッシュ
		- `Gemfile.lock` をキーにしたキャッシュキーの自動生成
		- `actions/cache` を内部で使う
	- 注意としては、 `Gemfile.lock` をリポジトリに加える（コミットする）必要あり。`.gitignore` には入れない。
- アーティファクト（Artifact）について
	- GitHub Actions時に生成されたファイル、異なるJobやWorkflow間でやり取りできる
	- ビルドジョブにて、`./_site` に生成されたファイルを `actions/upload-artifact@v4` にてアップロードしている
    - デプロイジョブでは、それを `actions/download-artifact@v4` を使用してダウンロードしている
    - ジョブごとに仮想マシンが違うので、アーティファクトが必要 
    
```yaml
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: jekyll-site
          path: ./_site
          retention-days: 1
```

