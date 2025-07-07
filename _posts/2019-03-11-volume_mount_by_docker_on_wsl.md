---
title: "Docker for WindowsとDocker on WSLにおけるボリュームマウント"
date: 2019-03-11
categories: 
  - "linux"
tags:
  - "docker"
  - "wsl"
---

# はじめに

まずは、今回の説明におけるホストマシンの環境から、

```console
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
:
```

OSは、Windows Subsystem for Linux (WSL) 上の Ubuntu Linux 18.04.1 そして、WSL上にて導入したDockerクライアントとdocker-composeのバージョンは次の通り。

```console
$ docker --version
Docker version 18.09.1, build 4c52b90

$ docker-compose version
docker-compose version 1.23.2, build 1110ad01
docker-py version: 3.6.0
CPython version: 3.6.7
OpenSSL version: OpenSSL 1.1.0f  25 May 2017
```

最後に、Dockerエンジンの情報は次の通り。

```console
$ curl -i http://localhost:2375/version
HTTP/1.1 200 OK
Api-Version: 1.39
Content-Length: 560
Content-Type: application/json
Date: Fri, 15 Feb 2019 22:22:24 GMT
Docker-Experimental: false
Ostype: linux
Server: Docker/18.09.2 (linux)
{"Platform":{"Name":"Docker Engine -  Community"},"Components"
:
``` 

Hyper-V上の仮想マシンにてDockerサーバが動いている状況です。

`docker run` コマンドを実行すると、イメージからコンテナを生成し、Dockerエンジンはイメージの中に読み書き可能なファイルシステムを作成します。コンテナが動いている（`running`）あるいは停止中（`exited`）状態であれば、そのファイルシステムを保持しますが、コンテナを削除してしまうと、コンテナ動作中に保存した（書き込んだ）データは消えてなくなります。

「-v」オプションを使用すると、ホストとコンテナの間でファイルを共有することができます。これは例えばホストにWebサイトのコンテンツを保存しておき、それをWebサーバイメージから構築されたコンテナにマウントしてコンテンツを公開するというようなことが可能です。

では順に説明します。

* * *

# マウントの簡単な確認

まずは、Cドライブマウントが有効かどうかの確認します。

```console
$ docker run --rm -v c:/Users:/data alpine ls /data
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
6c40cc604d8e: Pull complete Digest:  sha256:b3dbf31b77fd99d9c08f780ce6f5282aba076d70a513a8be859d8d3a4d0c92b8
Status: Downloaded newer image for alpine:latest
All Users
Default
Default User
Public
desktop.ini
foo
```

各オプションの意味は次の通り。

- `--rm` ... 終了時にコンテナを自動的削除する
- `-v HOST_DIR:CONTAINER_DIR` ... ホストのディレクトリ（`HOST_DIR`）をコンテナのディレクトリ（`CONTAINER_DIR`）にマウントする
- `alpine` ... 軽量Linuxである Alpine Linux イメージを指定
- `ls /data` ... コマンドの実行

つまり、ホストの「`C:\Users`」ディレクトリを、コンテナ内の「`/data`」ディレクトリにマウントして、コンテナ上でコマンド「`ls /data`」を実行させています。

Alpine Linuxのイメージがローカルホスト上に無いので、ダウンロード（`pull`）し、コンテナを生成しています。その後、コマンド「`ls /data`」が実行され（実行結果は以下の通り）、コンテナは自動的に削除されます。

```
All Users
Default
Default User
Public
desktop.ini
foo
```

これらは、Windowsの`C:\Users`ディレクトリの内容です。コマンド「`ls /data`」の結果、これが表示されたということは、ホスト側の「`C:\Users`」がコンテナ側の「`/data`」ディレクトリにマウントされたことを意味します。

* * *

# マウントする際のパスの認識

今度は、WSLのホームディレクトリ上の任意のディレクトリをマウントしてみます。 まずはカレントディレクトリのパスの確認から。

```console
[foo@host]$ pwd
/home/foo/Workspace/sample
```

シェルの操作が、ホスト側の操作なのかそれともコンテナ側の操作なのか分からなくならないように、ホスト側の操作の場合は、プロンプトを `[foo@host]$` で明記しています。

ホスト側にてのコンテナと共有するディレクトリ「src」を作成し、そこに適当なファイルを用意します。

```console
[foo@host]$ mkdir src
[foo@host]$ touch src/aaa
[foo@host]$ touch src/bbb
[foo@host]$ touch src/ccc
[foo@host]$ ls src/
aaa  bbb  ccc
```

まずは、vオプションにて相対パスで指定してコンテナを起動してみます。

