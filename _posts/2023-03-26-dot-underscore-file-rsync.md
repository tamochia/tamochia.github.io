---
title: "rsyncの際、アンダースコアで始まるファイル名の転送で失敗する"
date: 2023-03-26
categories: 
  - "linux"
tags: 
  - "tool"
---

SMBが動いているNASにCIFSマウントさせて、そこにrsyncでバックアップを取っているのだけど、毎回このようなエラーが発生して、あるファイルだけが転送（コピー）されないでしまっている現象が生じた。

```
rsync: mkstemp "/mnt/win/.../tcpdf/images/._blank.png.yOZZy3" 
failed: No such file or directory (2)
```

ファイル名の頭に「`._` 」（dot-underscore）が付いているファイルである。  
通常のLinuxのファイルシステム（XFS）上で作る分には何の問題もない

```console
[root@buzz ~]# cd
[root@buzz ~]# touch ._abc
[root@buzz ~]#
```

が、CIFSマウントした先で、「`._abc`」という名前のファイルを作成しようとすると…

```console
[root@buzz ~]# cd /mnt/win/
[root@buzz win]# touch ._abc
touch: `._abc' に touch できません: そのようなファイルやディレクトリはありません
```

問題のファイルは、「`._blank.png.yOZZy3`」となっているが、コピー元のファイルのファイル名は「`_blank.png`」ということで頭にドットは付いていない。

「ドットファイル」、mount などのキーワードで検索すると、「`.DS_Store`」などを無視する「`noappledouble`」オプションの記事が出てくるが、これはsshfsマウントの話であり関係ない。「`--exclude`」で問題のファイルを指定する方法があるが、これだと、バックアップ対象から外されてしまう。

色々調べてみると、rsyncはファイル転送の際に一時ファイルを作成するらしい。エラーメッセージにもあったように、mkstempによって一時ファイルを作成、その際、デフォルトでコピー先ファイルシステム上に作成するようだ。ファイル名がドット・アンダースコアで始まるファイルを受け付けないCIFSマウント先ファイルシステム上で作成しようとするから、「No such file or directory」と失敗するのである。

ということで、解決方法としては、rsync コマンドの「`--temp-dir`」オプションにて、コピー元のファイルシステム上をテンポラリ領域、例えば「`--temp-dir=/tmp/`」として指定すれば良い。結果、エラーは発生しなくなった。
