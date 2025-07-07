---
title: "Rails7＋MySQL9の開発環境をDockerで構築する"
date: 2024-11-10
categories: 
  - "ruby"
tags: 
  - "linux"
  - "docker"
  - "rails"
---

# はじめに

RailsでのアプリケーションをDockerコンテナ上で開発するための環境構築について、これまでも色々記事にしてきました。

今回がその集大成かなと思います。

パターンとしては、

- 既に rails new にてひな形が作成された状態でのコンテナイメージを利用
- Ruby及びBundlerが実行可能なコンテナイメージを準備し、コンテナ上でrails newを実行

毎回同じパターンで開発するなら前者でも良いですが、一からRails環境を構築したいなら後者になります。その場合、どうしても面倒な手順が必要なので、今回それをまとめたいと思います。

今回もWebサービスとしてWebコンテナ、データベースサービスとてDBコンテナ、2つのコンテナを Docker Composeにて定義して実行させます。

今回の環境は、Ruby 3.3、MySQL 9.1、そしてRailsはVersion 7.2 で、WebpackやNode.js（yarn）を使わず、代わりに以下のスタックを採用します。

- Propshaft - アセットパイプラインを処理 (ファイル名のダイジェストハッシュ、パブリックディレクトリのデプロイ)
- Importmap - バンドルせずにJavaScriptとCSSの依存関係を管理
- DartSass(dartsass-rails) - SCSS (Sass)のトランスパイルツール
- Tailwind CSS - CSSフレームワーク

なお、GitHubにもGitリポジトリとしてUPしていますので、ここで説明しているファイルはすべて入手可能です。  

