---
title: "Tailwind cssとAlpine.jsによるドロップダウンボタンの作成"
date: 2024-11-12
categories: 
  - "ruby"
tags: 
  - "css"
---

Tailwind cssとAlpine.jsによるドロップダウンボタンの作成

# はじめに

## Tailwind CSS について

Tailwind CSSは、Bootstrapと同様なCSSフレームワークの1つです。Bootstrapとの比較、メリットデメリットについては、他のブログサイトでも色々紹介されているのでそちらを参考にされたほうが良いと思います。

最初、私としては、クラス名の指定が長くなってしまい見づらくなってしまうのではという抵抗感がありました。あと、自分で色々デザインするのが面倒なので、ボタンは `btn-primary` などと既に決まっているスタイルを使用する Bootstrap の方が楽に思えたからです。

**Bootstrapの例**

```html
<button class="btn btn-primary">Hello!</button>
```

**Tailwind CSSの例**

```html
<button class="py-2 bg-blue-500 text-white w-40 rounded-md">Hello!</button>
```

そんな私が Tailwind CSS を使ってみようと思ったのは、Rails 7にてNode.jsやWebpackを使用しない方法で標準で使えるCSSが Tailwind CSS だったからです。もちろん Rails 7でも同様にBootstrapもimportmapを利用すればNode.jsなしに使用することは可能ですが、BootstrapのJavaScript機能を使用する際にExecJSを必要とし、ExecJSはNode.jsなどのJavaScriptランタイムを必要とするようです。