```console
[foo@host]$ docker run --rm -v "src:/app" alpine  ls /app
```

はい、これはNG（コマンド「`ls /app`」の結果が何も表示されない）でした。

$PWDの結果を使用して場合は、

```console
[foo@host]$ docker run --rm -v "$PWD/src:/app" alpine  ls /app
```

はい、これもNG

実際に/ルートから絶対パスを指定した場合は、

```console
[foo@host]$ docker run --rm -v "/home/foo/Workspace/sample/src:/app" alpine ls /app
```

はい、これもNG

マウントディレクトリから指定した場合は、

```console
[foo@host]$ docker run --rm -v "/mnt/c/Users/foo/Workspace/sample/src:/app" alpine ls /app
```

はい、これもNG

しかーし、ドライブ番号「`c:/`」から指定した場合はOKなのです。

```console
[foo@host]$ docker run --rm -v "c:/Users/foo/Workspace/sample/src:/app" alpine ls /app
aaa
bbb
ccc
```

ここがDocker for WindowsをDockerサーバとして利用している所以なのでしょう。パスを指定する際はWindowsファイルシステムを意識しなくてはなりません。面倒ですね。。。

この場合ダブルクォーテーションを取ってもいけます。

```console
[foo@host]$ docker run --rm -v  c:/Users/foo/Workspace/sample/src:/app alpine ls /app
aaa
bbb
ccc
```

【追記 2019-06-12】

-----ここから----- 

実は「`~`」（チルダ）から指定した場合もOKでした。

```console
[foo@host]$ docker run --rm -v "~/Workspace/sample/src:/app" alpine ls /app
aaa
bbb
ccc
```
-----ここまで-----

さて、マウントが可能になったので、コンテナ側でファイルを生成するとどうなるかやってみます。

```console
[foo@host]$ docker run --rm -v c:/Users/furuya/Workspace/sample/src:/app alpine touch /app/ddd
```

実際にファイル「`ddd`」が生成されたか、ホストの共有ディレクトリ内を確認してみます。

```console
[foo@host]$ ls -l src
合計 0
-rw-r--r-- 1 foo foo 0  2月 15 21:59 aaa
-rw-r--r-- 1 foo foo 0  2月 15 21:59 bbb
-rw-r--r-- 1 foo foo 0  2月 15 21:59 ccc
-rwxr--r-- 1 foo foo 0  2月 15 22:59 ddd
```

実行権限ついちゃうけど、ちゃんとできていました。

* * *

# Dockerfileでのパスの指定方法

今度は、Dockerfileにて、WSLのホームディレクトリ上の任意のディレクトリのデータを、コンテナ側のディレクトリにマウントではなくコピーをしてみます。 まずはカレントディレクトリのパスの確認から。

```console
[foo@host]$ pwd
/home/foo/Workspace/sample

[foo@host]$ ls src/
aaa  bbb  ccc
```

カレントディレクトリ内に、`Dockerfile`を準備します。

{% highlight docker linenos %}
FROM alpine
RUN mkdir /app
COPY src/\* /app
{% endhighlight %}

これは、Alpine Linuxのイメージを使用し、コンテナ内にディレクトリ「`/app`」を作成し、ホスト側カレントディレクトリ内にある「`src`」ディレクトリの中身を「`/app`」ディレクトリにコピーしたものを新たなイメージとして構築（`build`）します。今回は、イメージ名 `buzz` でビルドしてみます。

```console
[foo@host]$ docker build -t buzz .
Sending build context to Docker daemon  4.096kB
Step 1/3 : FROM alpine
---> caf27325b298
Step 2/3 : RUN mkdir /app
---> Running in 38cbde2e6d47
Removing intermediate container 38cbde2e6d47
---> d43c49e04b31
Step 3/3 : COPY src/\* /app
When using COPY with more than one source file, the destination  must be a directory and end with a /
```

COPYの行でなにやら警告のようなものが。。。 COPYにてコピー元を複数のファイルを指定した場合は、コピー先ディレクトリの終わりに「`/`」が必要らしい。

そこで Dockerfile を修正します。

{% highlight docker linenos %}
FROM alpine
RUN mkdir /app
COPY abc/\* /app/
{% endhighlight %}


再度ビルド！

```console
[foo@host]$ docker build -t buzz .
Sending build context to Docker daemon  4.096kB
Step 1/3 : FROM alpine
---> caf27325b298
Step 2/3 : RUN mkdir /app
---> Using cache
---> d43c49e04b31
Step 3/3 : COPY src/\* /app/
---> 580b1377fe57
Successfully built 580b1377fe57
Successfully tagged buzz:latest
```

