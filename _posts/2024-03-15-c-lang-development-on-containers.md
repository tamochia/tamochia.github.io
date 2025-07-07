---
layout: post
title: "軽量Dockerコンテナ上でC言語プログラミング環境を構築してみる"
date: 2024-03-15
categories:
  - "docker"
  - "linux"
tags:
  - "alpine"
  - "gcc"
---
初めてC言語に触れたのは大学時代、そして本格的に使い始めたのは社会人になってからで、その頃、1990年初め頃、C言語の開発環境といえば、Borand C/C++、Turbo C、Visual C/C++ などがありましたが、どれも有償で結構したので、若い自分にとっては、個人で買うのは厳しい時代でした。

そんな中でも、「LSIC-86 試食版」はフロッピーディスク1枚に収まるサイズで軽く、しかも無償、C言語の基礎を覚えるにはちょうど良いコンパイラでした。

それから、FreeBSD、Linuxなどで gcc を触れるようになり、今に至るのですが、Windows でC言語環境を整えようと思ったら、ちょっと前までは、Cygwin、MSYS2＋Vagrant、あと、VirualBoxとかでLinuxの仮想マシンを準備したりして、やっていましたが、今は、WSL2 (Windows Subsystem for Linux) という立派な環境があるので、開発環境としては恵まれております。

今回は、あえてそのWSL2は使用せずに、Docker Desktop for Windows だけを使って、PowerShellやVSCode上でDockerコンテナを起動して、そのコンテナを利用してC言語の開発環境を構築してみようという内容です。


## 環境について

今回使用した、PowerShell のバージョンです。

```console
PowerShell 7.4.1
PS C:\Users\hogeta> $PSVersionTable

Name                           Value
---- -----
PSVersion                      7.4.1
PSEdition                      Core
:
```

PowerShell から、Docker の環境を確認します。
Docker 環境は、Docker Desktop を使用しています。

```console
PS C:\Users\hogeta> docker version
Client:
 Cloud integration: v1.0.35+desktop.11
 Version:           25.0.3
 API version:       1.44
 Go version:        go1.21.6
 Git commit:        4debf41
 Built:             Tue Feb  6 21:13:02 2024
 OS/Arch:           windows/amd64
 Context:           default

Server: Docker Desktop 4.28.0 (139021)
 Engine:
  Version:          25.0.3
  API version:      1.44 (minimum version 1.24)
  :
```

基本的には、PowerShell から Docker が利用できさえすれば問題ありません。

## Dockerイメージのビルド

### 作業用ディレクトリの作成

PowerShell を起動し、DockerfileやCのソースファイルを格納するための作業ディレクトリを作成（ここでは、例として「`test_alpine`」としています。）し、そのディレクトリに移動します。

```console
PS C:\Users\hogeta> mkdir test_alpine
PS C:\Users\hogeta> cd test_alpine
PS C:\Users\hogeta\test_alpine>
```

### Dockerfileの作成

上で作成した作業ディレクトリ「`test_alpine`」上に、`Dockerfile` を作成します。
PowerShell (Windows) でも、`touch` コマンドが使えれば良いのですが。。。

```console
PS C:\Users\hogeta\test_alpine> touch Dockerfile
touch: The term 'touch' is not recognized as a name of a cmdlet, function, script file, or executable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
```

PowerShell の場合は、`New-Item` コマンドがそれに該当するようです。

