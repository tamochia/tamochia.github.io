---
title: "脱Node.jsにてRails 7.1(Propshaft ＋ Import Maps ＋ DartSass)でBootstrapを利用する"
date: 2024-03-22
categories: 
  - "ruby"
tags: 
  - "rails"
---

## はじめに

Railsにてアセット（画像、CSS、JavaScriptなど）を管理するアセットパイプライン機能、こちらは、Railsのバージョンが上がると共に色々と変わっていって、それに対応するのが大変なところです。

Rails 5 までは **Sprockets**、Rails 6 からは **Webpacker**、そして、Rails 7 ではその [Webpackerも開発がストップしてしまい、標準では組み込まれなくなりました。](https://world.hey.com/dhh/rails-7-will-have-three-great-answers-to-javascript-in-2021-8d68191b) なので、Rails 7.1 では、また Sprockets が標準的アセット管理 gem として復活しています。

これまで、JavaScriptのモジュールを管理するために、別途 npm や yarn を入れて、そのために Node.js も動かさなくてはならないなど、Ruby on Rails なのに、なんで Node.js 必要なん、てな感じで、ちょっと Webpacker を毛嫌っていたところがあったのです。

アセットパイプラインの役目としては、大きく以下の4つになると思います。

- ファイル名へのダイジェストハッシュの付加
- 複数のJSやCSSファイルのバンドリング（連結）及び圧縮（最小化）
- JS及びCSS上位言語のコンパイル（トランスパイル）
- 公開フォルダ public への配置

Rails 7 より、**importmap-rails** が標準になったことにより、複数のJSやCSSファイルをバンドリングして1つのファイルにする必要がなくなりました。背景としては、HTTP/2、ES6/ESM があるのですが、これにより、アセットパイプラインがよりシンプルになったと思います。

今回は、Sprokets に代わる新しいアセットパイプラインとして、今後標準的な存在になるであろう **Propshaft** を使用し、**Node.js に依存しない環境を構築**してみたいと思います。

Propshaft は、Sprokets よりもシンプルで、トランスパイルなどの機能がありません。なので、TypeScript や JSX を使用される場合は、別途ビルドツールが必要であり、その際は、importmap-rails ではなく、jsbundling-rails gem (esbuild) を導入することになりますが、そうなると Node.js も導入しなければならなくなってしまいます。

私は、Bootstrap や JQuery があれば良いだけのレガシー指向なので、React や Vue でバリバリフロントエンド（クライアントサイド）を構築したい場合は、いっそのことアセットパイプラインは排除して、APIモード、つまりバックエンド専用でRailsを動かした方が良いかと思います。

前置きが長くなりましたが、scss をコンパイルする Sass は使用したいので、これも Node.js を必要としない Dartで書かれた **dartsass-rails** を採用したいと思います。

それでは、Node.js に依存しない、次の gem ファイルを使用して、Bootstrap が利用できるまでやってみたいと思います。

- **dartsass-rails** … scss のトランスパイル(Sass)として
- **importmap-rails** … CSSやJSのバンドリング
- **propshaft** … ファイル名のダイジェストハッシュ付加、公開ディレクトリへの配置等

【参考】 [Rails 7: importmap-rails gem README（翻訳）｜TechRacho by BPS株式会社](https://techracho.bpsinc.jp/hachi8833/2025_01_23/112183)

## 最初の環境構築

Rails アプリプロジェクト用のディレクトリを作成し、そこに cd しておきます。ここでは、例として「`sample_app`」としています。

```console
$ mkdir sample_app
$ cd sample_app
$ touch Gemfile
```

当方の bundler の環境はこちらです。今回 Ruby の Docker イメージコンテナ上で構築しますので、BUNDLE_PATH が、`/usr/local/bundle` になります。別に、`vendor/bundle` でも良いと思います。

```console
$ bundle config
Settings are listed in order of priority. The top value will be used.
app_config
Set via BUNDLE_APP_CONFIG: "/usr/local/bundle"
:
```

いつものように、**Gemfile** に rails gem を指定しておきます。

**【 Gemfile 】**

```ruby
source "https://rubygems.org"
gem "rails", "~> 7.1.1"
```

「`rails new`」コマンドで特にオプションを指定しないと、

- sprockets gem
- sprockets-rails gem

の gem が入り、アセットパイプラインとして、Sprockets が採用されます。

今回は、Propshaft を採用しますので、オプションとして「**`-a propshaft`**」を加えます。

```console
$ bundle exec rails new . -a propshaft
```

そうすると、sprockets、sprockets-rails などの gem は導入されず、代わりに propshaft gem が導入されます。

次に、DartSass (dartsass-rails) と、foreman を導入させるために、**Gemfile**に記述します。dartsass-rails は foreman を必要とします。

**【 Gemfile 】**

```ruby
:
gem "dartsass-rails"
gem "foreman"
:
```

では、「**`bundle install`**」を実行します。

```console
$ bundle install
```

続いて、dartsass 環境を導入するために、「**`bin/rails dartsass:install`**」を実行します。

```console
$ bin/rails dartsass:install
```

実行すると、次のようなことが行われます。

- ディレクトリ `app/assets/builds` が生成
- `app/assets/stylesheets/application.scss` が生成
- `Procfile.dev` が生成

続いて、importmap 環境を導入するために、「**`bin/rails importmap:install`**」を実行します。

```console
$ bin/rails importmap:install
```

実行すると、次のようなことが行われます。

- `app/views/layouts/application.html.erb` に、タグ「`<%= javascript_importmap_tags %>`」が挿入
- `app/javascript/application.js` が生成
- pin によってダウンロードしたJSが格納されるディレクトリ `vendor/javascript` が生成
- `config/importmap.rb` が生成

では、プロジェクトディレクトリに生成された **Procfile.dev** にちょっと修正を入れます。

**【 Procfile.dev 】**

```ruby
web: bin/rails server -b 0.0.0.0 -p 3000
css: bin/rails dartsass:watch
```

ブラウザで「http://localhost:3000」でアクセスした際に、「_localhost からデータが送信されませんでした。(ERR_EMPTY_RESPONSE)_」とエラー画面になってしまう場合は、「**`-b 0.0.0.0`**」を追記を追記しておきます。

Rails用データベース（SQLite）の作成のため、「**`bin/rails db:create`**」を実行します。

```console
$ bin/rails db:create
Created database 'storage/development.sqlite3'
Created database 'storage/test.sqlite3'
```

では、やっと Rails の起動です。

```console
$ bin/dev
```

そこで、ログを確認すると。。。

```
:
Cannot render console from 10.0.13.1! Allowed networks: 127.0.0.0/127.255.255.255, ::1
:
```

ログの中に、`Cannot render console from 10.0.13.1! Allowed networks: 127.0.0.0/127.255.255.255, ::1` と出ていますが、これは単に、`127.0.0.0/8` 以外の許可されていないネットワークからアクセスされたということなので、このクライアントのIPアドレス `10.0.13.1` を許可してあげれば良いだけです。

**【 config/environments/development.rb 】**

```ruby
Rails.application.configure do
  # Settings specified here will take precedence over those in config/application.rb.
  :
  config.web_console.allowed_ips = '10.0.0.0/8'
end
```

railsの再起動、「`bin/dev`」の実行が必要です。

![](/assets/images/ed3cbf7d6c02613e9db0273f0f3fcb84.png?w=861)

とりあえず、この画面が表示されたらひとまずOKです。

## 画像の掲載

Webページに任意の画像を掲載してみましょう。

```console
$ bin/rails g controller hello index
```

とりあえず、「`rails generator controller`」コマンドにて、適当にコントローラクラス「`HelloController (hello_controller.rb)`」とそれに紐づくビュー「`Index (index.html.erb)`」を作成してみます。

```console
$ bin/rails g controller hello index
      create  app/controllers/hello_controller.rb
       route  get 'hello/index'
      invoke  erb
      create    app/views/hello
      create    app/views/hello/index.html.erb
      invoke  test_unit
      create    test/controllers/hello_controller_test.rb
      invoke  helper
      create    app/helpers/hello_helper.rb
      invoke    test_unit
```

ブラウザにて「http://localhost:3000/hello/index」にアクセスすると、次のような画面が出てきます。

![](/assets/images/c8dd2d5ea166cfbec8e002575ba4c209.png?w=618)

例えば、この殺風景なWebページに、猫ちゃんの写真を掲載してみたいと思います。  
画像ファイル（ここでは「`cat_m.jpg`」）を、`app/assets/images` にアップロードしておきます。

```console
$ ls app/assets/images/
cat_m.jpg
```

そして、ビューファイル `app/views/hello/index.html.erb` に、`image_tag` にて画像ファイルを指定します。

**【 app/views/hello/index.html.erb 】**

```xml
<h1>Hello#index</h1>
<p>Find me in app/views/hello/index.html.erb</p>

<div><%= image_tag 'cat_m.jpg', :width => '200' %></div>
```

ここで、「http://localhost:3000/hello/index」にアクセスすると、画像が表示されるのが確認できます。

![](/assets/images/f5c1fc2f5fdb52512a452ff34d2c8b05.png?w=651)

ブラウザにてページのソースを確認するとこんな感じ。

```html
:
  <body>
    <h1>Hello#index</h1>
<p>Find me in app/views/hello/index.html.erb</p>

<div><img width="300" src="/assets/cat_m-c2d44131b7f1ae479fd3a90ff694376df3b103b6.jpg" /></div>
:
```

アセットである画像ファイル名がダイジェストハッシュ付きになっているのが確認できます。

## CSS (application.css) の変更

一番大元のCSSである「application.css」を変更すると、それがRailsアプリのすべてのWebページに適用されます。  
といっても「application.css」ファイルを直接編集するのではなく、`.scss` ファイルの方「application.scss」を編集し、DartSass によって、`.scss` から `.css` に変換します。ここでは、任意の名前の `.scss` ファイル「_main.scss_」を作成して、それを `@use` にて「application.scss」に読み込ませるようにします。

例えば、その「_main.scss_」にて、こんな感じで `H1` タグに対し、少し緑かかった色 (`#33aa99`) を設定してみます。

**【 app/assets/stylesheets/main.scss 】**

```css
h1 {
    color: #33aa99;
}
```

そして、その「_main.scss_」を「application.scss」に読み込ませる設定をします。

**【 app/assets/stylesheets/application.scss 】**

```css
// Sassy
@use "main";
```

では、再度「http://localhost:3000/hello/index」にアクセスしてみますと。。。

![](/assets/images/299a41af25cf8183d2bd64546745ea07.png?w=636)

反映されました！

実際に、「application.css」の中を見てみますと、ちょっと圧縮された形のCSSファイルになっているのがわかります。

**【 app/assets/builds/application.css 】**

```css
h1{color:#3a9}
```

## Bootstrap (CSS) の適用

Rails で Bootstrap と言えば、bootstrap gem がありますが、この gem を bundle install した場合、今回の環境だと、「`ExecJS::RuntimeUnavailable: Could not find a JavaScript runtime.`」とエラーが出て失敗します。ExceJS が必要、つまり Node.js が必要になってきます。

まずは、Bootstrap も何も導入しない状態で、index.html.erb ファイルの方に、次のサイトからBootstrapを使ったサンプルソースを拝借して追記してみます。

【引用元】 [Dropdowns (ドロップダウン) · Bootstrap v5.0](https://getbootstrap.jp/docs/5.0/components/dropdowns/)

**【 app/views/hello/index.html.erb 】**

```xml
<div class="container">

<h1>Hello#index</h1>
<p>Find me in app/views/hello/index.html.erb</p>

<div><%= image_tag 'cat_m.jpg', :width => '300' %></div>

<br />

<div class="dropdown">
  <button class="btn btn-primary dropdown-toggle" type="button" id="dropdownMenuButton1" data-bs-toggle="dropdown" aria-expanded="false">
    Dropdown button
  </button>
  <ul class="dropdown-menu" aria-labelledby="dropdownMenuButton1">
    <li><a class="dropdown-item" href="#">Action</a></li>
    <li><a class="dropdown-item" href="#">Another action</a></li>
    <li><a class="dropdown-item" href="#">Something else here</a></li>
  </ul>
</div>

</div>
```

ブラウザにて「http://localhost:3000/hello/index」にアクセスしてみますと。。。

![](/assets/images/715ee5ac67f57eb566d2c393a5a96b7a.png?w=626)

単に、普通のボタンに普通のリンクが作成されています。

まずは、Bootstrap の CSS (**bootstrap.min.css**) をサイトからダウンロードしてきて、これを先程「_main.scss_」ファイルを置いた同じ場所 **`app/assets/stylesheets`** に置きましょう。今回ダウンロードは **wget** コマンドで行ってみます。

```console
$ cd app/assets/stylesheets
$ wget -nv https://raw.githubusercontent.com/twbs/bootstrap/main/dist/css/bootstrap.min.css
$ ls
application.css  application.scss  bootstrap.min.css  main.scss
```

そして、この「bootstrap.min.css」を「application.scss」に読み込ませるために、「application.scss」に次のように追記します。

**【 app/assets/stylesheets/application.scss 】**

```css
// Sassy
@use "bootstrap.min";
@use "main";
```

ここで、ブラウザで確認してみます。

![](/assets/images/9e88dd451ab64fb8eadf52914a69a3eb.png?w=656)

CSSは問題なく適用されましたが、JSが無いのでボタンをクリックしてもドロップダウンが有効になっていません。

ブラウザにて、ページのソースを確認してみますと、CSSファイルは、「`/assets/application-5bc445f7...37737dd2a41.css`」というようなダイジェストハッシュ付のファイル名になっています。

![](/assets/images/891d9e8e7208ed02bf9a5cd9919d43a3.png?w=916)

このCSSファイルのリンクをクリックすると、application.css の中身を確認することができます。

```css
/*!
* Bootstrap  v5.3.3 (https://getbootstrap.com/)
* Copyright 2011-2024 The Bootstrap Authors
* Licensed under MIT (https://github.com/twbs/bootstrap/blob/main/LICENSE)
*/:root,[data-bs-theme=light]{--bs-blue:#0d6efd;--bs-indigo:#6610f2;--bs-purple
:#6f42c1;--bs-pink:#d63384;--bs-red:#dc3545;--bs-orange:#fd7e14;--bs-yellow
:#ffc107;--bs-gr
:
flex{display:flex !important}.d-print-inline-flex{display:inline-flex !important}
.d-print-none{display:none !important}}h1{color:#3a9}    
```

↑「bootstrap.min.css」と「main.scss」の内容が結合されているのが分かります。

## Bootstrap (JS) の適用

JS（JavaScript）モジュールファイルについては、次のような流れ、

1. `bin/importmap pin` コマンドにて、必要なJSモジュールを `vendor/javascript` 配下にダウンロードする

3. `config/importmap.rb` ファイルに、ダウンロードしたファイル名と、ソースで利用する名前とをマッピングする

5. `app/javascript/application.js` ファイルに、`import` 文にて、上の2で設定した名前を指定する

で適用させる形が良いのですが、どうもこの「`importmap pin`」コマンドで取ってくるファイルが何を基準で取ってくるのかが良く分からないです。

上手くいった方法としては、上の1で、「`importmap pin`」コマンドではなく、別途 **wget とかでCDNサイトからダウンロードしたファイルを `vendor/javascript` に置く**というやり方です。

```console
$ cd vendor/javascript
$ wget -nv https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.min.js
$ wget -nv https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.min.js.map
$ wget -nv https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.8/dist/umd/popper.min.js
$ wget -nv https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.8/dist/umd/popper.min.js.map
$ ls
bootstrap.min.js  bootstrap.min.js.map  popper.min.js  popper.min.js.map
```

次に、 **`config/importmap.rb`** を編集します。

**【 config/importmap.rb】**

```ruby
# Pin npm packages by running ./bin/importmap

pin "application"
pin "@popperjs/core", to: "popper.min.js"
pin "bootstrap", to: "bootstrap.min.js"
```

ここで、「`to: "bootstrap.min.js"`」のファイル名の部分をCDNのURLにすると、ダウンロード済みのJSファイルではなく、CDNサイトを直接参照するようになります。

最後、 **`app/javascript/application.js`** を編集します。

**【 app/javascript/application.js 】**

```jscript
// Configure your import map in config/importmap.rb. Read more: https://github.com/rails/importmap-rails

import "@popperjs/core";
import "bootstrap";
```

では、ブラウザで確認してみます。

![](/assets/images/55af70b60d85ec786d99e5765084e384.png?w=646)

うまくドロップダウンできましたね！

ページのソースを見るとこんな感じ。ちゃんとJSファイルが読み込まれてますね。

```js
: 
   <script type="importmap" data-turbo-track="reload">{
  "imports": {
    "application": "/assets/application-98b46e9272...c35a810dbef48b91e.js",
    "@popperjs/core": "/assets/popper.min-b85c89d...c27ad6caf0f7c986a7c95.js",
    "bootstrap": "/assets/bootstrap.min-e3d895...fc460489eed2adfa64b4.js"
  }
}</script>
<link rel="modulepreload" href="/assets/application-98b46e927...c35a810dbef48b91e.js">
<link rel="modulepreload" href="/assets/popper.min-b85c89d...c27ad6caf0f7c986a7c95.js">
<link rel="modulepreload" href="/assets/bootstrap.min-e3d895...fc460489eed2adfa64b4.js">
<script type="module">import "application"</script>
:
```

## 補足1: bin/importmap コマンド

今回は使用しないで行いましたが、「`bin/importmap`」コマンドについても触れたいと思います。

「`bin/importmap pin`」コマンドにて、bootstrap.js をダウンロードするには次のようなコマンドをたたきます。

```console
$ bin/importmap pin bootstrap@5.3.3
```

結果は次のとおり、bootstrap と一緒に @popperjs/core もダウンロードされます。

```console
$ bin/importmap pin bootstrap@5.3.3
Pinning "bootstrap" to vendor/javascript/bootstrap.js via download from https://ga.jspm.io/npm:bootstrap@5.3.3/dist/js/bootstrap.esm.js
Pinning "@popperjs/core" to vendor/javascript/@popperjs/core.js via download from https://ga.jspm.io/npm:@popperjs/core@2.11.8/lib/index.js
```

ちなみに、「`bin/importmap pin bootstrap`」で実行した場合、「`Couldn't find any packages in ["bootstrap"] on jspm`」と出てダウンロードされない場合があります。その場合は、パッケージの指定を「`bootstrap@5.3.3`」のようにバージョンも含めた形で行うとダウンロードしてくれると思います。または、`from` オプションにて「`--from unpkg`」や「`--from jsdelivr`」などとCDNサイトを指定してみると良いです。

また、あるサイトには、「`--download`」オプションを付ければ良いみたいなことが書いてあったのですが、2024年3月時点では、このオプションを付加した場合、「`--download`」自体をパッケージ名として認識されてしまうようです。help を見てみます。

```console
$ bin/importmap --help
Commands:
  importmap audit              # Run a security audit
  importmap help [COMMAND]     # Describe available commands or one specific command
  importmap json               # Show the full importmap in json
  importmap outdated           # Check for outdated packages
  importmap packages           # Print out packages with version numbers
  importmap pin [*PACKAGES]    # Pin new packages
  importmap unpin [*PACKAGES]  # Unpin existing packages
  importmap update             # Update outdated package pins
```

このコマンドによって、JSファイルは、 `vendor/javascript/` 配下に保存されます。

```console
$ ls vendor/javascript/
@popperjs--core.js  bootstrap.js
```

併せて、 `config/importmap.rb` にも自動的に追記されています。

**【 config/importmap.rb 】**

```ruby
# Pin npm packages by running ./bin/importmap

pin "application"
pin "bootstrap" # @5.3.3
pin "@popperjs/core", to: "@popperjs--core.js" # @2.11.8
```

「`bin/importmap json`」コマンドにて、`<script type="importmap">` タグ内に記載されるJSONも確認できます。きちんとファイル名にダイジェストが付加されています。

```console
$ bin/importmap json
{
  "imports": {
    "application": "/assets/application-d8a8613a4adb2b058d6ae5ddc9d0114a5ab5dc2e.js",
    "bootstrap": "/assets/bootstrap-e25ad8d55dd45b21bf9f74f9cebdb21d9680d9a0.js",
    "@popperjs/core": "/assets/@popperjs--core-d968fa6b3a2fc168fa27f46456beea72c2c02cec.js"
  }
}
```

しかし、「`@popperjs/core`」が参照しているソース「`vendor/javascript/@popperjs--core.js`」の中身をみると、「`from"../_/a0ba12d2.js"`」などの記述があり、別の場所のJSを参照しているようで、これではダメダメ。。。

**【 vendor/javascript/@popperjs--core.js 】**

```javascript
export{afterMain,afterRead,afterWrite,auto,basePlacements,
beforeMain,beforeRead,beforeWrite,bottom,clippingParents,
end,left,main,modifierPhases,placements,popper,read,reference,
right,start,top,variationPlacements,viewport,write}from"./enums.js";
import"./modifiers/index.js";export{c as createPopperBase,
p as popperGenerator}from"../_/a0ba12d2.js";export{createPopper}from"./popper.js";
export{createPopper as createPopperLite}from"./popper-lite.js";
export{default as detectOverflow}from"./utils/detectOverflow.js";
export{default as applyStyles}from"./modifiers/applyStyles.js";
export{default as arrow}from"./modifiers/arrow.js";
export{default as computeStyles}from"./modifiers/computeStyles.js";
export{default as eventListeners}from"./modifiers/eventListeners.js";
export{default as flip}from"./modifiers/flip.js";export{default as hide}from"./modifiers/hide.js";e...
:
```

やっぱりなんか上手くいかないんです。。。

なので、今回は直接CDNサイトから参照し、wget でダウンロードしてきたというわけです。

## 補足2: assets:precompile の影響

Propshaft を使用した場合の注意として、development 環境では、「`assets:precompile`」を実行してしまうと、それ以降アセットの変更の反映がされなくなるようです。

【参考】 Rails: Sprockets->Propshaftアップグレードガイド（翻訳）｜TechRacho by BPS株式会社

> Propshaftは、developmentモードで動的アセットリゾルバを使います。ただしローカルでassets:precompileを実行すると、Propshaftが静的アセットリゾルバを使うように切り替わります。そうなるとアセットの変更が反映されなくなるので、アセットを変更するたびにアセットのプリコンパイルを実行しなければならなくなります。ここがSprocketsと異なる点です。
> 
> 動的なアセットリゾルバを再度有効にしたい場合は、ターゲットフォルダ（通常はpublic/assets）を削除する必要があります。これにより、propshaftがソースからの動的コンテンツの配信を開始するようになります。

「`assets:precompile`」を実行してしまうと、`Rails.application.config.assets.paths` で参照されるすべてのアセットが、「`public/assets`」に、ダイジェストハッシュを付与したファイル名でコピーされます。そうなると、それ以降、こちらのファイルを参照してしまいますので、元のJSファイルをいじったところで変化がありません。

その場合の回避策として、 `public/assets` の中身をクリアすると良いです。次のコマンドを実行します。

```console
$ bin/rails assets:clobber
```

【参考】 [【Tips】ローカルで `rails assets:precompile` を実行してしまい以後アセットの更新が反映されなくなってしまった場合の対応 #Ruby - Qiita](https://techracho.bpsinc.jp/hachi8833/2024_02_18/119856)

そして、その後 `bin/dev` で再起動です。

どうでしょうか、アセットパイプライン辺りがシンプルになって見通しがよくなって、しかも動作が軽くなったような気がするのは私だけでしょうか。これからもなるべく Ruby on Rails の流儀に従って開発していきたいです。