[tamochia/rails7-docker: Rails 7 application development environment on Docker containers](https://github.com/tamochia/rails7-docker)

まずは、作業用のディレクトリを作成します。

```console
$ mkdir rails7-docker/web
$ mkdir -p rails7-docker/db/sql
$ cd rails7-docker/
```

# ディレクトリ・ファイル構成

あらかじめ準備しておくディレクトリとファイルのツリー構成は次のとおりです。

```console
$ tree -na
.
├── .env
├── compose.yaml
├── db
│   ├── Dockerfile
│   └── sql
│       └── 00_init.sql
└── web
    ├── Dockerfile
    ├── Gemfile
    └── Gemfile.lock
```

## .env

Docker Compose の定義ファイル `compose.yaml` にて参照する環境変数の定義を格納するためのファイルです。  
また、`compose.yaml` の `build: > args:` 経由で、`Dockerfile` の **`ARG`** で指定している環境変数も定義します。

{% highlight bash linenos %}
MYSQL_ROOT_PASSWORD=hogehoge
RAILS_APP_PATH=/sample_app
RUN_USER=woody
RUN_UID=1000
{% endhighlight %}

各環境変数について

- `MYSQL_ROOT_PASSWORD` : MySQLのrootのパスワード
- `RAILS_APP_PATH` : `WORKDIR` で指定するRailsプロジェクトのディレクトリパス
- `RUN_USER` : Dockerコンテナ内での実行ユーザのID（ユーザ名）
- `RUN_UID` : Dockerコンテナ内での実行ユーザのUID

ホストユーザのホームディレクトリ配下でバインドマウントさせる場合は、そのホストユーザの権限でコンテナを動かした方が何かと都合が良いです。

## compose.yaml

以前は `docker-compose.yml` というファイル名が一般的に使われていましたが、現在はこちらの名前になっています。

- dbサービス
    - `db/Dockerfile` から構築したイメージを使用
    - DBデータを格納するディレクトリ `/var/lib/mysql` をボリュームとして定義
    - 文字コードセットとして `utf8mb4` を指定
- webサービス
    - `web/Dockerfile` から構築したイメージを使用
    - `web` ディレクトリ配下を `web/Dockerfile` にて WORKDIR に指定したRailsプロジェクトディレクトリである `/sample_app` にバインド
    - Bundlerによってインストールされたgemファイルを格納するディレクトリ `/usr/local/bundle` をボリュームとして使用
    - foreman を使用するため、Railsの実行は `bin/dev` コマンドを使用
    - `args:` は、`.env` から受け取った環境変数の定義を、`web/Dockerfile` に受け渡すために記述

{% highlight yaml linenos %}
services:
  db:
    build:
      context: ./db
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      TZ: "Asia/Tokyo"
    volumes:
      - mysql_volume:/var/lib/mysql
    ports:
      - '3306:3306'
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci

  web:
    build:
      context: ./web
      args:
        RAILS_APP_PATH: ${RAILS_APP_PATH}
        RUN_USER: ${RUN_USER}
        RUN_UID: ${RUN_UID}
    command: bash -c "rm -f tmp/pids/server.pid && bin/dev"
    volumes:
      - ./web:${RAILS_APP_PATH}
      - bundle_volume:/usr/local/bundle
    ports:
      - 3000:3000
    stdin_open: true
    tty: true
    depends_on:
      - db
volumes:
  mysql_volume:
  bundle_volume:
{% endhighlight %} 

## db/Dockerfile

dbサービスのDockerイメージをビルドするための設定ファイルです。MySQLはバージョン9.1を採用してみます。 `/docker-entrypoint-initdb.d` はコンテナ初回起動時（`/var/lib/mysql` にデータが無い場合）に実行するSQLファイルを格納するディレクトリです。`sql` フォルダにあるSQLファイルがコンテナにコピーされます。今回は、すべてのデータベースにフルアクセス可能なDBユーザ「dbadm」を作成するSQLファイルを用意しています。

```bash
FROM mysql:9.1
COPY sql/* /docker-entrypoint-initdb.d/
```

## db/sql/00_init.sql

{% highlight sql linenos %}
CREATE USER 'dbadm'@'%' IDENTIFIED BY 'hogehoge';
GRANT ALL PRIVILEGES ON *.* TO 'dbadm'@'%';
FLUSH PRIVILEGES;
{% endhighlight %}

## web/Dockerfile

WebサービスのDockerイメージをビルドするための設定ファイルです。Rubyはバージョン3.3を採用してみます。

- `WORKDIR /sample_app` : Railsプロジェクトディレクトリ名を「sample\_app」に指定（任意）

- `ENV RUN_UID=$RUN_UID RUN_USER=$RUN_USER` : 「 `.env`」で指定した環境変数の値（現在のホストのUID `1000` とユーザ名 `woody` が新たにDockerfileで使用する環境変数 `RUN_UID/RUN_USER` として指定
    - `RUN useradd` と `chown` にて、そのユーザ作成と権限を与え、`USER $RUN_USER:$RUN_USER` にてコンテナ実行ユーザを指定しています。

- `CMD ["/bin/bash"]` : 通常、ここでは rails を動かすようなコマンドを記述しますが、railsサービスの起動は docker compose （`compose.yaml`にて指定）で行うので、とりあえず bash が動いていれば良いとします。

{% highlight docker linenos %}
FROM docker.io/library/ruby:3.3
ARG RAILS_APP_PATH RUN_USER RUN_UID
WORKDIR $RAILS_APP_PATH
ENV TZ=Asia/Tokyo RUN_UID=$RUN_UID RUN_USER=$RUN_USER
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential default-libmysqlclient-dev default-mysql-client pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .
RUN useradd -u $RUN_UID $RUN_USER --create-home --shell /bin/bash && \
    chown -R $RUN_USER:$RUN_USER .
USER $RUN_USER:$RUN_USER
CMD ["/bin/bash"]
{% endhighlight %}

## web/Gemfile

`rails new` を実行するには rails gem をまずインストールする必要があるので、`Gemfile` に rails gem の指定だけします。rails gem は、最初の `RUN bundle install` によってコンテナ上でインストールされます。

{% highlight ruby linenos %}
source 'https://rubygems.org'
gem 'rails', '~> 7.2'
{% endhighlight %}

# 構築

## 最初のビルド

コンテナイメージビルドの際に、`COPY Gemfile Gemfile.lock ./` にて、 `Gemfile.lock` を `WORKDIR` で指定してる `/sample` コピーするので、予め空の `Gemfile.lock` を作成しておきます。

```console
$ touch web/Gemfile.lock
```

では、ビルドを実行してみます。

```console
$ docker compose build
```

特に面倒なことをしていないので、問題なく完了すると思います。  
`docker images` コマンドにて生成されたDockerイメージを確認します。

```console
$ docker images
REPOSITORY           TAG       IMAGE ID       CREATED       SIZE
rails72-docker-web   latest    1c4f6f2d0d3a   6 hours ago   1.06GB
rails72-docker-db    latest    5d948447db56   2 weeks ago   602MB
```

## rails newコマンドの実行

rails new を実行してしまうと、web/Dockerfile を**書き換えてしまう**ので、現在の Dockerfile を退避させておきます。

```console
$ cp -p web/Dockerfile web/Dockerfile.bak
```

`rails new` コマンドの実行ですが、Bundler経由にて行います。

- `--a propshaft` : アセットパイプラインとしてデフォルトのSprocketsではなくPropshaftを使用
- `--d mysql` : データベースはMySQLを使用
- `--css tailwind` : CSSフレームワークは Tailwind CSS を使用
- `--skip-git` : Gitはその上のディレクトリ（現在のディレクトリ配下）で管理するので不要
- `--skip-decrypted-diffs` : Gitを使用しない場合、これも指定しろと言ってくる
- `--skip-bundle` : Gemのインストールは後でするので、`bundle install` は走らせない

```console
$ docker compose run --rm web bundle exec rails new . -a propshaft -d mysql --css tailwind --force --no-deps --skip-bundle --skip-git --skip-decrypted-diffs
```

`docker compose run` 実行するとボリュームも作成されます。またdbのコンテナも実行されるようです。

```
 ✔ Network rails72-docker_default         Created                 0.2s
 ✔ Volume "rails72-docker_bundle_volume"  Created                 0.0s
 ✔ Volume "rails72-docker_mysql_volume"   Created                 0.0s
 ✔ Container rails72-docker-db-1          Created                 0.3s
[+] Running 1/1
 ✔ Container rails72-docker-db-1  Started                         1.4s
:
      create  app/assets/stylesheets/application.css
      remove  config/initializers/cors.rb
      remove  config/initializers/new_framework_defaults_7_2.rb
```

これにより `Gemfile` と `Gemfile.lock` が書き換わり、webディレクトリ配下にRailsプロジェクトのひな形ファイル群が作成されます。

生成されたボリュームを確認してみます。

```console
$ docker volume ls
DRIVER    VOLUME NAME
local     rails72-docker_bundle_volume
local     rails72-docker_mysql_volume
```

webサービス側のボリューム `rails72-docker_bundle_volume` のマウント先 `/usr/local/bundle` 内の gems ディレクトリ内を確認してみます。

```console
$ docker compose run --rm web ls -l /usr/local/bundle/gems
[+] Creating 1/0
 ✔ Container rails72-docker-db-1  Running                                                                   0.0s
total 232
drwxr-xr-x 4 root root 4096 Nov  5 15:46 actioncable-7.2.2
drwxr-xr-x 6 root root 4096 Nov  5 15:46 actionmailbox-7.2.2
drwxr-xr-x 3 root root 4096 Nov  5 15:46 actionmailer-7.2.2
:
```

`--skip-bundle` を指定しましたがどうやら既定のGemsはインストール済みのDockerイメージだったようです。

## Gemfileの修正と2回目のビルド

`rails new` を実行したことにより、Dockerfile も書き換えられます。自動生成された Dockerfile は、production 環境用なのでとりあえず使用しないで退避させて置き、元の退避させていた Dockerfile.bak を再度 Dockerfile として使用します。

```console
$ mv web/Dockerfile web/Dockerfile.prod
$ cp -p web/Dockerfile.bak web/Dockerfile
```

ここで、Gemfileに次の2文を追記します。

- `gem "dartsass-rails"`
- `gem "foreman"`

SASSとして DartSass を使用します。そのために foreman もインストールしておきます。

```diff
$ diff -u web/Gemfile.ORG web/Gemfile
--- web/Gemfile.ORG     2024-10-20 15:51:21.006990138 +0900
+++ web/Gemfile 2024-10-20 15:54:34.128788746 +0900
@@ -36,6 +36,9 @@
 # Use Active Storage variants [https://guides.rubyonrails.org/active_storage_overview.html#transforming-images]
 # gem "image_processing", "~> 1.2"

+gem "dartsass-rails"
+gem "foreman"
+
 group :development, :test do
   # See https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem
   gem "debug", platforms: %i[ mri windows ], require: "debug/prelude"
```

では Gemfile を修正したので、ここで2回目のビルドをします。

```console
$ docker compose build
```

ビルド中に `bundle install` が実行されますが、`docker compose build` ではボリュームは更新されないので `/usr/local/bundle` に新たに gem がインストールされません。なので、`docker compose run` にて `bundle install` を実行します。その際、`/usr/local/bundle` への書き込み権限が必要となるので、「`--user root`」を付けることを忘れずに。

```console
$ docker compose run --user root --rm web bundle install
```

## プロジェクトディレクトリ内の種々の設定

`bin/rails` コマンドにて、dartsass と importmap をインストール（プロジェクトで利用可能に）します。

```console
$ docker compose run --rm web bin/rails dartsass:install importmap:install
```

dartsassのインストールの際に、foreman gemを `/usr/local/bundle` 配下にインストールしようとして権限エラー（Gem::FilePermissionError）が表示されますが、先の `bundle install` 実行の際に foreman gem はインストールされているので放置して構いません。

次に、 `web/Procfile.dev` の修正をします。既存との diff は次のとおりです。`bin/rails server` コマンドにて「`-b 0.0.0.0`」を付加しています。これにより、ホストからrailsサーバにアクセスできるようになります。

```diff
$ diff -u web/Procfile.dev.ORG web/Procfile.dev
--- web/Procfile.dev.ORG       2024-10-18 15:33:03.018614176 +0900
+++ web/Procfile.dev    2024-10-18 15:35:50.541990523 +0900
@@ -1,2 +1,2 @@
-web: bin/rails server -p 3000
+web: bin/rails server -p 3000 -b 0.0.0.0
 css: bin/rails dartsass:watch
```

次に、`web/config/database.yml` の修正をします。RailsはDBに対し、root（MySQLの管理者）でアクセスさせますので、rootユーザのパスワードの指定と、MySQLが動いているサーバ名の指定「`host: db`」を行います。ここではサービス名である「`db`」を指定します。

```diff
$ diff -U 1 web/config/database.yml.ORG web/config/database.yml
--- web/config/database.yml.ORG 2024-10-18 14:16:21.416725409 +0900
+++ web/config/database.yml     2024-10-18 14:52:10.323273892 +0900
@@ -16,4 +16,4 @@
   username: root
- password:
- host: <%= ENV.fetch("DB_HOST") { "localhost" } %>
+  password: hogehoge
+  host: db
```

rootを指定する代わりに、`db/sql/00_init.sql` で作成したDBユーザ「`dbadm`」を指定しても良いです。`dbadm` も全てのデータベースに対してフルアクセス可能です。

次に、Railsアプリ用データベースを生成するために、別ターミナルにて dbサービスのコンテナを起動しておきます。

```console
$ cd rails7-docker
$ docker compose up db
[+] Running 1/0
 ✔ Container rails72-docker-db-1  Running                                                               0.0s
Attaching to db-1
```

元のターミナルに戻って、次のコマンドを実行します。

```console
$ docker compose run --rm web bin/rake db:create
```

このコマンドにより、「`sample_app_development`」と「`sample_app_test`」の2つのデータベースが生成されます。

```
[+] Creating 1/0
 ✔ Container rails72-docker-db-1  Running                                                                   0.0s
Created database 'sample_app_development'
Created database 'sample_app_test'
```

実際に、dbサービスのコンテナに対し、mysqlコマンドにてMySQLに接続してみます。

```console
$ docker compose exec db mysql -u root -p
Enter password: 《 MySQLのrootのパスワードを指定 》
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 9.1.0 MySQL Community Server - GPL
:
```

`show databases` コマンドで確認してみると、確かに2つのデータベースが存在しています。

```sql
mysql> show databases;
+------------------------+
| Database               |
+------------------------+
| information_schema     |
| mysql                  |
| performance_schema     |
| sample_app_development |
| sample_app_test        |
| sys                    |
+------------------------+

mysql> \q
```

ここで、別のターミナルで動いていたコンテナを **`[Ctrl] + [C]`** で停止しておきます。

```
:
Attaching to db-1
^CGracefully stopping... (press Ctrl+C again to force)
[+] Stopping 1/1
 ✔ Container rails72-docker-db-1  Stopped                                                               3.4s
canceled
```

## Railsの起動

コンテナ起動

```console
$ docker compose up
```

ブラウザにて https://localhost:3000 でアクセスしてみます。

![](/assets/images/6d35ce6a3a3e53419450b84d836d98bf.png){:width="1024px"}

ここで一旦 **`[Ctrl] + [C]`** でコンテナを終了

```
:
^CGracefully stopping... (press Ctrl+C again to force)
[+] Stopping 2/2
 ✔ Container rails72-docker-web-1  Stopped                 1.0s
 ✔ Container rails72-docker-db-1   Stopped                 0.9s
canceled
```

再度、今度は「**`-d`**」オプションを付けてバックグラウンドで実行します。

```console
$ docker compose up -d
[+] Running 2/2
 ✔ Container rails72-docker-db-1   Started                 1.1s
 ✔ Container rails72-docker-web-1  Started                 1.7s
$
```

何か1つページを作成してみます。HelloController クラスとその View ファイル `web/app/views/hello/index.html.erb` を生成させます。

```console
$ docker compose exec web bin/rails g controller hello index
      create  app/controllers/hello_controller.rb
       route  get "hello/index"
      invoke  tailwindcss
      create    app/views/hello
      create    app/views/hello/index.html.erb
:
```

とりあえずデフォルトのページ http://localhost:3000/hello/index を表示してみます。

![](/assets/images/44b2d02ca491c26c8eadc52a6d07b2de.png){:width="519px"}

## Tailwind CSS ＋ Alpine.js の実装

### Tailwind CSS の実装

とりあえず、Tailwind CSS をプロジェクト内で利用できるようにします。

```console
$ docker compose exec web bin/rails tailwindcss:install
       apply  /usr/local/bundle/gems/tailwindcss-rails-3.0.0/lib/install/tailwindcss.rb
  Add Tailwindcss include tags and container element in application layout
      insert    app/views/layouts/application.html.erb
  Build into app/assets/builds
       exist    app/assets/builds
   identical    app/assets/builds/.keep
  Add default config/tailwindcss.config.js
      create    config/tailwind.config.js
  Add default app/assets/stylesheets/application.tailwind.css
      create    app/assets/stylesheets/application.tailwind.css
      append    Procfile.dev
  Add bin/dev to start foreman
   identical    bin/dev
  Compile initial Tailwind build
         run    rails tailwindcss:build from "."

Rebuilding...

Done in 608ms.
         run  bundle install --quiet
```

`web/Procfile.dev` に、Tailwind CSS の CSS をリアルタイムに変更を反映してくれるよう3行目の部分が追加されました。

```ruby
web: bin/rails server -p 3000 -b 0.0.0.0
css: bin/rails dartsass:watch
css: bin/rails tailwindcss:watch
```

先程のページ http://localhost:3000/hello/index を再表示してみます。

![](/assets/images/0dd73003f6843fd260541f640c510b1e-1.png){:width="593px"}

先程の結果と変わり、マージンが増えたり、フォントも変わってしまいました。

とりあえずフォントの設定を元の状態に変えてみます。 `web/config/tailwind.config.css` の「fontFamily」の部分をコメントアウトします。

{% highlight ruby lineos %}
const defaultTheme = require('tailwindcss/defaultTheme')

module.exports = {
  content: [
    './public/*.html',
    './app/helpers/**/*.rb',
    './app/javascript/**/*.js',
    './app/views/**/*.{erb,haml,html,slim}'
  ],
  theme: {
    extend: {
    /*
      fontFamily: {
        sans: ['Inter var', ...defaultTheme.fontFamily.sans],
      },
    */
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('@tailwindcss/container-queries'),
  ]
}
{% endhighlight %}

config配下を修正したので、railsを再起動させます。手っ取り早くコンテナを再起動させます。

```console
$ docker compose stop
$ docker compose start
```

有効になっているかと思います。（分かりづらいですが。。。）

次に、妙にマージンが空いてしまっている部分を直します。

これは `web/app/views/layouts/application.html.erb` の

```xml
:
<main class="container mx-auto mt-28 px-5 flex"> 
:
```

の部分を修正します。とりあえず、`<main class="container mt-5 px-5">` とでもしておきます。これは rails を再起動させなくてもブラウザの再読み込みで変更を確認できます。

### Alpine.js の実装

Alpine.jsはNode.jsを必要としない軽量なJavaScriptフレームワークです。部分的にちょっとインタラクティブ要素を追加したいときに便利そうです。

Alpine.js は、Importmap（`bin/importmap pin`）を利用して読み込み使用しますが、その際に、alpine-turbo-drive-adapter なる Turbo (Turbolinks) とAlpine.jsが衝突しないよう解決するブリッジライブラリも一緒に読み込みます。

```console
$ docker compose exec web bin/importmap pin alpinejs
$ docker compose exec web bin/importmap pin alpine-turbo-drive-adapter
```

pin コマンドで読み込んだ JavaScript ライブラリは、`web/config/importmap.rb` に追加されます。

```ruby
# Pin npm packages by running ./bin/importmap

pin "application"
pin "alpinejs" # @3.14.3
pin "alpine-turbo-drive-adapter" # @2.1.0
```

実体は `web/vendor/javascript` 配下にあるようです。

```console
$ ls -lh web/vendor/javascript/
合計 60K
-rw-r--r-- 1 hoge hoge 3.4K 10月 26 14:15 alpine-turbo-drive-adapter.js
-rw-r--r-- 1 hoge hoge  55K 10月 26 14:14 alpinejs.js
```

Alpine.jsが使用できるよう `web/app/javascript/application.js` に次のように記載します。

{% highlight javascript %}
// Configure your import map in config/importmap.rb. Read more: https://github.com/rails/importmap-rails

import 'alpine-turbo-drive-adapter'
import Alpine from 'alpinejs'

window.Alpine = Alpine
Alpine.start()
{% endhighlight %}

### 実装確認

では、Tailwind CSS ＋ Alpine.js の実装を確認するために、`web/app/view/hello/index.html.erb` に次の div ブロックを追記してみます。

```xml
<div x-data="{view: false, res: 'Hello!'}" class="flex mt-2 mx-2">
  <button x-on:click="view = !view" class="bg-blue-500 text-white p-2 rounded">Toggle</button>
  <div x-show="view" class="p-2">
    <p x-text="res"></p>
  </div>
</div>
```

こちらの div ブロックは、「Toggle」ボタンを表示し、そのボタンをクリックするたびに「Hello!」という文字が現れたり消えたりします。 `x-data` にて定義した `view` という変数が、Toggleボタンをクリックするたび、`false` → `true` → `false` → … と繰り返し、 `x-show="view"` が `true` のときだけ、`<p x-text="res"></p>` のブロックが有効となり、変数 `res` に入っている文字列「Hello!」を表示する、という仕組みになっています。

![](/assets/images/518484f8f2d513b719e8f38a4a6005c5.png){:width="468px"}

今後は、もう少しTailwind CSS と Alpine.js を深堀していこうと思います。