【参考】 [PowerShellでtouchができない！代替コマンドとおすすめ解決法 - まいまいテックブログ](https://maimai-tech.com/programming/terminal/powershell-touch/)

```console
PS C:\Users\hogeta\test_alpine> New-Item Dockerfile
```

空のファイルができたら、それをメモ帳（notepad.exe）などのエディタで編集します。

```console
PS C:\Users\hogeta\test_alpine> notepad Dockerfile
```

空のファイルを作らずに、最初から「`notepad Dockerfile`」で行っても良いのですが、そうするとメモ帳で編集して保存するときに、ファイル名に拡張子「`.txt`」が付加され、「`Dockerfile.txt`」というファイル名になってしまいます。その場合は、「`ren`」コマンドを使って、「`ren Dockerfile.txt Dockerfile`」のようにファイル名の変更処理が必要になります。

それでは、今回の肝である Dockerfile の内容です。

#### Dockerfile

{% highlight docker linenos %}
FROM alpine
ARG UID USER
RUN apk add --no-cache bash gcc libc-dev vim

RUN addgroup -g $UID $USER && \
    adduser -u $UID -s /bin/bash -D -S $USER -G $USER
WORKDIR "/home/$USER"
COPY . .
RUN chown -R $USER:$USER .
USER $USER
ENTRYPOINT ["/bin/bash"]
{% endhighlight %}

それぞれ、各行についての説明を以下に記載します。

- `FROM alpine` … Dockerベースイメージは、超軽量Linuxディストリビューションといわれている Alpine Linux を指定します。
- `ARG UID USER` … このDockerfile内で使用する変数ですが、こちらは、「`docker build`」コマンド実行の際に、「`--build-arg UID=1000 --build-arg USER=hogeta`」のような形で指定して、それぞれ変数に読み込ませることができるようになります
- `RUN apk add --no-cache bash gcc libc-dev vim` … C言語開発に必要な gcc と libc-dev、そして vim と bash のパッケージだけをインストールします
- `RUN addgroup ... adduser ...` … ビルド時に指定した UID と USER の値を元にユーザを作成します
- `WORKDIR "/home/$USER"` … コンテナ起動時のディレクトリは、そのユーザのホームディレクトリとします。
- `COPY . .` … ホスト側の作業ディレクトリ「test\_alpine」の中のファイルを、コンテナ内のホームディレクトリにコピーします
- `RUN chown -R $USER:$USER .` … 上でコピーしたファイルの所有者は root になってしまっているので、ここで、所有者 USER で指定したユーザに変更します
- `USER $USER` … このコンテナは、USER で指定したユーザで起動します
- `ENTRYPOINT ["/bin/bash"]` … コンテナを起動したら bash が動くようにします

メモ帳で上書き保存したら、次はいよいよDockerイメージのビルドです。

### Dockerイメージのビルド

Dockerイメージのビルドは、次のコマンドになります。「`--buil-arg`」オプションにて、ここでは、`UID` にユーザIDとして一般的な「`1000`」、`USER` にユーザ名「`hogeta`」を指定しています。

```console
PS C:\Users\hogeta\test_alpine> docker build -t test_alpine --build-arg UID=1000 --build-arg USER=hogeta .
```

実際に、実行すると次のような画面になります。

```console
PS C:\Users\hogeta\test_alpine> docker build -t test_alpine --build-arg UID=1000 --build-arg USER=hogeta .
[+] Building 8.2s (12/12) FINISHED                                                docker:default
 => [internal] load build definition from Dockerfile                                        0.0s
 => => transferring dockerfile: 304B                                                        0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                            8.0s
:
View build details: docker-desktop://dashboard/build/default/default/vpzll4gwxrfkhkkuul4frsd8j

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview
```

完了したら、実際に作成されたイメージを確認してみます。

```console
PS C:\Users\hogeta\test_alpine> docker images
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
test_alpine         latest    1f7f5131dd50   33 minutes ago   192MB
:
```

今回作成したDockerイメージのサイズは 192MB 程です。

## コンテナの作成と起動

では、作成したDockerイメージ「`test_alpine`」をもとに、コンテナの作成及び起動を試みます。
結果的には、次のようなコマンドになります。

```console
PS C:\Users\hogeta\test_alpine> docker run --rm -it --name test-shell --mount "type=bind,source=$(pwd),target=/home/hogeta" test_alpine
```

基本は、対象となるDockerイメージを引数に指定して、「`docker run --rm -it test_alpine`」とすればよいです。この時、「`--rm`」オプションで、コンテナ終了時にコンテナを削除、「`-it`」オプションで、対話モードでコンテナを起動します。

そして、もう一つのオプション「`--mount`」の部分

```
--mount "type=bind,source=$(pwd),target=/home/hogeta"
```

ですが、ここでは、

- バインド元 (source) として、ホスト側の `C:\Users\hogeta\test-alpine`
- バインド先 (target) として、コンテナ側の `/home/hogeta`

にてバウンドマウントを指定しています。ここは、それぞれ物理パスでの指定になります。
PowerShellにて、`$(pwd)` と指定すると、現在のカレントディレクトリの物理パスを返すようです。

【参考】 [バインド マウント(bind mount) の使用 — Docker-docs-ja 24.0 ドキュメント](https://docs.docker.jp/storage/bind-mounts.html)

これにより、コンテナ内で作成したCソースファイルやコンパイル結果などのファイルは、マウントしたホスト側の `C:\Users\hogeta\test_alpine` に作成されることになりますので、コンテナを終了及び削除したとしても、そのままホスト上に残ることになります。

では、実際に実行してみます。

```console
PS C:\Users\hogeta\test_alpine> docker run --rm -it --name test-shell --mount "type=bind,source=$(pwd),target=/home/hogeta" test_alpine
2cb24bf8bf7e:~$
```

Bashのプロンプト「`$`」が表示され、コンテナ内に入ったことがわかります。

では、さっそく「`pwd`」コマンドで現在のディレクトリを確認します。

```console
2cb24bf8bf7e:~$ pwd
/home/hogeta
```

次に、「`ls`」コマンドでカレントディレクトリの内容を見てみます。

```console
2cb24bf8bf7e:~$ ls -al
total 8
drwxrwxrwx    1 root     root       512 Mar 15 03:34 .
drwxr-xr-x    1 root     root      4096 Mar 15 03:27 ..
-rw-------    1 hogeta   hogeta      22 Mar 15 03:30 .bash_history
-rwxrwxrwx    1 root     root       265 Mar 15 03:26 Dockerfile
```

次に、「`id`」コマンドでユーザID等の情報を確認します。

```console
2cb24bf8bf7e:~$ id
uid=1000(hogeta) gid=1000(hogeta) groups=1000(hogeta)
```

次に、「`touch`」コマンドでファイル（ここでは、「`hello.c`」）を作成し、その後「`ls`」コマンドで確認します。作成したファイルの所有者がユーザ `hogeta` であることが確認できます。

```console
2cb24bf8bf7e:~$ touch hello.c
2cb24bf8bf7e:~$ ls -l
total 0
-rwxrwxrwx    1 root     root       265 Mar 15 03:26 Dockerfile
-rw-r--r--    1 hogeta   hogeta       0 Mar 15 03:36 hello.c
```

ちょっとここで、コンテナ内ではなく、別のウィンドウでPowerShellを起動して、test-alpine ディレクトリの内容を確認してみます。

```console
PS C:\Users\hogeta\test_alpine> dir

    Directory: C:\Users\hogeta\test_alpine

Mode  LastWriteTime       Length Name
----  -------------       ------ ----
-a--- 2024/03/15    12:30     22 .bash_history
-a--- 2024/03/15    12:26    265 Dockerfile
-a--- 2024/03/15    12:36      0 hello.c
```

コンテナと同様に、ファイル「`hello.c`」があることがわかりますが、ファイル作成時間が9時間程ずれているのがわかります。

それは、コンテナ内で「`date`」コマンドを実行してみるとわかりますが、タイムゾーンが UTC（世界標準時）になっているからです。タイムゾーンを変更する方法もありますが、今回は、これでも別に支障はありません。

```console
2cb24bf8bf7e:~$ date
Fri Mar 15 03:38:13 UTC 2024
```

## Cソースファイル作成とコンパイル

では、Vimを使って、Cソースファイル「`hello.c`」を編集します。

```console
2cb24bf8bf7e:~$ vim hello.c
```

簡単に、Hello World プログラムです。

{% highlight c linenos %}
#include <stdio.h>

int main(void){
    printf("Hello world!\n");
    printf("こんにちは、世界！\n");

    return 0;
}
{% endhighlight %}

画面スクショで観るとこんな感じ。

![](/assets/images/c-lang-development-on-containers_fig01.png?w=426)

ちゃんとシンタックスハイライトされています。

それでは、gccコマンドを使用して、コンパイル（ビルド）し、生成された実行ファイルを実行してみます。

```console
2cb24bf8bf7e:~$ gcc -o hello hello.c
2cb24bf8bf7e:~$ ./hello
Hello world!
こんにちは、世界！
```

無事に、ビルドされ実行できました！

ディレクトリの内容は次のとおり。実行形式ファイル「`hello`」が生成されているのが確認できます。

```console
2cb24bf8bf7e:~$ ls -l
total 20
-rwxrwxrwx    1 root     root        265 Mar 15 03:26 Dockerfile
-rwxr-xr-x    1 hogeta   hogeta    18472 Mar 15 03:42 hello
-rw-r--r--    1 hogeta   hogeta      120 Mar 15 03:41 hello.c
```

コンテナの終了は「`exit`」コマンドです。

```console
2cb24bf8bf7e:~$ exit
exit
PS C:\Users\hogeta\test_alpine>
```

コンテナが起動しているか否かは、「`docker ps -a`」コマンドで確認できます。

```console
PS C:\Users\hogeta\test_alpine> docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

コンテナがきれいさっぱり削除されているのがわかります。

では、ホスト側に戻りましたので、「`dir`」コマンドにて、現在のディレクトリの内容を確認します。

```console
PS C:\Users\hogeta\test_alpine> dir

    Directory: C:\Users\hogeta\test_alpine

Mode  LastWriteTime        Length Name
----  -------------        ------ ----
-a--- 2024/03/15    12:43     144 .bash_history
-a--- 2024/03/15    12:41     781 .viminfo
-a--- 2024/03/15    12:26     265 Dockerfile
-a--- 2024/03/15    12:42   18472 hello
-a--- 2024/03/15    12:41     120 hello.c
```

コンテナの中身と変わらないです。もちろん、Cソースファイルも次のようにメモ帳で編集することも可能です。

```console
PS C:\Users\hogeta\test_alpine> notepad hello.c
```

![](/assets/images/c-lang-development-on-containers_fig02.png?w=488)

メモ帳で編集したCソースファイルを、コンテナ上でコンパイル（ビルド）することも可能です。

```console
2ae99a56354d:~$ gcc -o hello hello.c
2ae99a56354d:~$ ./hello
Hello world!
こんにちは、世界！

おやすみ～
```

もし、VSCode (Visual Studio Code) がインストール済みであれば、PowerShell上で次のように実行すれば、VSCodeが起動します。

```console
PS C:\Users\hogeta\test_alpine> code .
```

VSCode内のターミナルにて、コンテナ起動（`docker run --rm -it ...`）すれば、ソースファイルも編集しやすく、コンパイル結果も一つの画面で確認でき使いやすいと思います。

![](/assets/images/c-lang-development-on-containers_fig03.png?w=1005)

## 後片付け

Dockerイメージが不要でしたら、「`docker rmi`」コマンドでDockerイメージを指定して削除することが可能です。

```console
PS C:\Users\hogeta\test_alpine> docker rmi test_alpine
Untagged: test_alpine:latest
Deleted: sha256:ce5e5208b6131178c05b1a8cc279f19e4c09ea8bf8825876380973b78c0171d5
```

もし、こんな感じで表示され、エラーになってしまったら。。。

```console
PS C:\Users\hogeta\test_alpine> docker rmi test_alpine
Error response from daemon: conflict: unable to remove repository reference "test_alpine" (must force) - container 4b5ca9752fa6 is using its referenced image ce5e5208b613
```

これは、コンテナがまだ動いている状態ですので、「`docker ps -a`」コマンドで確認してみましょう。

```console
PS C:\Users\hogeta\test_alpine> docker ps -a
CONTAINER ID   IMAGE         COMMAND       CREATED         STATUS         PORTS     NAMES
4b5ca9752fa6   test_alpine   "/bin/bash"   6 minutes ago   Up 6 minutes             test-shell
```

コンテナが動いていることがわかります。動いているコンテナの停止は「`docker stop`」コマンドです。

```console
PS C:\Users\hogeta\test_alpine> docker stop test-shell
test-shell
```

これでコンテナが削除されたので、Dockerイメージも削除可能です。
もし、「`docker stop`」でも削除できなかった場合は、「`docker rm <コンテナID>`」でコンテナを削除します。