上手くいったようです。イメージ名もきちんと付いています。

```console
[foo@host]$ docker images
REPOSITORY  TAG         IMAGE ID        CREATED            SIZE
buzz        latest      580b1377fe57    45  seconds ago    5.53MB
alpine      latest      caf27325b298    2  weeks ago       5.53MB
```

コンテナを起動してみます。

```console
[foo@host]$ docker run --rm buzz ls /app
aaa bbb ccc
```

きちんとホストの「`src`」ディレクトリの内容が、コンテナ内の「`/app`」ディレクトリにコピーされていたようです。

後の作業のためにDockerイメージ「`buzz`」を削除しておきます。

```console
[foo@host]$ docker rmi buzz
Untagged: buzz:latest
Deleted:  sha256:580b1377fe57dfaec83b1d5dd3c0c95d7a4414c0fc858b65a42e11...
:
``` 

* * *

# Docker Compose を使用してマウントさせる場合

`docker run` コマンドにて毎回「`-v`」オプションでいちいち長々と入力指定するのは面倒なので、docker compose を使用した方法を説明します。マウントの指定は `docker-compose.yml` でやっておけばいいので楽ですね。

ですが。。。

まずは、ホスト側のカレントディレクト配下の状態はこちら。

```console
[foo@host]$ tree .
.
├── Dockerfile
├── docker-compose.yml
└── src
    ├── aaa
    ├── bbb
    └── ccc
```

そして、「`docker-compose.yml`」ファイルの中身がこちら。

{% highlight yaml linenos %}
version: '3'
services:
  app:
    build: . 
    volumes:
      - ./src:/app
    command: ls /app
{% endhighlight %}

「`build: .` 」は、Dockerfileの場所を指定しています。カレントディレクトリに存在するので「.」としています。「`volumes:`」がマウントの部分で、「`:`」を挟んで「`./src`」がホスト側、「`/app`」がコンテナ側ディレクトリになります。そして最後にコマンドを実行します。そのコマンドの内容が「`command: ls /app`」です。

「`Dockerfile`」については、先で作成したものをそのまま使用します。では、ビルドしてみます。

```console
[foo@host]$ docker-compose build
Building app
Step 1/3 : FROM alpine
---> caf27325b298
Step 2/3 : RUN mkdir /app
---> Running in 074f931e1947
Removing intermediate container 074f931e1947
---> 721d56efd977
Step 3/3 : COPY src/\* /app/
---> 2a8d59203c8a
Successfully built 2a8d59203c8a
Successfully tagged sample\_app:latest
```

一応ビルドはできたみたい。。。

```console
[foo@host]$ docker images
REPOSITORY     TAG         IMAGE ID        CREATED              SIZE
sample_app     latest      2a8d59203c8a    About  a minute ago  5.53MB
alpine         latest      caf27325b298    2  weeks ago         5.53MB
```

コンテナを生成し起動させてみます。

```console
[foo@host]$ docker-compose up
Creating network "sample_default" with the default driver
Creating sample_app_1 ... done
Attaching to sample_app_1
sample_app_1 exited with code 0
```

しかし、何も反応なし... コンテナを削除し、イメージも削除しておきます。

```console
[foo@host]$ docker-compose down
[foo@host]$ docker rmi sample_app:latest
```

記述は間違っていないと思うので、今度は、**Windows PowerShell** で確認してみます。

Windows PowerShellを起動して、まずはカレントディレクトリの状況から確認します。

```console
PS C:\Users\foo> cd Workspace\sample
PS C:\Users\foo\Workspace\sample> tree /f
フォルダー パスの一覧:  ボリューム Local Disk
ボリューム シリアル番号は 408D-31CD です
C:.
│  docker-compose.yml
│  Dockerfile
│
└─src
    aaa
    bbb
    ccc
```

パスの表記はWindows（というかMS-DOSのときから）特有ですが、WSLから見た構成と同じです。 「docker-compose.yml」や「Dockerfile」の内容もそのままで変えていません。

ではビルドしてみます。

```console
PS C:\Users\foo\Workspace\sample> docker-compose build
Building app
Step 1/3 : FROM alpine
---> caf27325b298
Step 2/3 : RUN mkdir /app
---> Running in 1cd18ec25230 Removing intermediate container 1cd18ec25230
---> d1f17cf053cf
Step 3/3 : COPY src/* /app/
---> c546edc21288
Successfully built c546edc21288
Successfully tagged sample_app:latest
```

PowerShellでも問題なくイメージ作成できています。ではコンテナを生成して起動してみます。

