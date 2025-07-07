---
title: "WSL2ストア版のインストールについて"
date: 2023-06-26
categories: 
  - "wsl"
tags:
  - "wsl"
---

# はじめに

この記事について、実は2023年2月頃にメモとして作成しており、遅ればせながら今回ブログに上げさせていただきました。  
参考したのは次のサイトです。

【参考】[ASCII.jp： Microsoftストア版WSLが正式版になり、Windows 10でも動作可能に (1/2)](https://ascii.jp/elem/000/004/114/4114859/)

これまで、WSL／WSL2をインストールするには、コンパネの「プログラムと機能」の「Windowsの機能の有効化または無効化」にて、**「Linux用Windowsサブシステム」を有効化**する必要がありました。

更に、WSL2では、wslコマンド（wsl.exe）でWSLのインストールが可能になりました。

コマンド例

- 「`wsl --install`」でUbuntuがインストールされる
- 「`wsl --install -d Debian`」でDebianがインストールされる

しかし、現在のwslコマンドは挙動が違います。

- 「`wsl --install`」で、Microsoftストア版がインストールされる
- 「`wsl --install --inbox -d ディストリ名`」で、コンポーネント版がインストールされる

これまでは、Microsoftストア版はプレビュー版（Ver. 0.X）で、正式版としては、Windowsにコンポーネントとして付属しているコンポーネント版がその位置づけでした。それが2022年11月にMicrosoftストア版が正式版（Ver. 1.X）となりました。

Microsoftストア版をインストールするための条件は次のとおり。

- Windows 10 21H1以降でKB5020030（22H2の累積更新プログラム）がインストールされている
- Windows 11 22H2以降

# WSL2セットアップ

まずは、WSL2をインストールする前のWindows 11のコンポーネントの状態を確認してみます。  
コントロールパネル→「プログラムと機能」→「Windowsの機能の有効化または無効化」

![](/assets/images/4fd329f2ae78f6ceb6a5e865cf97f81a.png?w=696)

![](/assets/images/17ffd22e52e6f48c6df1636493e74f81.png?w=696)

関係ありそうなコンポーネントの状態は次のとおり。

- Hyper-V 【チェックなし】
- Linux用Windowsサブシステム 【チェックなし】
- 仮想マシンプラットフォーム 【チェック有り】

それでは、MicrosoftストアからWSL2をインストールしてみます。  
Microsoftストアにて「Ubuntu」で検索した結果（2023/02/13時点）は、次のとおり。

- Ubuntu
- Ubuntu 22.04.1 LTS
- Ubuntu 20.04.5 LTS
- Ubuntu 18.04.5 LTS

とりあえず、「Ubuntu 22.04.1 LTS」（549.1MB）を選択、インストールしてみます。  
インストール後、スタートメニューから、インストールされた「Ubuntu 22.04.1 LTS」を起動してみると。。。

```
Installing, this may take a few minutes...
WslRegisterDistribution failed with error: 0x8007019e
The Windows Subsystem for Linux optional component is not enabled. 
Please enable it and try again.
See https://aka.ms/wslinstall for details.
Press any key to continue...
```

Linux用Windowsサブシステムコンポーネントが有効になっていないので有効にしろ、とのエラーが出ます。  
そう、確かに「Windowsの機能の有効化または無効化」で有効にしていないからですね。でも。問題ありません。

Microsoftストアで、Ubuntuをインストールしただけでは、Linux用Windowsサブシステム、いわゆるWSLは有効にならないようです。  
そこで、次のコマンドを実行すると、コンポーネントにて有効せずとも、ストア版としてのLinux用Windowsサブシステムをインストールできるようです。

「`wsl --install --no-distribution`」とするとLinuxディストリビューションはインストールされませんが、WSL2としての機能、私自身良くわかっておりませんが、ベースとなる部分（サイズにして456MB）がだけがインストールされるようです。

```console
PS C:\Users\hoge> wsl --install --no-distribution
インストール中: Linux 用 Windows サブシステム
Linux 用 Windows サブシステム はインストールされました。
要求された操作は正常に終了しました。変更を有効にするには、システムを
再起動する必要があります。
```

一旦、Windowsの再起動が必要、その後自動でインストールが始まります。  
Windows再起動、再度、Ubuntu 22.04.1 LTSを起動してみます。

```
Installing, this may take a few minutes...
Please create a default UNIX user account. The username does not need to match
your Windows username.
For more information visit: https://aka.ms/wslusers
Enter new UNIX username: hoge
New password:
Retype new password:
passwd: password updated successfully
Installation successful!
:
```

上手くインストールできたようです。  
ちなみに、コントロールパネルの「Windowsの機能の有効化または無効化」のところは依然、

- Hyper-V 【チェックなし】
- Linux用Windowsサブシステム 【チェックなし】
- 仮想マシンプラットフォーム 【チェック有り】

のままですが、「設定」→「アプリ」→「インストールされているアプリ」のリストの中に、「**Linux用Windowsサブシステム**」 が入っておりました。これが恐らくストア版WSLということになるのだと思います。

これで、コンポーネント版なしに、完全にストア版のWSLを導入できました。

コンポーネント版のWSLが存在する場合とそうでない場合では、「wsl --status」コマンドを実行すれば分かります。

コンポーネント版がない場合

```console
PS C:\Users\hoge> wsl --status
既定のディストリビューション: Ubuntu-22.04
既定のバージョン: 2
WSL1 は、現在のマシン構成ではサポートされていません。
WSL1 を使用するには、"Linux 用 Windows サブシステム" オプション 
コンポーネントを有効にしてください。
```

コンポーネント版がある場合

```console
PS C:\Users\hoge> wsl --status
既定のディストリビューション: Ubuntu-22.04
既定のバージョン: 2
```

また、コンポーネント版、ストア版、どちらでWSL2が動いているかどうかは、上記参考サイトによれば、「wsl --version」コマンドにて、ヘルプが出れば「コンポーネント版」、バージョン表示出れば「ストア版」だそうです。

また「systemd」にも対応可能になったようです。「/etc/wsl.conf」に次を記述。

```
[boot]
systemd = true
```
