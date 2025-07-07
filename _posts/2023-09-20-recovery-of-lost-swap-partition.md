---
title: "HDD交換にて消失したswap領域の復旧"
date: 2023-09-20
categories: 
  - "linux"
tags:
  - "linux"
  - "raid"
---

# swap領域が消えた

先般の投稿「[ソフトウェアRAID構成におけるHDD交換 \| tamo's archives](hdd-replacement-in-software-raid.html)」で、HDD交換したLinuxの物理サーバですが、気づいたら Syslogにて「`Timed out waiting for device ...`」が頻発していました。

```
:
Sep 19 13:57:45 hoge systemd[1]: dev-mapper-cl\x2dswap.device: Job dev-mapper-cl\x2dswap.device/start timed out.
Sep 19 13:57:45 hoge systemd[1]: Timed out waiting for device dev-mapper-cl\x2dswap.device.
Sep 19 13:57:45 hoge systemd[1]: Dependency failed for /dev/mapper/cl-swap.
Sep 19 13:57:45 hoge systemd[1]: dev-mapper-cl\x2dswap.swap: Job dev-mapper-cl\x2dswap.swap/start failed with result 'dependency'.
Sep 19 13:57:45 hoge systemd[1]: dev-mapper-cl\x2dswap.device: Job dev-mapper-cl\x2dswap.device/start failed with result 'timeout'.
:
```

freeコマンドで確認してみます。

```console
$ free
              total        used        free      shared  buff/cache   available
Mem:       32814316     1813788    29246188        2304     1754340    30577800
Swap:             0           0           0
```

swapの領域がまったくないですね。。。

```console
$ cat /proc/swaps
Filename                                Type            Size            Used            Priority
$
```

swapパーティションが無いよう。。。

```console
$ grep swap /etc/fstab
/dev/mapper/cl-swap     swap                    swap    defaults        0 0
```

`/etc/fstab` には定義してあるようではあるが。。。ディレクトリ「`/dev/mapper`」には「`cl-swap`」なるものは存在していません。

```console
$ ls -l /dev/mapper/
合計 0
crw------- 1 root root 10, 236  9月 14 12:14 control
```

さてどうしよう。とりあえず、ボリュームの状態を見てみます。。。

<!--more-->

# ボリュームの状態確認

まずは、

- LV（論理ボリューム） … コマンド `lvs`、`lvdisplay`
- VG（ボリュームグループ） … コマンド `vgs`、`vgdisplay`
- PV（物理ボリューム） … コマンド `pvs`、`pvdisplay`

それぞれ、ボリュームレイヤーを簡易的に確認するコマンド「`lvs`」、「`vgs`」、「`pvs`」で確認してみます。

```console
$ sudo lvs
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to /dev/sdc1).
  LV   VG Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  swap cl -wi-----p- 18.62g
```

```console
$ sudo vgs
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to /dev/sdc1).
  VG #PV #LV #SN Attr   VSize  VFree
  cl   3   1   0 wz-pn- 18.62g    0
```

3つの物理ボリュームからなるボリュームグループ「`cl`」の上で、論理ボリューム「`swap`」がある、ということになりますが、UUIDが「`4Qn12e-ytJo-....`」のデバイスが見つからないと警告が出ています。

最後に物理ボリュームを確認してみます。

```console
$ sudo pvs
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to /dev/sdc1).
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  cl lvm2 a-- <6.21g    0
  /dev/sdb1  cl lvm2 a-- <6.21g    0
  [unknown]  cl lvm2 a-m  <6.21g    0
```

おっと、ここで「`[unknown]`」が出ました。  
ここはRAIDを構成しているパーティション（物理ボリューム）の「`/dev/sdc1`」が出てこなくてはなりません。（参照: 先般の投稿「ソフトウェアRAID構成におけるHDD交換」）

論理ボリュームを「`lvdisplay`」コマンドにてより詳しく見てみます。

```console
$ sudo lvdisplay
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to /dev/sdc1).
  --- Logical volume ---
  LV Path                /dev/cl/swap
  LV Name                swap
  VG Name                cl
  LV UUID                6wf2LY-ewJC-wMfE-otL8-eM5d-WpSh-EzWjbx
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2021-06-28 10:11:16 +0900
  LV Status              NOT available (partial)
  LV Size                18.62 GiB
  Current LE             4767
  Segments               3
  Allocation             inherit
  Read ahead sectors     auto
```

「`LV Status`」が、一部利用不可となっているのが確認できます。しかも論理ボリュームを作成したのが、2021/06/28 と、2年前、このサーバを最初にセットアップした時の日付になっています。つまり、HDD交換した2023年8月ではないということです。

コマンド「`blkid`」にてブロックデバイスの属性を確認してみます。

```console
$ sudo blkid
/dev/sda1: UUID="e8c486bd-2a79-4936-9bfe-e7c75e985915" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="0005ca63-01"
/dev/sda2: UUID="4FrV6q-DwfQ-cxQs-j20E-fkCf-MWo2-0oVXmp" TYPE="LVM2_member" PARTUUID="0005ca63-02"
/dev/sda3: UUID="9090b326-c255-5cef-09af-61c551ba0d3d" UUID_SUB="18d79a61-cdf8-cc14-6f89-8e38b8b54b5b" LABEL="localhost.localdomain:root" TYPE="linux_raid_member" PARTUUID="0005ca63-03"
/dev/sdb1: UUID="sPsefj-bjgH-4w56-4Ukn-2YcP-I4X7-nK0pR1" TYPE="LVM2_member" PARTUUID="0007a577-01"
/dev/sdb2: UUID="9090b326-c255-5cef-09af-61c551ba0d3d" UUID_SUB="72187d18-2e55-8b31-e827-849c5a3ede00" LABEL="localhost.localdomain:root" TYPE="linux_raid_member" PARTUUID="0007a577-02"
/dev/sdc2: UUID="9090b326-c255-5cef-09af-61c551ba0d3d" UUID_SUB="afc78a6f-40dd-7a4d-207d-4987ca348537" LABEL="localhost.localdomain:root" TYPE="linux_raid_member" PARTLABEL="primary" PARTUUID="7fc74dcd-579c-4bf3-a9e0-b8830e21527f"
/dev/md127: UUID="a4ef59e1-ff85-4bb0-80c1-eb13bab60840" BLOCK_SIZE="512" TYPE="xfs"
/dev/sdc1: PARTLABEL="primary" PARTUUID="ce65c6eb-69b3-49bd-b570-17f8a9845ba4"
```

ちょっと分かりづらいので、ツリー表示してくれる「`lsblk`」コマンドで確認してみます。

```console
$ sudo lsblk -f
NAME      FSTYPE            LABEL                      UUID                                   MOUNTPOINT
sda
├─sda1    xfs                                          e8c486bd-2a79-4936-9bfe-e7c75e985915   /boot
├─sda2    LVM2_member                                  4FrV6q-DwfQ-cxQs-j20E-fkCf-MWo2-0oVXmp
└─sda3    linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127 xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
sdb
├─sdb1    LVM2_member                                  sPsefj-bjgH-4w56-4Ukn-2YcP-I4X7-nK0pR1
└─sdb2    linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127 xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
sdc
├─sdc1
└─sdc2    linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127 xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
```

やっぱりswapパーティションが消えてしまっています。マウントポイントに swap がありません。恐らく3つのディスクそれぞれにあるUUID「`9090b326-c255...`」が swap のパーティションなんだと思います。（後述しますが、正確には違います。。。）

GRUB2のカーネルコマンドラインパラメーターを確認してみます。

```console
$ sudo grep swap /boot/grub2/grub.cfg
  set kernelopts="root=UUID=a4ef59e1-ff85-4bb0-80c1-eb13bab60840 ro rd.md.uuid=9090b326:c2555cef:09af61c5:51ba0d3d rd.lvm.lv=cl/swap rhgb quiet intel_iommu=on "
```

コロンとハイフンの違い、区切り位置の違いもありますが、UUID値としては同じです。

# ボリュームグループの再構築

Unknownデバイスがあるのは気持ち悪いので、一回ボリュームグループ「`cl`」を削除して、改めて再作成してみます。

参考にしたサイトはこちら。

【参考】[How to fix "pvs shows unknown device" in RHEL/CentOS 7/8 \| GoLinuxCloud](https://www.golinuxcloud.com/fix-pvs-shows-unknown-device-redhat-linux/)

まずは、「`vgreduce cl --removemissing`」コマンドにて、欠落している物理ボリュームを「`cl`」ボリュームグループから消去してみます。

```console
$ sudo vgreduce cl --removemissing
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to [unknown]).
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: Partial LV swap needs to be repaired or removed.
  There are still partial LVs in VG cl.
  To remove them unconditionally use: vgreduce --removemissing --force.
  To remove them unconditionally from mirror LVs use: vgreduce --removemissing --mirrorsonly --force.
  WARNING: Proceeding to remove empty missing PVs.
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
```

ダメでしたね。今度は「`--force`」を付加して強制的に実行してみます。（怖い。。。）

```console
$ sudo vgreduce cl --removemissing --force
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: VG cl is missing PV 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd (last written to [unknown]).
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  WARNING: Removing partial LV cl/swap.
  WARNING: Couldn t find device with uuid 4Qn12e-ytJo-ONeW-Yn0P-Cgmq-JPnq-NQX7Pd.
  Logical volume "swap" successfully removed.
  Wrote out consistent volume group cl.
```

消去できたようです。

```console
$ sudo vgs
  VG #PV #LV #SN Attr   VSize  VFree
  cl   2   0   0 wz--n- 12.41g 12.41g
```

現在、ボリュームグループ「`cl`」は、物理ボリューム（PV）2個のグループになっています。

```console
$ sudo pvs
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  cl lvm2 a-- <6.21g <6.21g
  /dev/sdb1  cl lvm2 a-- <6.21g <6.21g
```

「unknown」だった物理ボリュームが消えています。（大丈夫か。。。）  
上記のサイトの情報を信じて、「`pvcreate`」コマンドにて物理ボリューム「`/dev/sdc1`」を再作成してみます。

```console
$ sudo pvcreate /dev/sdc1
  Physical volume "/dev/sdc1" successfully created.
```

とりあえず、Unknownではなく「`/dev/sdc1`」と表示されました。でも、現時点では「`/dev/sdc1`」はどこのものでもありません。（ボリュームグループ「`cl`」に属していません。）

```console
$ sudo pvs
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  cl lvm2 a-- <6.21g <6.21g
  /dev/sdb1  cl lvm2 a-- <6.21g <6.21g
  /dev/sdc1     lvm2 --- <6.21g <6.21g
```

では、ボリュームグループ「`cl`」を拡張して、物理ボリューム「`/dev/sdc1`」を加えてやります。「`vgextend`」コマンドを使用します。

```console
$ sudo vgextend cl /dev/sdc1
  Volume group "cl" successfully extended
```

コマンド「`pvs`」、「`vgs`」で、それぞれ、物理ボリューム、ボリュームグループの状態を確認します。

```console
$ sudo pvs
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  cl lvm2 a-- <6.21g <6.21g
  /dev/sdb1  cl lvm2 a-- <6.21g <6.21g
  /dev/sdc1  cl lvm2 a-- <6.21g <6.21g
$ sudo vgs
  VG #PV #LV #SN Attr   VSize  VFree
  cl   3   0   0 wz--n- 18.62g 18.62g
```

でも、「`lvs`」は何も表示されない。。。

```console
$ sudo vgs
$
```

再起動したら何とかなる、ということで「`sudo reboot`」としてみるも、Syslogには、「`lvm[199669]: pvscan[199669] PV /dev/sdc1 not used.`」

```
Sep 19 16:55:13 hoge systemd[1]: Condition check resulted in WDC_WD10EZEX-00B primary being skipped.
Sep 19 16:55:13 hoge lvm[199669]:  pvscan[199669] PV /dev/sdc1 not used.
Sep 19 16:55:13 hoge systemd[1]: Starting LVM event activation on device 8:33...
Sep 19 16:55:13 hoge systemd[1]: Started LVM event activation on device 8:33.
```

「`swapon`」コマンドで、swap領域を有効にしようとしても「`/dev/mapper/cl-swap`」が無いよとでます。だって本当に無いのですから！

```console
$ sudo swapon -a
swapon: /dev/mapper/cl-swap を open できません: No such file or directory

$ ls -l /dev/mapper/
合計 0
crw------- 1 root root 10, 236  9月 14 12:14 control
```

# GRUB2パラメータと/etc/fstabをいじってみる（失敗）

【参考】[/etc/default/grubに定義したresumeデバイスの情報がinitrdに反映されない \| SUSE](https://www.suse.com/ja-jp/support/jp/kb/tids/00100065/)

によると、

> 手動でswap情報の設定を行う必要がある場合は、/etc/fstabにも上記GRUB\_CMDLINE\_LINUX\_DEFAULTのエントリで設定したresume(swap)ディスクの情報を定義します。  
> ※/etc/fstabにディスク情報を反映させる場合はUUID値(デフォルト)で指定します。

GRUB2のカーネルコマンドラインパラメーターのUUID値と、「`/etc/fstab`」のUUID値が異なっていると何やらまずいっぽい。

GRUB2の設定ファイル「`/etc/default/grub`」を確認してみます。

```console
$ sudo cat /etc/default/grub
GRUB_ENABLE_BLSCFG=true
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.md.uuid=9090b326:c2555cef:09af61c5:51ba0d3d rd.lvm.lv=cl/swap rhgb quiet intel_iommu=on"
GRUB_DISABLE_RECOVERY="true"
```

「`blkid`」で確認すると、関係するPVのUUID値は「`9090b326-c255-5cef-09af-61c551ba0d3d`」です。コロンとハイフンの違いはあるのでしょうか。（後述しますが、大きな勘違いをしています。。。）

デフォルトの「`/etc/fstab`」のswapのエントリはUUIDではなく「`/dev/mapper/cl-swap`」になっていたので、ここをUUID値に変えてみました。

```
UUID=a4ef59e1-ff85-4bb0-80c1-eb13bab60840 /                       xfs     defaults        0 0
UUID=e8c486bd-2a79-4936-9bfe-e7c75e985915 /boot                   xfs     defaults        0 0
#/dev/mapper/cl-swap     swap                    swap    defaults        0 0
UUID=9090b326-c255-5cef-09af-61c551ba0d3d   swap                    swap    defaults        0 0
```

結果として、だめでした。そして、GRUB2のパラメータを「`rd.md.uuid=9090b326-c255-5cef-09af-61c551ba0d3d rd.lvm.lv=cl/swap`」のコロンからハイフン表記にしても、結果は同じでした。。。

```
Sep 20 09:27:29 hoge systemd[1]: dev-disk-by\x2duuid-9090b326\x2dc255\x2d5cef\x2d09af\x2d61c551ba0d3d.device: Job dev-disk-by\x2duuid-9090b326\x2dc255\x2d5cef\x2d09af\x2d61c551ba0d3d.device/start timed out.
Sep 20 09:27:29 hoge systemd[1]: Timed out waiting for device dev-disk-by\x2duuid-9090b326\x2dc255\x2d5cef\x2d09af\x2d61c551ba0d3d.device.
Sep 20 09:27:29 hoge systemd[1]: Dependency failed for /dev/disk/by-uuid/9090b326-c255-5cef-09af-61c551ba0d3d.
Sep 20 09:27:29 hoge systemd[1]: dev-disk-by\x2duuid-9090b326\x2dc255\x2d5cef\x2d09af\x2d61c551ba0d3d.swap: Job dev-disk-by\x2duuid-9090b326\x2dc255\x2d5cef\x2d09af\x2d61c551ba0d3d.swap/start failed with result 'dependency'.
```

ログをよーく確認してみると、「`Dependency failed for /dev/disk/by-uuid/9090b326-c255-5cef-09af-61c551ba0d3d.`」の記述があります。「`/dev/disk/by-uuid`」を確認してみます。

```console
$ ls -l /dev/disk/by-uuid/
合計 0
lrwxrwxrwx 1 root root 11  9月 20 09:24 a4ef59e1-ff85-4bb0-80c1-eb13bab60840 -> ../../md127
lrwxrwxrwx 1 root root 10  9月 20 09:24 e8c486bd-2a79-4936-9bfe-e7c75e985915 -> ../../sda1
```

確かにここに、UUID「`9090b326-c255-5cef-09af-61c551ba0d3d`」のシンボリックリンクがありません。

```console
$ sudo lvs
$
```

そもそも論理ボリュームが表示されないのがおかしいのです。

# 論理ボリューム再作成

次のサイトを参考に、論理ボリュームを再作成してみます。

【参考】[LVMについて個人的なまとめ (概要とLVMの作成まわりについて) - Qiita](https://qiita.com/409_nu/items/3359b789e3ec692b4f3f)

コマンド「`lvcreate`」にて、ボリュームグループ「`cl`」の上に、論理ボリューム「`swap`」を作成してみます。サイズ「`-L 18.62g`」は、

```console
$ sudo vgs
  VG #PV #LV #SN Attr   VSize  VFree
  cl   3   0   0 wz--n- 18.62g 18.62g
```

の結果の「`VSize`」、つまりボリュームグループ「`cl`」のサイズ全部を指定します。

```console
$ sudo lvcreate -L 18.62g -n swap cl
  Rounding up size to full physical extent 18.62 GiB
WARNING: swap signature detected on /dev/cl/swap at offset 4086. Wipe it? [y/n]: y
  Wiping swap signature on /dev/cl/swap.
  Logical volume "swap" created.
```

おっ、論理ボリューム「`swap`」ができたようです。早速「`lvs`」コマンドで確認します。

```console
$ sudo lvs
  LV   VG Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  swap cl -wi-a----- 18.62g
```

おおーっ、ついでに「`lvdisplay`」コマンドでも確認してみます。

```console
$ sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/cl/swap
  LV Name                swap
  VG Name                cl
  LV UUID                WUxegF-PNOn-5rUZ-O681-KaaH-7usZ-0s5MQq
  LV Write Access        read/write
  LV Creation host, time hoge.example.com, 2023-09-20 10:40:55 +0900
  LV Status              available
  # open                 0
  LV Size                18.62 GiB
  Current LE             4767
  Segments               3
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```

おおーっ、LV Statusが available です。

更に、「`lsblk`」コマンドで確認してみます。

```console
$ sudo lsblk -f
NAME        FSTYPE            LABEL                      UUID                                   MOUNTPOINT
sda
├─sda1      xfs                                          e8c486bd-2a79-4936-9bfe-e7c75e985915   /boot
├─sda2      LVM2_member                                  4FrV6q-DwfQ-cxQs-j20E-fkCf-MWo2-0oVXmp
│ └─cl-swap
└─sda3      linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127   xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
sdb
├─sdb1      LVM2_member                                  sPsefj-bjgH-4w56-4Ukn-2YcP-I4X7-nK0pR1
│ └─cl-swap
└─sdb2      linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127   xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
sdc
├─sdc1      LVM2_member                                  vFnzYo-TZ8J-r6jB-3pKI-aNEH-GnEx-fVarC0
│ └─cl-swap
└─sdc2      linux_raid_member localhost.localdomain:root 9090b326-c255-5cef-09af-61c551ba0d3d
  └─md127   xfs                                          a4ef59e1-ff85-4bb0-80c1-eb13bab60840   /
sr0
```

「`cl-swap`」はできていますが、MOUNTPOINTに「`[SWAP]`」がないです。。。

そして、「`/dev/mapper`」には、「`cl-swap`」があるので、

```console
$ ls -l /dev/mapper
合計 0
lrwxrwxrwx 1 root root       7  9月 20 10:41 cl-swap -> ../dm-0
crw------- 1 root root 10, 236  9月 20 10:40 control
```

「`/etc/fstab`」のswapのエントリを元に（UUID値で無いほうに）戻します。

```
UUID=a4ef59e1-ff85-4bb0-80c1-eb13bab60840 /                       xfs     defaults        0 0
UUID=e8c486bd-2a79-4936-9bfe-e7c75e985915 /boot                   xfs     defaults        0 0
/dev/mapper/cl-swap     swap                    swap    defaults        0 0
# UUID=9090b326-c255-5cef-09af-61c551ba0d3d none                    swap    defaults        0 0
```

「`cl-swap`」の形式は、「ボリュームグループ名-論理ボリューム名」から来ているようです。次のサイトを参考。

【参考】[linux : LVM-ボリュームグループの名前を変更し、システムが起動できることを確認しますか？](https://www.fixes.pub/linux/265347.html)

> LVMを使用する場合、RHEL /CentOS 7 initramfsジェネレーターは、ルートファイルシステムを含むデバイスを指定するroot= オプションを自動生成するようです。また、生成されるエントリはroot= /dev /mapper /VGname-LVname の形式になります。  
> initramfsフェーズ内でアクティブ化するLVを指定する他の1つまたは2つのブートオプションもあります。ルートファイルシステムのLVとプライマリスワップのLV（LVにスワップがある場合）です。これらのオプションの形式はrd.lvm.lv= VGname /LVname です。

では、「`sudo reboot`」してみます。

```console
$ sudo lsblk -i
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda           8:0    0 931.5G  0 disk
|-sda1        8:1    0   953M  0 part  /boot
|-sda2        8:2    0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm
`-sda3        8:3    0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sdb           8:16   0 931.5G  0 disk
|-sdb1        8:17   0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm
`-sdb2        8:18   0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sdc           8:32   0 931.5G  0 disk
|-sdc1        8:33   0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm
`-sdc2        8:34   0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sr0          11:0    1  1024M  0 rom
```

やっぱりだめ。

# swap領域の再作成

rebootしてもダメだったので、手動でswapを有効にしてみます。

```console
$ sudo swapon -a
swapon: /dev/mapper/cl-swap: スワップヘッダの読み込みに失敗しました
```

そうですよね。swap用のパーティションを作ったけど、swap領域を作らないとダメですよね。

【参考】[linux スワップ（swap）領域の作成](https://kazmax.zpp.jp/linux_beginner/mkswap.html)

「`mkswap`」コマンドにて、swap領域を作成します。

```console
$ sudo mkswap /dev/mapper/cl-swap
スワップ空間バージョン 1 を設定します。サイズ = 18.6 GiB (19994243072 バイト)
ラベルはありません, UUID=099475d5-4ecd-4a55-8c55-bba254683632
```

「`swapon`」コマンドで、swap領域を有効にします。

```console
$ sudo swapon /dev/mapper/cl-swap -v
swapon: /dev/mapper/cl-swap: 署名が見つかりました: [ページサイズ=4096, 署名=swap]
swapon: /dev/mapper/cl-swap: ページサイズ=4096, スワップサイズ=19994247168, デバイスサイズ=19994247168
スワップ /dev/mapper/cl-swap を有効化しています
```

イエイッ！

```console
$ sudo lsblk -i
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda           8:0    0 931.5G  0 disk
|-sda1        8:1    0   953M  0 part  /boot
|-sda2        8:2    0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm   [SWAP]
`-sda3        8:3    0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sdb           8:16   0 931.5G  0 disk
|-sdb1        8:17   0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm   [SWAP]
`-sdb2        8:18   0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sdc           8:32   0 931.5G  0 disk
|-sdc1        8:33   0   6.2G  0 part
| `-cl-swap 253:0    0  18.6G  0 lvm   [SWAP]
`-sdc2        8:34   0 924.4G  0 part
  `-md127     9:127  0   1.8T  0 raid5 /
sr0          11:0    1  1024M  0 rom
```

MOUNTPOINT「`SWAP`」が表示されるようになりました。

それでは「`free`」コマンドでswap領域使用状況を確認します。

```console
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           31Gi       461Mi        30Gi       2.0Mi       367Mi        30Gi
Swap:          18Gi          0B        18Gi
```

感動！

以前syslogに出ていた「`/dev/disk/by-uuid/`」の状態ですが、

```console
$ ls -l /dev/disk/by-uuid/
合計 0
lrwxrwxrwx 1 root root 10  9月 20 11:55 099475d5-4ecd-4a55-8c55-bba254683632 -> ../../dm-0
lrwxrwxrwx 1 root root 11  9月 20 11:55 a4ef59e1-ff85-4bb0-80c1-eb13bab60840 -> ../../md127
lrwxrwxrwx 1 root root 10  9月 20 11:55 e8c486bd-2a79-4936-9bfe-e7c75e985915 -> ../../sda1
```

ということで、swapの論理ボリュームのUUID値が「`099475d5-4ecd-4a55-8c55-bba254683632`」になっています。  
これは、「`lsblk`」の結果からも分かります。

```console
$ sudo lsblk -f | grep swap
│ └─cl-swap swap                  099475d5-4ecd-4a55-8c55-bba254683632   [SWAP]
│ └─cl-swap swap                  099475d5-4ecd-4a55-8c55-bba254683632   [SWAP]
│ └─cl-swap swap                  099475d5-4ecd-4a55-8c55-bba254683632   [SWAP]
```

一方、GRUB2のカーネルパラメータのUUID値の方は、

```console
$ sudo grep UUID /boot/grub2/grub.cfg
  set kernelopts="root=UUID=a4ef59e1-ff85-4bb0-80c1-eb13bab60840 ro rd.md.uuid=9090b326:c2555cef:09af61c5:51ba0d3d rd.lvm.lv=cl/swap rhgb quiet intel_iommu=on "
```

以前のままです。よくよく考えてみれば、スワップの指定は、「`rd.lvm.lv=cl/swap`」のほうで、「`rd.md.uuid=9090b326:c2555cef...`」のほうは、ソフトウェアRAIDのmdデバイスにおけるUUIDの指定でした。

ちゃんちゃん。
