---
title: "初心者のためのviエディタの使い方"
date: 2015-01-26
categories: 
  - "linux"
---

本ページのコンテンツの内容はこちらです。

# TOC

* TOC
{:toc}

## はじめに

（参考文献：「[RUNNING LINUX 導入からネットワーク構築まで](https://www.amazon.co.jp/dp/4900900001/ref=nosim?tag=tamoamazon-22) (オライリー・ジャパン出版)」より，入力用サンプルの文章もこちらから引用させて頂いています．）

vi(Vim)は，UNIXシステム用として最初に作成された真のスクリーンエディタで，シンプルで軽快なアプリケーションです．

システム管理者であるならば，viを習得しておくことは非常に大切です． Linux（UNIX）システムで**viが入っていないことは無いに等しい**ので，viを知っておくことはLinuxを扱う人にとって必要不可欠です．なぜなら，Linuxサーバをリモートで保守する場合，たいていの場合はGUIではなく，マウス等を使わないコンソール（テキスト）ベースでの作業となります．その環境で常に軽快で使えるエディタとなると，このviぐらいしかありません．

最近のディストリビューションでは，vi の代わりにその拡張版である vim がインストールされています． **vimは，viのクローン**で，かつ，viを上回る機能を持っています．

通常 vim は `/usr/bin/vi` にシンボリックリンクが張られ，「`vi`」とコマンドを入力しても，viではなくvimが起動されるようになっています．

## viの起動

- 書式: `vi [ファイル名]`

それでは，ファイル「hello.txt」を編集してみましょう．

```console
$ vi hello.txt[Enter]
```

編集画面は次のようになります．

```console
~
~
~
~
~ 
"hello.txt" [New file]
```

「**`~`**」（チルダ）はファイルの終わり\[EOF\]を表しています．

[↑ Back to top](#toc)

## 編集モードとコマンドモード

viには次の３つのモードがあります．

- コマンドモード
- 編集モード
- exモード

文字を入力する，または編集する為には，「編集モード」にしなくてはなりません．

また，文字を入力後カーソル等を移動させたい場合は，「コマンドモード」にしなくてはなりません． この部分が，viを初めて扱う人には使いづらいところです．

とりあえず，入力したい場合は `[i]` キーを押して「編集モード」にしなくてはなりません．．

※起動直後は「コマンドモード」になっていますので，テキストを挿入するには「編集モード」にする必要があります．

それでは `[i]` キーを押して編集モードに入り，次のように入力してください．

```text
Now is the time for all men to come to the aid of the party.
~
~
```

間違いを訂正する場合は `[Backspace]` キーを押して訂正します．

入力が終わったら，コマンドモードに戻りましょう．コマンドモードに戻るには `[Esc]` キーを入力します．

コマンドモードにいる間は，カーソルを移動する為に矢印キーを使用することができます． また，カーソルキーの代わりに次のキーを利用することもできます．

- `[h]`：左
- `[j]`：下
- `[k]`：上
- `[l]`：右

テキストを挿入する（編集モードに入る）為のコマンドは `[i]` の他に `[a]` があります．

- `[i]`：カーソル位置に文字を挿入(insert)
- `[a]`：カーソル位置の直後に文字を挿入(append)

それでは，矢印キーを利用して，語 "all" と "men" の間にカーソルを移動し，`[a]` キーを押し，「wo」 と入力し `[Esc]` キーで編集モードを抜けてください．

```text
Now is the time for all women to come to the aid of the party.
~
~
~
```

現在行の下に空行を挿入して，テキストを挿入するには `[o]`  （オー）コマンドを使用します．

- `[o]`：空行挿入(open)

それでは `[o]` を押して，１行追加してください．次のように文字を入力した後 `[Esc]` を押して，コマンドモードにしてください．

```text
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for pizza and beer.  
~
~
```

コマンドモードと編集モードの切り替えを覚えましょう．

```
                ==> [i] or [a] or [o] ==> 
(command mode)                             (edit mode)
                       <== [Esc] <==
```


※どのモードいるのかわからなくなったら `[Esc]` を押してください． （コマンドモードで `[Esc]` 押したとしても，Beep音が鳴るだけで問題ありません．）

[↑ Back to top](#toc)

## テキストの削除と変更の復元

コマンドモードにて，文字を削除するには `[x]` コマンドを利用します．

それでは，`[x]` を５回押してください．次のようになります．

```
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for pizza and_
~
~
```

ここで，`[a]` を押して何かテキストを入力して，`[Esc]`キーを押します．

```
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for pizza and Diet Coke.
~
~
```

`[dd]` コマンド（同じ行で2回 `[d]` を押す）を使用すると，現在行全体を削除することができます．

カーソルが2番目の行にあるとき，`[dd]`と入力すると次のようになります．

```
Now is the time for all women to come to the aid of the party.
~
~
~
```

- `[x]`：カーソルの１文字を削除
- `[dd]`：カーソルのある行（現在行）を削除
- `[ndd]`：カーソルのある行を含む n 行を削除（nは任意の行数）

削除されたテキストは `[p]` コマンドを使用して再度挿入することができます．

`[p]`を押してみてください．

```
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for pizza and Diet Coke. 
~
~
```

- `[p]`：現在行の直後の行に挿入
- `[P]`：現在行の直前の行に挿入

`[p]`, `[P]` コマンドはバッファにあるテキストを挿入します．

`[u]` コマンドは最新の変更を取り消します．

- `[u]`：元に戻す(undo)

カーソルにある単語を削除するには `[dw]` コマンドを使用します．

"Diet" という語の上にカーソルを置いて，`[dw]`と入力してください．

```
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for pizza and Coke.
~
~
```

u コマンドを使って，元に戻してください．

[↑ Back to top](#toc)

## テキストの変更

`[r]`, `[R]` コマンドはテキストを置換（replace）します．つまりテキストを上書きすることができます．

- `[R]`：カーソル以降の文字を上書きする
- `[r]`：カーソルの位置にある１文字を置換

カーソルを pizza の先頭文字に置いて，`[R]`を押し，次のように入力してください．

```
Now is the time for all women to come to the aid of the party.
Afterwards, we'll go out for burgers and fries. 
~
~
```

`[r]`を押しても編集（挿入）モードには入りません．従って，`[Esc]`を押してコマンドモードに戻る必要はありません．

`[~（チルダ）]`キーを押すと，カーソルの下にある文字を大文字から小文字に，小文字から大文字に変換します．

"Now" の "o" の上にカーソルを置いて，`[~]`を何度も押すと次のようになります．

```
NOW IS THE TIME FOR ALL WOMEN TO COME TO THE AID OF THE PARTY.
Afterwards, we'll go out for burgers and fries.
~
~
```

[↑ Back to top](#toc)

## 移動コマンド

文書中のカーソルを矢印キーで移動させる方法は前述しましたが，他にも以下のものがあります．

- `[w]`：次の単語(word)の先頭の位置に移動
- `[^]`：現在行の先頭の位置に移動，または\[0(ゼロ)\]を押す
- `[$]`：現在行の末尾の位置に移動
- `[Ctrl]+[f]`：次画面へ移動(forward)
- `[Ctrl]+[b]`：前画面へ移動(backward)
- `[G]`：ファイルの末尾に移動
- `[1G]`：ファイルの先頭に移動

任意の行（ n 行目）に移動するには，n`[G]` と入力します．

たとえば，2行目に移動したいときは，2`[G]` と入力します．

`[/]`キーの後にパターンを入力して`[Enter]`キーを押すと，カーソル以降のテキストの中で，そのパターンが最初に現れる場所にジャンプします．

たとえば，カーソルをファイルの先頭に置いて，`/burg[Enter]` と押すと，カーソルは burgers の先頭に移動します．

- `[/パターン]`：下方向にパターン検索
- `[?パターン]`：上方向にパターン検索

それでは，カーソルをファイルの先頭に置いて，`/THE[Enter]` と入力してみてください．

```
NOW IS THE TIME FOR ALL WOMEN TO COME TO THE AID OF THE PARTY.
Afterwards, we'll go out for burgers and fries. 
~
~
```

するとカーソルは，"NOW IS THE TIME"の "THE" の先頭に移動します． 下候補は `[n]` コマンドで，上候補は `[N]` コマンドです．

- `[n]`：下候補，下方向に次候補を探します．
- `[N]`：上候補，上方向に次候補を探します．

パターンについては，grep コマンド等で使用する正規表現が使えます．たとえば，

```
/[AT]
```

というパターンで検索すると，文書中の "A" または "T" の文字にカーソルが移動します．

移動コマンドを削除などのほかのコマンドに結びつけることができます．たとえば，

- `[d$]`：カーソル位置から行末までを削除
- `[dG]`：カーソル位置からファイルの末尾まで削除

というのがそれにあたります．

※注意 最近のvi（Vim）は，デフォルトで検索文字列が反転カラー表示されてしまいます．

検索中は見やすく便利が良いのですが，この反転カラー表示が煩わしいときがあります．

その時は，ex モードにて次のようにコマンドを入力すると，検索文字列のカラー表示を取り消してくれます．

```
:set nohlsearch
```

反対は，

```
:set hlsearch
```

[↑ Back to top](#toc)

## ファイルの保存とviの終了

viにおいてファイルを扱うコマンドのほとんどが「exモード」から起動されます．

exモードには，コマンドモードから `[:]` を押して入ります．

`[:]` を入力すると，画面の最下行に移動し，そこでさまざまな種類の拡張コマンドを入力することができます．

- `:w` ファイルを上書き保存
- `:w [ファイル名]` ファイルを名前を付けて保存
- `:q` viを終了
- `:wq` ファイルを書き込んでからviを終了
- `ZZ` （ :wq と同等）
- `:q!` 強制的にviを終了（変更内容を保存しないで終了）
- `:r [ファイル名]` 別のファイルの内容をバッファに読み込む
- `:r! [コマンド]` コマンドの実行結果をバッファに読み込む

[↑ Back to top](#toc)

## 広域検索と置換

パターンにマッチした文字列に対し置換を行うコマンドは，次のとおりです，

```
:[開始行,終了行]s/パターン/置換文字列/フラグ
```

開始行と終了行の間にあるパターンを検索し，現れたパターンを置換文字列で置換します． パターンは正規表現を使用します．

例として，1行目から10行目までに現れた "foo" を "hoge" に置換したい場合は，次のように入力します．

```
:1,10s/foo/hoge/g
```

行番号を指定する代わりに，全ファイルを参照する `％` 記号を使用することもできます．

その他の特殊記号も行番号の代わりに使用することができます．

フラグに `g` を指定すると，各行に現れた全てのパターンを置換します．

フラグを指定しなければ，各行で最初に現れたパターンのみを置換します．

それでは，次のようにコマンドを入力してみてください．

```
:%s/THE/The[Enter]
```

次のようになるはずです．

```
NOW IS The TIME FOR ALL WOMEN TO COME TO THE AID OF THE PARTY.
Afterwards, we'll go out for burgers and fries. 
~
~
```

次に，フラグ `g` を付加して実行してみてください．

```
:%s/THE/The/g
```

次のようになるはずです．

```
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
Afterwards, we'll go out for burgers and fries. 
~
~
```

全ての "THE" が "The" に置換されることが確認できます．

[↑ Back to top](#toc)

## コピー＆ペースト

`[dd]` コマンドは現在行を削除し，その行の内容をバッファにコピーするものでした．

これに似ているのが `[yy]` コマンドで，これは現在行を削除しないで，バッファにコピーします．

それでは，先頭行にカーソルを持っていき，`[yy]` と入力し，２行目にカーソルを移動させ， `[p]` コマンドを入力してみてください．

次のようになります．

```
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
Afterwards, we'll go out for burgers and fries. 
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
~
```

また，`[dd]` の場合と同様 n`[yy]`（nは任意の行数） とするとカーソルのある行を含む n 行をバッファにコピーします．

また，移動コマンドと組み合わせることで，たとえば `[y$]` とすると，カーソル位置からその行の末尾までをコピーします．

それでは，２行目の "burgers" 以降のその行の末尾までを，４行目，５行目にコピーしてみてください．

まず，"burgers" の "b" の位置にカーソルを移動させ，`[y$]` を実行します．

その後，３行目にカーソルを移動させ，`[o]`（オー） コマンドで空行を挿入し，`[Esc]`で，コマンドモー ドに戻し，p コマンドを実行します．

５行目についても，同様に，`[o]` --> `[Esc]` --> `[p]` とコマンドを実行していきます．

次のようになります．

```
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
Afterwards, we'll go out for burgers and fries. 
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
burgers and fries.
burgers and fries.
~
```

次に，２行目の "go out for"を，４行目の "and" と "fries" の間にコピーする手順を行ってみます．

1. ２行目の "go" の "g" の位置にカーソルを移動する．
2. "go" を含む３語をコピーするため，コマンド `[3yw]` を実行する．
3. ４行目の "and" の次の空白文字の位置にカーソルを移動する．
4. `[p]` コマンドを実行する．

次のようになります．

```
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
Afterwards, we'll go out for burgers and fries. 
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
burgers and go out for fries. 
burgers and fries.
~
```

[↑ Back to top](#toc)

## viを拡張する

exモードで「`:set` 」コマンドを実行することにより vi をカスタマイズできます．

たとえば行番号を表示させたいときは，次のコマンドを実行します．

```
:set number
```

```
1  NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
2  Afterwards, we'll go out for burgers and fries.
3  NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
4  burgers and go out for fries.
5  burgers and fries. 
~
```

解除するときは，次のように実行します

```
:set nonumber
```

自動字下げ機能をオンにするには，次のコマンドを実行します．

```
:set ai
```

自動字下げ機能をオンにして６行目以降を入力してください．

- `[Tab]` ：レベルを１つ下にする（字下げ）
- `[Ctrl]+[d]`：レベルを１つ上にする（復帰）

```c
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
Afterwards, we'll go out for burgers and fries. 
NOW IS The TIME FOR ALL WOMEN TO COME TO The AID OF The PARTY.
burgers and go out for fries.
burgers and fries.
int main(){
        int a, b;
        for(a = 0; a < 10; a++){
                b = b + 1;
        } 
        printf("b: %d\n", b); 
}
~
```

これも，解除するときは次のコマンドを実行します．

```
:set noai
```

これらの環境設定をexモードで行う場合は一時的ですが， 常に，行番号表示などの環境にしておきたい場合は，ホームディレクトリに，次のようなファイルを置いておくこ とで，viの起動時にその環境が有効になります．

「`.vimrc`」ファイル（※ vi の場合は「`.exrc`」）

```
set number            "<-- :set コマンドの内容そのまま記述
set ai                "    （但し : を付ける必要はない）
set tabstop=4         "タブ幅を4文字分に設定する
set shiftwidth=4      "自動で挿入されるタブ幅
set softtabstop=0     "こちらは0にして，タブ幅はtabstopの値に任せる
syntax enable         "構文ハイライトを有効
colorscheme delek     "カラースキーマの変更
```

※参考「デフォルトでインストールされている — 名無しのvim使い」 http://nanasi.jp/colorscheme/default\_install.html

上記，ホームディレクトリに「`.vimrc`」ファイル作成後，viを起動し，動作を確認してください．

確認後，「`.vimrc`」ファイルを削除してください．（そのまま残していても構いません）
