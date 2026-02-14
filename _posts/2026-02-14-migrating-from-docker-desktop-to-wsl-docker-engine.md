---
title: "Docker DesktopをやめてWSL2 (Ubuntu 24.04) 上のDocker Engineに移行する"
date: 2026-02-14
categories:
  - "WSL"
tags:
  - "docker"
  - "wsl"
---

# はじめに

これまで、Windows環境ということで「Docker Desktop」を使用してきました。
また、教育機関ということでPersonalライセンスを使用してきましたが、完全無償というわけではないということと、普段からWSL2を使っていることから、別にPowerShallから使用しなくても良いし、Windowsコンテナを使うこともないですし、WSL2上のネイティブなDocker Engineが利用可能ということで、Docker Desktopをやめて、WSL2上のUbuntuネイティブなDocker Engineに移行しようということで、ちょっと今回はChatGPTとCopilotに色々と助言をもらいながらまとめてみました。

ここに記載したのは、当方の環境での実行結果であり、説明した動作を保証したものではないです。
間違っている記述もあるかと思いますので、あくまでもご参考まででお願いします。

# 事前準備

まずは、WSL側で systemd が有効化確認します。

```console
$ ps -p 1
  PID TTY          TIME CMD
    1 ?        00:00:00 systemd
```

ここが、`systemd` ではなく `init` だった場合は、`/etc/wsl.conf` で次のような記述をして、WSLを再起動します。

**`/etc/wsl.conf`**

```
[boot]
systemd=true
```

# Desktop側データのバックアップ

Docker Desktopでは、あくまでも検証用（テスト用）として使用していたもので、特に何か重要なコンテナが動いていたり、Dockerイメージを置いていたわけではないので、あっさり Docker Desktop をアンインストールして、ネイティブ Docker Engine をインストールで良いのですが、ついでですので、Docker Desktop 環境で作成した image や volume も移行を前提に行ってみます。

## image のバックアップ

まずは、image の確認
```console
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
test_alpine   latest    14fb0b7ecad5   15 minutes ago   340MB
alpine        latest    25109184c71b   2 weeks ago      13MB
```

イメージがある場合、次のコマンドで適当なtarファイル（例: `images-backup.tar`）にエクスポートしておきます。

```console
$ docker save -o images-backup.tar test_alpine:latest alpine:latest
$ gzip images-backup.tar
```

## volume のバックアップ

次に、volume の確認
```console
$ docker volume ls
DRIVER    VOLUME NAME
local     test_alpine_mydata
```

この volume の実体はこちらにあるみたい。

```
\\wsl.localhost\docker-desktop\mnt\docker-desktop-disk\data\docker\volumes\test_alpine_mydata\_data
```

Copilotに教えてもらったコマンドで、Docker Volume をバックアップします。

```console
$ docker run --rm -v mydata:/volume -v "$(pwd)":/backup alpine sh -c 'cd /volume && tar cf /backup/mydata.tar .'
gzip mydata.tar
```

# ネイティブ Docker Engine のインストール

ChatGPTとCopilot、どちらも Docker Desktop をアンインストールする前に、ネイティブDocker Engineをインストールせよと言ってきます。理由は、ネイティブ Docker Engine を入れても、Docker Desktop はアンインストールしない限り動いているので、動いているうちは、image や volume の救出が可能であるから、という理由みたい。

ネイティブ Docker Engine のインストール基本は、こちらを参考にDocker公式APTリポジトリから入れるみたい。