[Rails7でBootstrapをNode.jsを使わずに利用する - PICO LAB](https://picolab.dev/2022/03/09/rails7-bootstrap/)


## Alpine.js について

CSSフレームワークは Tailwind CSS に決めて、では「動作」の部分であるJavaScriptはどうするかという問題があります。定番のjQueryという選択もありますが、今回選んだのは Alpine.js です。

Alpine.jsは軽量なJavaScriptフレームワークで、HTMLファイルに直接読み込んで使用することが可能です。もちろんNode.jsを必要としません。書き方としては、Vue.jsやReactのような宣言的アプローチを採用しているらしいです。あと、Tailwind CSSとの相性も良いそうです。

部分的にちょっとインタラクティブにしたい、というような使い方の場合はこれでよさそうです。

- [初心者のためのAlpine.js #Alpine.js - Qiita](https://qiita.com/hakuisan/items/8a4797c4857cf4450249)
- [Alpine.js 5分で説明するよ #JavaScript - Qiita](https://qiita.com/kohashi/items/ae8b1186567ec34b2a75)

## 両者組合わせ

ということで、Tailwind CSS ＋ Alpine.js の組み合わせは最強ではないかと思うわけです。

とりあえず、どちらともCDNでサクッと読み込んで1つのHTMLファイルで完結できるので、簡単に動きを試してみることが可能です。

{% highlight html linenos %}
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Sample</title>
  <script src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js" defer></script>
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.x.x/dist/tailwind.min.css" rel="stylesheet">
</head>
<body>
:
{% endhighlight %}

# ドロップダウンボタンの作成

では、さっそく順を追って Tailwind CSS と Alpine.js によるドロップダウンボタンを作成してみたいと思います。

## CSSを使用していない状態

まずは、**CSSを一切使用していない状態**から作成してみます。

{% highlight html %}
<div>
  <div>Please choice.</div>
  <button>Dropdown button</button>
  <div>
    <a href="#">Apple</a>
    <a href="#">Banana</a>
    <a href="#">Orange</a>
  </div>
</div>
{% endhighlight %}

**【実行例】**

![](/assets/images/ss2024-11-12_122336.png?w=252)

「`Dropdown button`」のところがボタンで、「`Apple`」、「`Banana`」、「`Orange`」が選択肢となるようなリストを作成し、ユーザが選んだ結果が「`Please choice.`」のところに表示されるようにしていきます。

## Tailwind CSSでposition設定

それでは、Tailwind CSS を使用してみます。まずは各エレメントのpositionの指定から、 **`absolute`** と **`block`** を使用してボタンの下に候補リストが表示されるようにします。

{% highlight html %}
<div>
  <div class="py-2">Please choice.</div>
  <button class="py-2">Dropdown button</button>
  <div class="absolute w-48">
    <a href="#" class="block py-2">Apple</a>
    <a href="#" class="block py-2">Banana</a>
    <a href="#" class="block py-2">Orange</a>
  </div>
</div>
{% endhighlight %}

**【実行例】**

![](/assets/images/860f70f8543de4900008fa9cc506dcc0.png?w=261)

## さらに細かいTailwind CSS設定

次に、ボタンの背景色 `bg-blue-500` や角の丸み `rounded-md` 、候補リストについてマウスホバー時のスタイル `hover:bg-gray-200` を設定し、それらしいデザインにしてみます。

{% highlight html %}
<div>
  <div class="py-2 px-2">Please choice.</div>
  <button class="py-2 bg-blue-500 text-white w-40 rounded-md">Dropdown button</button>
  <div class="absolute w-48 mt-2 bg-white rounded shadow-lg">
    <a href="#" class="block py-2 px-2 text-gray-800 hover:bg-gray-200">Apple</a>
    <a href="#" class="block py-2 px-2 text-gray-800 hover:bg-gray-200">Banana</a>
    <a href="#" class="block py-2 px-2 text-gray-800 hover:bg-gray-200">Orange</a>
  </div>
</div>
{% endhighlight %}

**【実行例】**

![](/assets/images/2932ca79e6dd4b9bcf79f193ca93b7da.png?w=303)

デザインはこれでOKとします。

次に、Alpine.jsによる動作の設定を加えますが、上記の状態（ソース）だとコードがごちゃごちゃしてわかりづらいので、ひとつ前のソースに戻して説明します。

## Alpine.jsによる動作の設定

詳細は Alpine.js のサイトをご確認いただくことにして、そのサイトの情報から見よう見まねでとりあえず実装してみます。私のようにあまりJavaScriptに詳しくない者でも、やろうとしていることがなんとなくわかります。

{% highlight html %}
<div x-data="{open: false, sel: 'Please choice.'}">
  <div x-text="sel" class="py-2"></div>
  <button x-on:click="open = !open" class="py-2">Dropdown button</button>
  <div x-show="open" x-on:click.away="open = false" class="absolute w-48">
    <a href="#" x-on:click="sel = 'apple'; open = false" class="block py-2">Apple</a>
    <a href="#" x-on:click="sel = 'banana'; open = false" class="block py-2">Banana</a>
    <a href="#" x-on:click="sel = 'orange'; open = false" class="block py-2">Orange</a>
  </div>
</div>
{% endhighlight %}

一番外側ブロックの div に `x-data` を設定していますが、このブロック内で使用する変数（ `open` と `sel` ）の定義と初期値の設定をしています。

内側の最初の div にて、`x-text="sel"` とすることによって、変数 `sel` の値、つまり「Please choice.」の文字を表示してくれます。

ボタンをクリックすることによって、`x-on:click="open = !open"` が実行されるのですが、`!open` は `open` の否定なので、`open = true` ということになり、リストのブロック div が `x-show="open"` のところが、`x-show = true` となり、リストが表示されるようになります。

`x-on:click.away="open = false"` は、他の部分をクリックした際に実行されるので、その際は `open = false` となり、リストが表示されなくなります。

各リストのアイテムは、そのクリックによって、 `x-on:click="sel = 'apple'; open = false"` が実行され、変数 `sel` に各アイテムで設定した値が格納され、リストを非表示にします。

これで、内側の最初の div の `x-text="sel"` によって、ユーザが選んだアイテムの値が表示されるようになります。

**【実行例】**

![](/assets/images/8be708b4b02c8c839aba16aa015085cd.png?w=631)

## 最終形

上のAlpine.jsによる動作の設定をしたものに、その前のTailwind CSSでの詳細な設定を組み合わせたもの（最終形）がこちらになります。

{% highlight html %}
<div x-data="{open: false, sel: 'Please choice.'}">
  <div x-text="sel" class="px-2 py-2"></div>
  <button x-on:click="open = !open" class="bg-blue-500 text-white w-40 py-2 rounded-md">Dropdown button</button>
  <div x-show="open" x-on:click.away="open = false" class="absolute mt-2 w-48 bg-white rounded shadow-lg">
    <a href="#" x-on:click="sel = 'apple'; open = false" class="block px-2 py-2 text-gray-800 hover:bg-gray-200">Apple</a>
    <a href="#" x-on:click="sel = 'banana'; open = false" class="block px-2 py-2 text-gray-800 hover:bg-gray-200">Banana</a>
    <a href="#" x-on:click="sel = 'orange'; open = false" class="block px-2 py-2 text-gray-800 hover:bg-gray-200">Orange</a>
  </div>
</div>
{% endhighlight %}

**【実行例】**  

![](/assets/images/c26ebdf763bb8f5d6469073f21843cee.png?w=317)

クラスの指定部分が多く、かつタグ内に動作の指定も記載するので、一つのタグの記載が非常に長くなってしまい見づらいですが、別途JavaScriptのコードを準備する必要もないので、これはこれでアリなのではないでしょうか。

## おまけ

ちなみに、あまりカスタマイズできませんでしたが、シンプルに `select` と `option` タグをつかった例がこちらです。

{% highlight html %}
<div x-data="{sel: 'Please choice'}">
  <div x-text="sel" class="px-2 py-2"></div>
  <select x-model="sel" class="w-40 rounded-md">
    <option value="apple">Apple</option>
    <option value="banana">Banana</option>
    <option value="orange">Orange</option>
  </select>
</div>
{% endhighlight %}

**【実行例】**  

![](/assets/images/2af0eb6196f2bf1356f3a9d70e741adc.png?w=255)
