---
title: "WSL2からHyper-Vの仮想マシンにSSHする"
date: 2023-08-15
categories: 
  - "linux"
tags: 
  - "wsl"
---

# はじめに

WindowsでLinux系システムの開発構築を行う場合の手段としては、WSL2にてDockerを利用する方法が手っ取り早いと思いますが、コンテナでの開発ではなく、Linux環境丸ごと利用して開発を行いたい場合は、仮想環境にて仮想マシンにLinuxを導入して開発環境を構築するのが手っ取り早いと思います。

仮想環境といえば、WindowsにははじめからHyper-Vが同梱されているので、そちらで仮想マシンは作成できます。Hyper-Vマネージャーのコンソールを使用すると日本語が使用できません（日本語表示が化ける）が、sshdを起動させて、Windows上のSSHクライアントからSSHでアクセスするようにすれば問題ありません。

Hyper-Vの仮想マシンにSSHアクセスしたい場合、PowerShellを利用する方法がありますが、WSL2上で何か作業しながら、PowerShellを起動するのも面倒なので、どうせならWSL2上からSSHでHyper-Vの仮想マシンにアクセスできるようにしたいものです。

WSL2もHyper-Vのハイパーバイザー上、つまり仮想マシンとして動いているはずですが、WSL2とHyper-Vマネージャーで個々に作成した仮想マシンとでは、それぞれネットワークが異なるので通信できません。それを何とかして、WSL2からHyper-Vの仮想マシンにSSHさせてみようと思います。

# Hyper-V仮想マシンとWSL2のネットワーク

Hyper-Vを有効にすると、「`vEthernet(Default Switch)`」の仮想ネットワークアダプタ（NIC）が追加されます。  
Hyper-Vマネージャーから、仮想スイッチマネージャーを確認すると、仮想スイッチとして「`Default Switch`」が存在しています。

一方、WSL2のネットワークは、Hyper-Vの仮想スイッチである「`WSL`」を使用します。  
そして、ホスト側との接続は、「`vEthernet(WSL)`」の仮想NICで接続されます。

つまり、ホスト側（Windows OS側）には、仮想NICとして「`vEthernet(Default Switch)`」と「`vEthernet(WSL)`」が存在します。

![](/assets/images/virtual-network01.png?w=1000)


図のように、

- Hyper-V仮想マシンが接続しているネットワークは、172.27.224.0/20
- WSL2(Ubuntu)が接続しているネットワークは、172.25.208.0/20

で、それぞれ異なります。それぞれ、ホスト内部のNATの機能（Windows NAT, WinNAT）を経由して外部ネットワークにアクセスすることは可能ですが、次の図のように、WSL2からHyper-V上の仮想マシンにアクセスすることはできません。

![](/assets/images/virtual-network02.png?w=1000)

PowerShellからのHyper-V上の仮想マシンへのSSHアクセスは可能です。

次のサイトにあるように、それぞれネットワークインターフェース間のForwardingをEnabledに設定すると、Hyper-Vの仮想マシンとWSL2が互いに通信できるようになり、WSL2からのSSHアクセスも可能になります。

