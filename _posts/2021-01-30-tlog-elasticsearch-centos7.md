---
title: "CentOS 7 によるセッション記録（Tlog）ログのElasticsearchへの転送"
date: 2021-01-30
categories: 
  - "linux"
tags: 
  - "elasticsearch"
---

CentOS 8の新機能 Cookpit の Session Recordingは、ユーザが端末に対し行った操作（入出力）をログに記録し、後で再生して見られる機能であり、その元となっているプログラムがTLogです。

![cockpit](/assets/images/ss-2021-01-30_133453.png)

[GitHub - Scribery/tlog: Terminal I/O logger Package tlog - man pages \| ManKier](https://github.com/Scribery/tlog)

ユーザが行った操作は、tlog-rec あるいは、tlog-rec-session によってJournalログ（syslog）に記録され、それを再生するコマンド、tlog-play が用意されています。Journal に記録してもログローテーションによりいずれは消えてしまいます。監査証跡としてこの操作記録ログ（Tlogログ）を永続的に残す方法として、ログの保存先をMongoDBなど、JSON形式で保存できるストアなどがありますが、tlog-playがその再生元データストアをElasticsearchにできることから、TlogログをRsyslog経由でElasticsearchに転送する方法があります。

今回は、それらをCentOS 7にて実装したいと思います。

<!--more-->

## Tlogのインストール

こちらを参考にしました。

[CentOS 7にターミナルの操作ログをSyslogでJSON形式に記録、再生もできる『tlog』をインストールする \| 俺的備忘録 〜なんかいろいろ〜](https://orebibou.com/ja/home/201701/20170119_001/)

TLogはCentOS 8（RHEL8）では標準で用意されているが、CentOS 7に導入する場合はソースパッケージからビルドする必要があります。

まずはビルドのために必要なパッケージをインストールしておきます。

```console
$ sudo yum install libtool autoconf git json-c-devel libcurl libcurl-devel systemd-devel libutempter-devel
```

git cloneして、ビルド、そしてインストールします。

```console
$ git clone https://github.com/Scribery/tlog
$ cd tlog
$ autoreconf -i -f
$ ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var 
$ make
$ sudo make install
```

早速、tlog-recを実行して操作ログの記録を開始してみます。

```console
$ tlog-rec --writer=file --file-path=tlog.log
tlog-rec: error while loading shared libraries: libtlog.so.0: cannot open shared object file: No such file or directory
```

が、早速エラーが発生。tlogのライブラリファイルのシンボリックリンクを、/lib64/に張ります。

```console
$ sudo ln -s /usr/lib/libtlog.so.0 /lib64/
```

操作ログの記録をカレントディレクトリのファイル「tlog.log」に保存します。

```console
$ tlog-rec --writer=file --file-path=tlog.log
```

上のコマンドを実行した時点で記録が開始され、「exit」コマンドで記録を終了させます。

操作ログが記録されたファイル「tlog.log」を再生してみます。

```console
$ tlog-play --reader=file --file-path=tlog.log
```

ライターとしてjournaldに渡すことも可能です。

```console
$ tlog-rec --writer=journal
```

journalログを確認

```console
$ journalctl -e
:
1月 18 08:35:26 alice.example.org tlog-rec[20150]: {"ver":"2.3","host":"alice.example.org","rec":"7287e2aa48304
```

または、

```console
$ sudo tail -100 /var/log/messages
```

で確認してみます。  
再生は、recの値を指定して実行します。

```console
$ tlog-play -r journal -M TLOG_REC=7287e2aa48304........
```

## Tlogでのユーザのセッション自動記録

ユーザがログインしたと同時にセッション記録を始める場合、ログインシェルを /usr/bin/tlog-rec-session を使用するようにすれば良いようです。tlog-rec-session は端末の記録をjournaldに送ります。

ユーザ hoge のログインシェルを置き換える方法としては、usermodコマンドを使用して、

```console
$ sudo usermod -s /usr/bin/tlog-rec-session hoge
```

とする方法がありますが、ここでは、上のように指定するのではなく、CentOS 8 に同梱されているCockpitでのセッション録画ソリューションでも利用しているSSSDにてtlog-rec-sessionを呼び出すように設定してみます。

[セッションの録画 Red Hat Enterprise Linux 8 \| Red Hat Customer Portal RHEL8で端末の入出力を記録する - 赤帽エンジニアブログ](https://rheb.hatenablog.com/entry/tlog)

SSSD (System Security Service Daemon)とは、リモートディレクトリと認証メカニズムへのアクセスを管理する一連のデーモンを提供する、とのことですが、LDAPやADなどの複数の認証サービスの管理を行うデーモンらしい。ここでは、このSSSDを使用して、tlogの録画を行うユーザを指定することにします。

SSSDのインストール

```console
$ sudo yum install sssd
```

設定ファイルを2つ（sssd.conf、sssd-session-recording.conf）作成します。

**\[/etc/sssd/sssd.conf\]**

```
[domain/local]
id_provider = files

[sssd]
domains = local
services = nss, pam, ssh, sudo
```

**\[/etc/sssd/conf.d/sssd-session-recording.conf\]**

```
[session_recording]
scope = all
# users = 
# groups =
```

とりあえず、「`scope = all`」とすることで、すべてのユーザを記録対象とします。なお、「`scope = some`」を指定することで、記録するユーザ（hoge）を個別に設定することが可能です。

```
[session_recording]
scope = some
users = hoge
# groups =
```

それぞれ、必ずrootしかreadできないようにパーミッションを設定しておきます。

```console
$ sudo chmod 600 /etc/sssd/sssd.conf
$ sudo chmod 600 /etc/sssd/conf.d/sssd-session-recording.conf
```

通常ログインシェルはロケール設定を環境変数からではなく、/etc/locale.conf ファイルから読み込みます。tlog-rec-session は実際のシェルではなく、/etc/locale.confを読むことはできません。そこで、PAMの pam\_env.so モジュールを使用して、tlog-rec-session が起動する前にロケールを読み込むように /etc/pam.d/system-auth に次の1行を追記します。

**\[ /etc/pam.d/system-auth \]**

```
session    required    pam_env.so  readenv=1  envfile=/etc/locale.conf
```


<span style="color: red;">\[2021/10/13追記\] と、ここに説明されてますが、下手に設定すると sudo できなくなる可能性もありますし、単に警告を黙らせるだけの設定なので、無理に行う必要はありません。</span>

続いて、tlog-rec-session が、シェルの名前を含む名前（例えば /bin/bash の場合は、tlog-rec-session-shell-bin-bash という名前）で起動された場合、tlog-rec-session がその指定したシェルで起動してくれるようになるそうなので、次のようにシンボリックリンクを作成しておきます。

```console
$ sudo ln -s /usr/bin/tlog-rec-session /usr/bin/tlog-rec-session-shell-bin-bash
```

tlog-rec-session の後に "-shell" を付けて、その後 /bin/zsh だったら、"/" を "-" （ダッシュ）に変えて、tlog-rec-session-shell-bin-zsh のようにします。

では、sssdを起動してみます。また、自動起動されるようにも設定します。

```console
$ sudo systemctl start sssd.service
$ sudo systemctl enable sssd.service
```

では、一旦ログアウトし、再度ssh接続してみます。

```console
$ ssh alice.example.org
hoge@alice.example.org s password:
Last login: Sat Jan 16 08:47:48 2021 from 1.2.3.4
許可がありません 
Failed creating lock file /var/run/tlog/session.18.lock 
Assuming session was unlocked

ATTENTION! Your session is being recorded!
```

「ATTENTION! Your session is being recorded!」と出れば、そのユーザの操作は記録状態になっています。表示されていない場合は、ユーザログイン時に sssd に問合せしていない可能性がありますので。/etc/nsswitch.conf を確認して、passwd が、ファイルではなく sssd が優先になっているかどうか確認します。

**\[ /etc/nsswitch.conf \]**

```
passwd:     sss files
shadow:     files sss
group:      sss files
```

/etc/nsswitch.conf を修正後、ログインし直せば解決しているはず。しかし、次のような警告が表示されているのが気になります。

```
許可がありません
Failed creating lock file /var/run/tlog/session.18.lock
Assuming session was unlocked
```

「許可がありません( Permission denied)」ということで、tlog-rec-session が root権限で実行されていないようです。

```console
$ ps aux | grep tlog
hoge   2161  0.0  0.3 233768  3920 pts/0    Ss+  09:07   0:00 -tlog-rec-session
hoge   2186  0.0  0.0 112824   968 pts/1    S+   09:08   0:00 grep --color=auto tlog
```

そこで、tlog-rec-session がroot権限で起動できるようSUID/SGIDを付与しておきます。

```console
$ sudo chmod u+s /usr/bin/tlog-rec-session
$ sudo chmod g+s /usr/bin/tlog-rec-session
$ ls -l /usr/bin/tlog-rec-session
-rwsr-sr-x 1 root root 31472 12月 20 10:39 /usr/bin/tlog-rec-session
```

また、ロックファイル「/var/run/tlog/session.18.lock」が作成できないとありますが、/var/run/tlog ディレクトリが存在しないので、ロックファイルが作成できないのは当たりまえ。まずは、/var/run 配下に tlog ディレクトリを作成します。

```console
$ sudo mkdir /var/run/tlog
```

ただ、/var/run 配下にディレクトリを作成しても、ログアウト時に消えるので、/usr/lib/tmpfiles.d に、次のような内容の「var-run.conf」ファイルを作成します。

（参考）「CentOS 7～Stream 8 : /var/run 直下に作ったディレクトリが消えないようにする - eTuts+ Server Tutorial」

**\[/usr/lib/tmpfiles.d/var-run.conf\]**

```
d  /var/run/tlog   0755  root  root -
```

再度ログインし直すと、先ほどの警告は消えているはずです。

## Tlogログをrsyslog経由でElasticsearchへ送る

tlog-play コマンドはリーダとして Elasticsearch を指定することが可能です。ということは、Tlog ログを何らかのかたちで Elasticsearchへ送る方法を考えますが、一番シンプルな方法として、rsyslog にて Elasticsearch にログを送る方法があります。

Rsyslog の Elasticsearch 用モジュール「rsyslog-elasticsearch」のインストール

```console
$ sudo yum install rsyslog-elasticsearch
```

「GitHub - Scribery/tlog: Terminal I/O logger」の「Recording sessions to Elasticsearch」の章を参考に、設定ファイル「/etc/rsyslog.d/elasticsearch.conf」を作成します。

**\[ /etc/rsyslog.d/elasticsearch.conf \]**

```ruby
#module(load="omelasticsearch")

$MaxMessageSize 16k
$ModLoad omelasticsearch

template(name="tlog" type="list") {
    constant(value="{")
    property(name="timegenerated"
        outname="timestamp"
        format="jsonf"
        dateFormat="rfc3339")
    constant(value=",")
    property(name="msg"
        regex.expression="{\\(.*\\)"
        regex.submatch="1")
    constant(value="\n")
}

if $programname == '-tlog-rec-session' then {
    action(name="tlog-elasticsearch"
        type="omelasticsearch"
        server="172.xx.xx.xx"
        searchIndex="oplog-tlog"
        searchType="tlog"
        bulkmode="on"
        template="tlog")
#   action(name="tlog-file"
#       type="omfile"
#       file="/var/log/tlog.log"
#       fileCreateMode="0600"
#       template="tlog")
}
```

まず、

```ruby
#   action(name="tlog-file"
#       type="omfile"
#       file="/var/log/tlog.log"
#       fileCreateMode="0600"
#       template="tlog")
}
```

部分は、ログをローカルの /var/log/tlog.log に保管する場合の設定。ローカルで確認したい場合は、ここのコメントを外します。

次に、

```ruby
property(name="timegenerated"
    outname="timestamp"
    format="jsonf"
    dataFormat="rfc3339")
```

は、Kibanaで検索したりする場合に必要な timestamp のフィールド。しかも、ISO8601(RFC3339)形式、つまり、

```
"2021-01-18T11:21:53.933+09:00"
```

のような形式でのフィールドが必要ということで、設けているようです。

次に、

```ruby
server="172.xx.xx.xx"
searchIndex="oplog-tlog"
searchType="tlog"
```

は、それぞれ Elasticsearch が動いているサーバのIPアドレス、Elasticsearch のインデックス名とタイプ名であり、それぞれ任意ですが、ここではインデックス名を「oplog-tlog」としています。

次に、

```ruby
if $programname == '-tlog-rec-session' then {
```

の if 文の部分、journalに記録されたログを確認すると、

```console
$ journalctl -e
:
1月 18 12:08:30 alice.example.org -tlog-rec-session[6399]: {"ver":"2.3","host":"alice.example.org","rec":"34c123838....
```

となっているので、「 `$programname == '-tlog-rec-session'` 」と条件に指定しています。

では、rsyslogを起動してみます。また、自動起動されるようにも設定します。

```console
$ sudo systemctl start rsyslog.service
$ sudo systemctl enable rsyslog.service
```

## ElasticsearchからのTlog再生

Elasticserch が動いているサーバにて、次のコマンドにてインデックス一覧を確認します。

```console
$ curl http://localhost:9200/_cat/indices?v
```

するとその一覧の中に、Tlog転送元サーバの rsyslog にて設定したインデックス名「oplog-tlog」が存在しているのが確認できます。存在しない場合は、一旦Tlog転送元サーバにて、Tlogログを新たに取得できるよう適当な操作を行ってみてください。

```console
$ curl http://localhost:9200/_cat/indices?v
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   oplog-tlog                      IH2WjA9LRUKWigefyLmCqw   1   1         36            0    197.5kb        197.5kb
```

では、インデックスの中身を「pretty」を指定して、curl で確認してみます。

```console
$ curl http://localhost:9200/oplog-tlog/_search?pretty
{
  "took" : 1,
  "timed_out" : false,
:
    "hits" : [
      {
        "_index" : "oplog-tlog",
        "_type" : "tlog",
        "_id" : "xnloE3cBB5gkoWLJC0_B",
        "_score" : 1.0,
        "_source" : {
          "timestamp" : "2021-01-18T11:52:01.853026+09:00",
          "ver" : "2.3",
          "host" : "alice.example.org",
          ：
```

このデータのうち、クエリーにて必要な部分を tlog-play で再生するのですが、session（session id）でクエリーすると良さそうです。まず、Tlogログ転送元のホストにて、Session ID 確認しておきます。

```console
$  cat /proc/self/sessionid
1234
```

ログ収集側（Elasticsearch が動いている）ホストにて、tlog-play コマンドで、Elasticsearchを指定して再生してみます。当然、こちらのホストにも Tlog をインストールしておく必要があります。

```console
$ tlog-play -r es --es-baseurl=http://localhost:9200/oplog-tlog/_search --es-query='session:1234'
```

いつまでたっても再生されない場合があります。これは一時停止中になっている場合があるので、キーボードの「.」（ドット）を押すと、次のステップまでスルーしてくれます。

Kibanaでログを確認するとこんな感じ。

![kibana](/assets/images/ss-2021-01-30_132222.png)