```console
PS C:\Users\foo\Workspace\sample> docker-compose up
Creating network "sample_default" with the default driver
Creating sample_app_1 ... done
Attaching to sample_app_1
app_1  | aaa
app_1  | bbb
app_1  | ccc
sample_app_1 exited with code
```

おおお、こちらはきちんと「`/app`」の中身が表示されました。

やっぱりパスの指定が悪いんじゃー、と思い、再度WSL側で色々と挑戦してみます。

{% highlight yaml linenos %}
version: '3'
services:
  app:
    build: . 
    volumes:
      - ~/Workspace/sample/src:/app
    command: ls /app
{% endhighlight %}


ホスト側ディレクトリの部分「`./src`」を、相対パスではなく、きちんとホームディレクトリ「~/」から指定します。

```console
[foo@host]$ docker-compose up
Creating network "sample_default" with the default driver
Creating sample_app_1 ... done
Attaching to sample_app_1
sample_app_1 exited with code 0
```

やっぱりだめー(\*\_\*)

では、これではどうか。きちんと「`/`」から絶対パス指定。

```yaml
volumes:
  - /home/foo/Workspace/sample/src:/app
```

やっぱりNG(;´･ω･)

マウントパスをしっかり書いて指定してみます。。。

```yaml
volumes:
  - /mnt/c/Users/foo/Workspace/sample/src:/app
```

だめNG(T\_T)

では、「`C:\`」から記述してみよう。。。

```yaml
volumes:
  - c:\Users\foo\Workspace\sample\src:/app
```

今度は何かエラーでた(;´･ω･)

```
ERROR: Named volume "c:\Users\foo\Workspace\sample\src:/app" is 
used in service "app" but no declaration was found in the volumes section.
```

そうか「`\`」をエスケープしなきゃだ。

```yaml
volumes: - c:\\Users\\foo\\Workspace\sample\\src:/app
```

やっぱりエラー(´；ω；\`)ウウウ

```
ERROR: Named volume  "c:\\Users\\foo\\Workspace\\sample\\src:/app" is 
used in service "app" but no declaration was found in the volumes section.
```

どうもこうもうまくいかない。 やっぱりWSLのdockerクライアントからDocker for Windowsを利用するのはWindowsファイルシステムへのアクセスという大きな障壁があるのか。。。

とさんざん悩んでいたところ、Compose file の Version 3.2 より、volumes にて長い文字列表記での実装が可能と言う事なので、Version 3.2の表記で記載してみる！

【参考】[Compose file version 3 reference \| Docker Documentation](https://docs.docker.com/compose/compose-file/#long-syntax-3)

ふむふむ、typeにてbindを指定して、sourceにはホスト側の、targetにはコンテナ側の共有するディレクトリを記載すればいいのね。

とりあえずカレントディレクトリからの相対パスを指定してみました。相対パスを使用する際は「.」または「..」から始めるということなので、次のように。

{% highlight yaml linenos %}
version: '3.2'
services:
  app:
    build: .
    volumes:
      - type: bind
        source: ./src
        target: /app
    command: ls /app
{% endhighlight %}


コンテナを起動してみる。

```console
[foo@host]$ docker-compose up
Creating network "sample_default" with the default driver
Creating sample_app_1 ... done
Attaching to sample_app_1
sample_app_1 exited with code 0
```

やっぱダメ。でもへこたれずユーザホームからのパスに書き換えてみる。

{% highlight yaml linenos %}
version: '3.2'
services:
  app:
    build: .
    volumes:
      - type: bind
        source: ~/Workspace/sample/src
        target: /app
    command: ls /app
{% endhighlight %}

さて、コンテナを起動してみます。

```console
[foo@host]$ docker-compose up
Creating network "sample_default" with the default driver
Creating sample_app_1 ... done
Attaching to sample_app_1
app_1  | aaa
app_1  | bbb
app_1  | ccc
sample_app_1 exited with code 0
```

おおーっ上手くいった！(ノ・ω・)ノオオオォォォ-ー

# 結論

DockerfileでCOPYをする場合、ホスト側のパスは相対パスでOK！

`docker run` で `-v(volume)` でホスト側のディレクトリを共有する場合は、絶対パスでないとNG

またその絶対パス指定も「`c:/`」から始めないとだめ。それと事前に Docker for Windows の設定にて C ドライブ共有設定が必要です。

docker-compose.yml に、volumes（マウント情報）を記載する際は、Compose fileのフォーマットバージョンを 3.2 にして、ホスト側のパス（source）は絶対パスで指定します。その際にホームディレクトリを示す「`~/`」が利用可能です。