**参考** [WSL2とHyper-V仮想マシンの間で通信ができるようにする - メモの日々(2020-12-08)](http://ogawa.s18.xrea.com/tdiary/20201208p01.html)

ですが、大きな問題として、ホストであるWindowsの再起動の度に、これら仮想スイッチ（vSwitch）が再構築（削除と作成）されるので、ネットワークアドレスが変わり、DHCPのIPアドレスレンジも変わり、それぞれ割り当てられるIPアドレスが変わわってしまいます。それゆえSSHする際は、その都度、Hyper-Vマネージャーのコンソールから仮想マシンにアクセスし、仮想マシンのIPアドレスを確認する必要があります。

**参考** [WSL バージョンの比較 - Microsoft Learn](https://learn.microsoft.com/ja-jp/windows/wsl/compare-versions)

直接ホストの物理NICにブリッジする「外部ネットワーク」タイプの仮想スイッチに変更すれば、固定のIPアドレスを振れそうですが、開発環境の仮想マシンを社内LAN上に晒したくないですし、Windowsが用意した仮想スイッチを色々いじるのもしたくはないですね。

では、どうしたらよいか、その一解決方法をご紹介する前に、現状のWindows仮想ネットワークを確認します。

# Windows仮想ネットワークアダプタ

ネットワークアダプタの一覧を表示させるには、**設定**からでは、

「**ネットワークとインターネット**」＞「**ネットワークの詳細設定**」＞「**ネットワークアダプターオプションの詳細**」

また、**コントロールパネル**からでは、

「**ネットワークとインターネット**」＞「**ネットワークと共有センター**」＞「**アダプターの設定の変更**」

にて確認できますが、いつしか仮想ネットワークアダプタ（vEthernet）が非表示になりました。

「**ネットワークとインターネット**」＞「**ネットワークの詳細設定**」＞「**ハードウェアと接続のプロパティ**」

で確認はできますが、有効／無効化はできないようです。

PowerShellで確認してみましょう。`Get-NetAdapter` コマンドで情報が取得できます。

```console
PS C:\Users\hoge> Get-NetAdapter

Name                    InterfaceDescription    ifIndex Status       MacAddress
----                    --------------------    ------- ------       ----------
Bluetooth ネットワーク…  Bluetooth Device (Pers…      23 Disconnected 0C-96-E6-XX-XX-XX
イーサネット             Intel(R) Ethernet Conn…      20 Disconnected C8-F7-50-XX-XX-XX
Wi-Fi                   Qualcomm QCA61x4A 802.…      18 Up           0C-96-E6-XX-XX-XX
```

仮想ネットワークアダプタ（vEthernet）を表示させるためには「`-IncludeHidden`」オプションを付加します。

```console
PS C:\Users\hoge> Get-NetAdapter -IncludeHidden

Name                      InterfaceDescription               ifIndex Status       MacAddress
----                      --------------------               ------- ------       ----------
Bluetooth ネットワーク…  Bluetooth Device (Personal Area …      23 Disconnected 0C-96-E6-XX-XX-XX
ローカル エリア接続* 2    Microsoft Wi-Fi Direct Virtua...#2      22 Disconnected 1E-96-E6-XX-XX-XX
ローカル エリア接続* 9    WAN Miniport (PPPOE)                    21 Disconnected
vSwitch (Default Switch)  Hyper-V Virtual Switch Extension…      17 Up
vSwitch (WSL)             Hyper-V Virtual Switch Extens...#2      43 Up
イーサネット              Intel(R) Ethernet Connection (4)…      20 Disconnected C8-F7-50-XX-XX-XX
ローカル エリア接続* 6    WAN Miniport (IKEv2)                    19 Disconnected
Wi-Fi                     Qualcomm QCA61x4A 802.11ac Wirel…      18 Up           0C-96-E6-XX-XX-XX
vEthernet (Default Swit… Hyper-V Virtual Ethernet Adapter        26 Up           00-15-5D-XX-XX-XX
Teredo Tunneling Pseudo…                                         15 Not Present
```

なんかいっぱい出てきました。Nameに「`WSL`」と付くものには、「vSwitch」と「vEthernet」と2種類あります。

```console
PS C:\Users\hoge> Get-NetAdapter -IncludeHidden | Where-Object {$_.Name -like '*WSL*'} | Format-List

Name                       : vSwitch (WSL)
InterfaceDescription       : Hyper-V Virtual Switch Extension Adapter #2
InterfaceIndex             : 49
MacAddress                 :
MediaType                  : 802.3
PhysicalMediaType          : Unspecified
InterfaceOperationalStatus : Up
AdminStatus                : Up
LinkSpeed(Gbps)            : 10
MediaConnectionState       : Connected
ConnectorPresent           : False
DriverInformation          : Driver Date 2006-06-21 Version 10.0.22621.1 NDIS 6.83

Name                       : vEthernet (WSL)
InterfaceDescription       : Hyper-V Virtual Ethernet Adapter #2
InterfaceIndex             : 51
MacAddress                 : 00-15-5D-XX-XX-XX
MediaType                  : 802.3
PhysicalMediaType          : Unspecified
InterfaceOperationalStatus : Up
AdminStatus                : Up
LinkSpeed(Gbps)            : 10
MediaConnectionState       : Connected
ConnectorPresent           : False
DriverInformation          : Driver Date  Version
```

「**vEthernet**」の方は「MacAddress」があるので仮想NICであることが分かります。「**vSwitch**」の方は、「Hyper-V仮想スイッチ拡張アダプタ (Hyper-V Virtual Switch Extension Adapter)」とありますが、仮想スイッチということで良いのかな。

続いて、`Get-NetIpInterface` コマンドで、各ネットワークインターフェース情報を見てみます。  
`Where-Object` でパイプして IPv4 の情報だけを抽出してみます。

```console
PS C:\Users\hoge> Get-NetIPInterface | Where-Object {$_.AddressFamily -eq 'IPv4'}

ifIndex InterfaceAlias              AddressFamily NlMtu(Bytes) InterfaceMetric Dhcp     ConnectionState
------- --------------              ------------- ------------ --------------- ---- ---------------
45      vEthernet (WSL)             IPv4                  1500            5000 Disabled Connected
26      vEthernet (Default Switch)  IPv4                  1500            5000 Disabled Connected
22      ローカル エリア接続* 2      IPv4                  1500              25 Enabled  Disconnected
23      Bluetooth ネットワーク接続  IPv4                  1500              65 Enabled  Disconnected
13      ローカル エリア接続* 1      IPv4                  1500              25 Enabled  Disconnected
20      イーサネット                IPv4                  1500               5 Enabled  Disconnected
1       Loopback Pseudo-Interface 1 IPv4            4294967295              75 Disabled Connected
18      Wi-Fi                       IPv4                  1500              45 Enabled  Connected
```

`Get-NetAdapter` コマンドがL2レベルで、`Get-NetIPInterface` コマンドがL3レベルの情報を取得する、ということでしょうか。

# 新規に仮想スイッチを作成する

それでは、WSL2からHyper-Vの仮想マシンにSSHアクセス可能にするための解決策として、「内部ネットワーク」タイプの仮想スイッチを利用した方法をご紹介します。

Hyper-Vマネージャーの仮想スイッチマネージャーを開いて、新しい仮想ネットワークスイッチ「`Host-Only Switch`」を作成してみます。接続の種類を「**内部ネットワーク**」します。

- 「外部ネットワーク」
    - 上で説明した通り、ホストの物理NICに直接ブリッジする
- 「内部ネットワーク」
    - Hyper-V上の仮想マシン同士、仮想マシンとホストとのアクセスは可、外部ネットワークとのアクセスは不可
- 「プライベートネットワーク」
    - Hyper-V上の仮想マシン同士のみアクセスか、ホストや外部ネットワークとのアクセスは不可

**参考** [仮想スイッチの種別と用途：Windows 10 Hyper-V入門 - ＠IT](https://atmarkit.itmedia.co.jp/ait/articles/2008/14/news018.html)

![](/assets/images/01e4bbaee683b3e382b9e382a4e38383e38381e381aee4bd9ce68890.png?w=723)

次に、仮想マシンの設定を開き、「ハードウェアの追加」で「ネットワークアダプター」を1つ追加します。

![](/assets/images/02e3838fe383bce38389e382a6e382a7e382a2e381aee8bfbde58aa0.png?w=717)

追加した「ネットワークアダプター」の設定にて、仮想スイッチを作成した「Host-Only Switch」に指定します。

![](/assets/images/03e3838de38383e38388e383afe383bce382afe382a2e38380e38397e382bf-1.png?w=717)

これで、仮想マシンにNICが1つ追加（eth1）されたことになります。

`Get-NetIPInterface` コマンドにて確認してみます。

```console
PS C:\Users\hoge> Get-NetIPInterface | Where-Object {$_.InterfaceAlias -like 'vEthernet*'}

ifIndex InterfaceAlias               AddressFamily NlMtu(Bytes) InterfaceMetric Dhcp     ConnectionState PolicyStore
------- --------------               ------------- ------------ --------------- ----     --------------- -----------
57      vEthernet (Host-Only Switch) IPv6                  1500              15 Enabled  Connected       ActiveStore
45      vEthernet (WSL)              IPv6                  1500            5000 Enabled  Connected       ActiveStore
26      vEthernet (Default Switch)   IPv6                  1500            5000 Enabled  Connected       ActiveStore
57      vEthernet (Host-Only Switch) IPv4                  1500              15 Enabled  Connected       ActiveStore
45      vEthernet (WSL)              IPv4                  1500            5000 Disabled Connected       ActiveStore
26      vEthernet (Default Switch)   IPv4                  1500            5000 Disabled Connected       ActiveStore
```

さらに、IPv4の「`vEthernet (Host-Only Switch)`」に限定して、`Get-NetIPInterface` コマンドにて確認します。

```console
PS C:\Users\hoge> Get-NetIPAddress -InterfaceAlias "vEthernet (Host-Only Switch)" | Where-Object {$_.AddressFamily -eq 'IPv4'}

IPAddress         : 169.254.188.169
InterfaceIndex    : 57
InterfaceAlias    : vEthernet (Host-Only Switch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 16
PrefixOrigin      : WellKnown
SuffixOrigin      : Link
AddressState      : Preferred
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore
```

リンクローカルアドレスが割り振られているようです。  
「`vEthernet (Host-Only Switch)`」は、「設定」＞「ネットワークとインターネット」＞「ネットワークの詳細設定」＞「ネットワークアダプターオプションの詳細」でもプロパティが表示されるので、そこで設定変更しても良いのですが、せっかくなので、PowerShellにて `New-NetIPAddress` コマンドで設定してみましょう。

- IPアドレス（ホスト側）: `10.0.0.1`
- ネットマスク: `255.255.255.0`

とします。

```console
PS C:\Users\hoge> New-NetIPAddress 10.0.0.1 -PrefixLength 24 -ifIndex 57
```

再度、`Get-NetIPInterface` コマンドにて確認します。

```console
PS C:\Users\hoge> Get-NetIPAddress -InterfaceAlias "vEthernet (Host-Only Switch)" | Where-Object {$_.AddressFamily -eq 'IPv4'}

IPAddress         : 10.0.0.1
InterfaceIndex    : 57
InterfaceAlias    : vEthernet (Host-Only Switch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Preferred
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore
```

仮想マシンのほうに追加したNIC（eth1）については、Linux上で設定します。ここでは、`10.0.0.10` で設定します。設定方法については割愛あいます。

一方、デフォルトの eth0 については動的IPアドレス割り当てのままにしておきます。

仮想マシン（Linux）で、 `ip` コマンドで確認するとこんな感じ。

```console
[hoge@centos6 ~]$ ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    inet 172.27.237.23/20 brd 172.27.239.255 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    inet 10.0.0.10/24 brd 10.0.0.255 scope global eth1
```

現在のネットワーク構成はこのようになっています。

![](/assets/images/virtual-network03.png?w=1000)

仮想マシン（Linux）にて、ルーティングテーブルを確認します。

```console
[hoge@centos6 ~]$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth1
172.27.224.0    0.0.0.0         255.255.240.0   U     0      0        0 eth0
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
169.254.0.0     0.0.0.0         255.255.0.0     U     1003   0        0 eth1
0.0.0.0         172.27.224.1    0.0.0.0         UG    0      0        0 eth0
```

デフォルトゲートウェイは「`172.27.224.1`」、即ち「`vEthernet(Default Switch)`」のIPアドレスになっています。

NICの eth1（`10.0.0.1`）はホストとのアクセスが可能ですので、結果として、WSL2側からSSHが可能になるはずです。

![](/assets/images/virtual-network04.png?w=1000)

では、WSL2側にて、`ssh 10.0.0.10` コマンドをたたいてみます。

```console
hoge@slinky:~$ ssh 10.0.0.10
Unable to negotiate with 10.0.0.10 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss
```

もし、このようなメッセージが表示されて、結果接続できなかった場合は、クライアント側、即ちWSL2側のOpenSSHのバージョンがサーバ側に比べ新しい場合にこのようなことになるようです。クライアント側にて、古いタイプの鍵でも使えるように、`.ssh/config` にて次のように設定しておきます。

```console
hoge@slinky:~$ cat .ssh/config
Host 10.0.0.10
        HostKeyAlgorithms ssh-rsa,ssh-dss
        PubkeyAcceptedAlgorithms +ssh-rsa
```

ついに、WSL2からHyper-Vの仮想マシンにSSHアクセスできるようになりました。

```console
hoge@slinky:~$ ssh 10.0.0.10
hoge@10.0.0.10s password:
Last login: Tue Aug 15 13:39:46 2023 from 172.27.224.1
[hoge@centos6 ~]$
```

ホスト（Windows）の再起動による仮想スイッチ「`vSwitch(Default Switch)`」の再構築によって、仮想マシン（Linux）の eth0 側のIPアドレスが変わったとしても、eth1 側のIPアドレスは固定なので影響ありません。

違う方法が他にあると思いますが、自分の場合はこれで満足しています。