[Ubuntu | Docker Docs](https://docs.docker.com/engine/install/ubuntu/)

```console
$ sudo apt update
```
→ これは定番

```console
$ sudo apt install -y ca-certificates curl gnupg
```
→ すでにこれらのパッケージが入っている場合は不要

```console
$ sudo install -m 0755 -d /etc/apt/keyrings
```
→ デフォルトで `/etc/apt/keyrings` が作成されているのでしなくてもいいかも

ここで、GPGキーの追加をします。
```console
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.asc
```

AIに聞いたら、echoコマンドにて、

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable
```

の文字列を形成し、これを `/etc/apt/sources.list.d/docker.sources` に出力するように説明されたが、ここは一応、公式通りにします。
```console
$ sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

結果を確認してみます。
```console
$ cat /etc/apt/sources.list.d/docker.sources
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```

リポジトリを更新し、Docker Engineのインストール
```
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

なお、インストールしたらDockerサービスも自動で起動してしまうもよう。
```console
$ systemctl status docker.service
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-02-14 17:38:55 JST; 25min ago
TriggeredBy: ● docker.socket
:
```

サービス自動起動も enabled になっています。
```console
$ systemctl list-unit-files | grep docker
docker.service                               enabled         enabled
docker.socket                                enabled         enabled
```

## image のリストア

「`docker info`」を実行したところ、「`Docker Root Dir: /var/lib/docker`」と出るので、すでに ネイティブ Docker Engine に切り替わっているよう。WSL2上から Docker Desktop Engine は使用できません（context を変えたらできる）が、PowerShell 上からは使用できます。

なので、image も volume もない状態

```console
$ docker images
IMAGE   ID             DISK USAGE   CONTENT SIZE   EXTRA

$ docker volume ls
DRIVER    VOLUME NAME
```

では、バックアップしていた image をインポートしてみます。

```console
$ gzip -d images-backup.tar.gz
$ docker load -i images-backup.tar
Loaded image: test_alpine:latest
Loaded image: alpine:latest
```

問題なくインポートできたもようです。
```console
$ docker images
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:latest        25109184c71b         13MB         3.86MB
test_alpine:latest   14fb0b7ecad5        340MB         89.7MB
```

## volume のリストア

次に、volume のリストアをしてみます。もちろん現在は volume は1つもありません。

```console
$ docker volume ls
DRIVER    VOLUME NAME
```

Copilotに教えてもらったコマンドで volume をリストアしてみます。

```console
$ gzip -d mydata.tar.gz
$ docker run --rm -v mydata:/volume -v "$(pwd)":/backup alpine sh -c 'cd /volume && tar xf /backup/mydata.tar .'
```

volumeができました。

```console
$ docker volume ls
DRIVER    VOLUME NAME
local     mydata
```

では、インポートしたイメージから `docker compose` でコンテナを作成実行してみます。

```console
$ docker compose run --rm test-shell
[+]  1/1te 1/1
 ✔ Volume test_alpine_mydata Created                            0.0s
Container test_alpine-test-shell-run-e8f70a6592e7 Creating
Container test_alpine-test-shell-run-e8f70a6592e7 Created
```

すると、勝手に「test_alpine_mydata」という volume を作ってしまいます。

```console
$ docker volume ls
DRIVER    VOLUME NAME
local     mydata
local     test_alpine_mydata
```

compose は、頭にプロジェクト名を付けた名前のボリューム「プロジェクト名_volume」を新規で自動で作るようです。
すでにある volume に紐づけるには、docker-compose.yml に、外部ボリューム指定（external）を追加することで、Composeによる自動作成を抑止し、既存の volume を使用できるようになるみたいです。

```yaml
services:
  test-shell:
    image: test_alpine
    platform: linux/amd64
    tty: true
    stdin_open: true
    working_dir: /home/furuya

    volumes:
      - mydata:/home/furuya

volumes:
  mydata:
    external: true
    name: mydata
```

では、`docker compose run` をしてみます。今度は、新しい volume は作成されないようです。

```console
$ docker compose run --rm test-shell
Container test_alpine-test-shell-run-8407b2976af8 Creating
Container test_alpine-test-shell-run-8407b2976af8 Created
4a65fd6bb82c:~$ ls
Dockerfile          docker-compose.yml  hello               hello.c
```

実際に、volume の実体を確認してみると、同じようにファイルが存在していることがわかります。

```console
$ sudo ls /var/lib/docker/volumes/mydata/_data
Dockerfile  docker-compose.yml  hello  hello.c
```


# Docker Desktop のアンインストール
はい、これで Docker Desktop をアンインストールもOKなので、PowerShellにて、`winget uninstall` で Docler Desktop 	をアンインストールします。

```console
PS C:\Users\hoge> winget uninstall Docker.DockerDesktop
```

そして、WSL2を再起動します。

```console
PS C:\Users\hoge> wsl --shutdown
```

これで完了です。

