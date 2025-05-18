---
title: "ソフトウェアRAID構成におけるHDD交換"
date: 2023-08-14
categories: 
  - "linux"
tags:
  - "linux"
  - "raid"
---

# はじめに

CentOS 7.9 の研究室用サーバですが、メンテ作業のため、システムリブートさせたところ、S.M.A.R.T でエラーが出てしまいました。  
使用開始して、もう5～6年は経っていると思うのですが、もうディスクエラーかよ、っていう感じです。  
このサーバのディスクですが、SATA 1TBのHDDを3台で、RAID5を構成しています。  
ハードウェアRAIDではなく、LinuxによるソフトウェアRAIDです。

まずは、ソフトウェアRAIDの構成の確認してみます。

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdc2[3] sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/8 pages [0KB], 65536KB chunk

unused devices: <none>
```

`/dev/sda3`, `/dev/sdb2`, `/dev/sdc2` で RAID5 を構成、デバイス名は `/dev/md127`  
ディスク全て3台とも、`[UUU]` となっているので、特に縮退している様子もないですね。

syslogで確認すると、確かにRAID関係のエラーが連発しているようです。

```
Mar 19 07:17:51 buzz kernel: md/raid:md127: read error corrected (8 sectors at 1649026336 on sdc2)
Mar 19 07:18:16 buzz kernel: md/raid:md127: read error NOT corrected!! (sector 1649026584 on sdc2).
Mar 19 07:18:39 buzz kernel: md/raid:md127: read error corrected (8 sectors at 1649026832 on sdc2)
```

うーん、`/dev/sdc2` にてセクタの読み込みに失敗しているようです。

![](/assets/images/replacement-hdd1024.jpg?w=1024)

<!--more-->

# 状況確認

## 個別のディスクの状況確認（smartctlコマンド）

`smartctl`コマンドでスキャンしてみます。

```console
$ sudo smartctl --scan
/dev/sda -d scsi # /dev/sda, SCSI device
/dev/sdb -d scsi # /dev/sdb, SCSI device
/dev/sdc -d scsi # /dev/sdc, SCSI device
```

問題のあった `/dev/sdc` のドライブ詳細情報を、smartctlコマンドにて確認してみます。

```console
$ sudo smartctl /dev/sdc -i
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Family:     Hitachi Deskstar 7K1000.C
Device Model:     Hitachi HDS721010CLA332
Serial Number:    JP2921HQ1PL9DA
LU WWN Device Id: 5 000cca 35dd7e801
Firmware Version: JP4OA39C
User Capacity:    1,000,204,886,016 bytes [1.00 TB]
Sector Size:      512 bytes logical/physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        In smartctl database [for details use: -P show]
ATA Version is:   ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 2.6, 3.0 Gb/s
Local Time is:    Fri Mar 31 10:06:49 2023 JST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```

S/Nが表示されているので、交換するディスクを間違えないようこの番号（S/N）を控えておきます。

さらに、問題のあった `/dev/sdc` の詳細情報を確認します。

```console
$ sudo smartctl /dev/sdc -A
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   085   085   016    Pre-fail  Always       - 11468880
  2 Throughput_Performance  0x0005   134   134   054    Pre-fail  Offline      - 102
  3 Spin_Up_Time            0x0007   125   125   024    Pre-fail  Always       - 303 (Average 305)
  4 Start_Stop_Count        0x0012   100   100   000    Old_age   Always       - 57
  5 Reallocated_Sector_Ct   0x0033   001   001   005    Pre-fail  Always   FAILING_NOW 1736
  7 Seek_Error_Rate         0x000b   100   100   067    Pre-fail  Always       - 0
  8 Seek_Time_Performance   0x0005   132   132   020    Pre-fail  Offline      - 34
  9 Power_On_Hours          0x0012   096   096   000    Old_age   Always       - 34659
 10 Spin_Retry_Count        0x0013   100   100   060    Pre-fail  Always       - 0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       - 57
192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       - 191
193 Load_Cycle_Count        0x0012   100   100   000    Old_age   Always       - 191
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       - 33 (Min/Max 14/43)
196 Reallocated_Event_Count 0x0032   010   010   000    Old_age   Always       - 1803
197 Current_Pending_Sector  0x0022   095   095   000    Old_age   Always       - 282
198 Offline_Uncorrectable   0x0008   100   100   000    Old_age   Offline      - 0
199 UDMA_CRC_Error_Count    0x000a   200   200   000    Old_age   Always       - 0
```

「`Reallocated_Sector_Ct`」の項目が「`FAILING_NOW`」となっています。

```
  5 Reallocated_Sector_Ct   0x0033   001   001   005    Pre-fail  Always   FAILING_NOW 1736
```

「`Reallocated_Sector_Ct`」は、Wikipediaによると、「代替処置（データを特別に予約した予備エリアに移動する）を施された不良セクタの数」のことらしい。

【参考】[Self-Monitoring, Analysis and Reporting Technology - Wikipedia](https://ja.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology)

自己診断(SelfTest)を行ったときにかかる予測時間を確認してみます。

```console
$ sudo smartctl -c /dev/sdc
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
General SMART Values:
Offline data collection status:  (0x82) Offline data collection activity
                                        was completed without error.
                                        Auto Offline Data Collection: Enabled.
Self-test execution status:      (   0) The previous self-test routine completed
                                        without error or no self-test has ever
                                        been run.
Total time to complete Offline
data collection:                (10635) seconds.
Offline data collection
capabilities:                    (0x5b) SMART execute Offline immediate.
                                        Auto Offline data collection on/off support.
                                        Suspend Offline collection upon new
                                        command.
                                        Offline surface scan supported.
                                        Self-test supported.
                                        No Conveyance Self-test supported.
                                        Selective Self-test supported.
SMART capabilities:            (0x0003) Saves SMART data before entering
                                        power-saving mode.
                                        Supports SMART auto save timer.
Error logging capability:        (0x01) Error logging supported.
                                        General Purpose Logging supported.
Short self-test routine
recommended polling time:        (   1) minutes.
Extended self-test routine
recommended polling time:        ( 177) minutes.
SCT capabilities:              (0x003d) SCT Status supported.
                                        SCT Error Recovery Control supported.
                                        SCT Feature Control supported.
                                        SCT Data Table supported.
```

「`Short`」だと1分、「`Extended`」だと177分、それぞれかかるようです。

自己診断の履歴を確認します。

```console
$ sudo smartctl -l selftest /dev/sdc
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART Self-test log structure revision number 1
No self-tests have been logged.  [To run self-tests, use: smartctl -t]
```

これまで自己診断をやった実績なしです。  
では、`Short`での自己診断を行ってみます。

```console
$ sudo smartctl -t short /dev/sdc
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Short self-test routine immediately in off-line mode".
Drive command "Execute SMART Short self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 1 minutes for test to complete.
Test will complete after Fri Mar 31 10:18:16 2023

Use smartctl -X to abort test.
```

すぐにプロンプトに戻りました。バックグラウンドで処理されているようです。  
中止したい場合は、「`smartctl -X`」をすればよいらしいです。`short`なので1分で終了します。

再度、自己診断の履歴を確認してみます。

```console
$ sudo smartctl -l selftest /dev/sdc
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.80.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     34659         -
```

「`Completed without error`」ということで、`Short`でのselftestはエラーはなかったということになりますね。

## RAIDドライブの状況確認（mdadmコマンド）

`mdadm`コマンドによりRAIDドライブ `/dev/md127` の様子を確認してみます。

```console
$ sudo mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Mon Jun 28 10:11:18 2021
        Raid Level : raid5
        Array Size : 1938284544 (1848.49 GiB 1984.80 GB)
     Used Dev Size : 969142272 (924.25 GiB 992.40 GB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Fri Mar 31 09:16:39 2023
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : bitmap

              Name : localhost.localdomain:root
              UUID : 9090b326:c2555cef:09af61c5:51ba0d3d
            Events : 13282

    Number   Major   Minor   RaidDevice State
       0       8        3        0      active sync   /dev/sda3
       1       8       18        1      active sync   /dev/sdb2
       3       8       34        2      active sync   /dev/sdc2
```

さらに、問題のあった `/dev/sdc2` の詳細を確認してみます。

```console
$ sudo mdadm --examine /dev/sdc2
/dev/sdc2:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : 9090b326:c2555cef:09af61c5:51ba0d3d
           Name : localhost.localdomain:root
  Creation Time : Mon Jun 28 10:11:18 2021
     Raid Level : raid5
   Raid Devices : 3

 Avail Dev Size : 1938284544 sectors (924.25 GiB 992.40 GB)
     Array Size : 1938284544 KiB (1848.49 GiB 1984.80 GB)
    Data Offset : 262144 sectors
   Super Offset : 8 sectors
   Unused Space : before=262056 sectors, after=0 sectors
          State : clean
    Device UUID : bfdefb90:056d88ba:1b7f2ade:1306554a

Internal Bitmap : 8 sectors from superblock
    Update Time : Fri Mar 31 09:26:58 2023
  Bad Block Log : 512 entries available at offset 72 sectors
       Checksum : 63596596 - correct
         Events : 13282

         Layout : left-symmetric
     Chunk Size : 512K

   Device Role : Active device 2
   Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
```

「`Array State`」を見る限り、他のドライブ同様 `AAA` なので、問題はなさそう。

でも、確かにsyslogではエラーが出ているので、予防交換することにします。

## 交換対象ディスクのパーティション確認

新しいディスクに交換する際、そのディスクをフォーマットする際に情報として必要なので、現行のディスクのフォーマット情報を`parted`コマンドにて取得し、セクタの開始や終了位置をメモ書きしておきます。

```console
$ sudo parted /dev/sdc -s unit s -s p
モデル: ATA Hitachi HDS72101 (scsi)
ディスク /dev/sdc: 1953525168s
セクタサイズ (論理/物理): 512B/512B
パーティションテーブル: msdos
ディスクフラグ:

番号  開始       終了         サイズ       タイプ   ファイルシステム  フラグ
 1    2048s      13025279s    13023232s    primary                    lvm
 2    13025280s  1951571967s  1938546688s  primary                    raid
```

ちなみに、`fdisk`コマンドで確認すると次のとおり。

```console
$ sudo fdisk -l /dev/sdc

Disk /dev/sdc: 1000.2 GB, 1000204886016 bytes, 1953525168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O サイズ (最小 / 推奨): 512 バイト / 512 バイト
Disk label type: dos
ディスク識別子: 0x000dc4be

デバイス ブート      始点        終点     ブロック   Id  システム
/dev/sdc1            2048    13025279     6511616   8e  Linux LVM
/dev/sdc2        13025280  1951571967   969273344   fd  Linux raid autodetect
```

# 交換作業

## 現行のディスクを取り外す前に

mdadmコマンドにて、`/dev/sdc2` に、failフラグを付けます。

```console
$ sudo mdadm /dev/md127 --fail /dev/sdc2
[   88.630643] md/raid:md127: Disk failure on sdc2, disabling device.
[   88.630643] md/raid:md127: Operation countinuing on 2 devices.
mdadm: set /dev/sdc2 faulty in /dev/md127
```

mdstatを確認してみます。

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdc2[3](F) sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      bitmap: 7/8 pages [28KB], 65536KB chunk

unused devices: <none>
```

`sdc2`に、フラグ「`(F)`」が付いて、`[UU_]` の縮退運転になっているのが確認できます。  
さらに、mdadmコマンドにて、raidグループから、failされたディスク（`/dev/hdc2`）を取り除きます。

```console
$ sudo mdadm /dev/md127 --remove /dev/sdc2
mdadm: hot removed /dev/sdc2 from /dev/md127
```

あっさりとした結果出力ですね。

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      bitmap: 7/8 pages [28KB], 65536KB chunk

unused devices: <none>
```

`sdc2`の文字が表示されなくなりました。これでソフトウェア的にはディスクは取り外されたことになります。

## 実際のディスク交換

サーバの電源をOFF（shutdown）し、問題のあるディスクを交換します。  
交換する新しいディスクのモデル名とシリアル番号を、忘れないようメモしておきます。

交換した後、サーバの電源をOnします。

smartctlコマンドにて新しいディスクの確認してみます。

```console
$ sudo smartctl /dev/sdc -i
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-3.10.0-1160.88.1.el7.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Family:     Western Digital Blue
Device Model:     WDC WD10EZEX-00BN5A0
Serial Number:    WD-WCC3F3DR29FD
LU WWN Device Id: 5 0014ee 2b73e1095
Firmware Version: 01.01A01
User Capacity:    1,000,204,886,016 bytes [1.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Device is:        In smartctl database [for details use: -P show]
ATA Version is:   ACS-2, ACS-3 T13/2161-D revision 3b
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Sun Aug 13 11:47:52 2023 JST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```

今回入れ替えた新しいディスクは、Western Digitalです。モデル名、シリアル値が一致しているか確認します。

# 新規ディスクのフォーマット

`parted`コマンドにて、ディスクをフォーマット（パーティション分け）をしていきます。

```console
$ sudo parted /dev/sdc
GNU Parted 3.1
/dev/sdc を使用
GNU Parted へようこそ！ コマンド一覧を見るには 'help' と入力してください。
(parted) 
```

まずはコマンド「`p`」にて、状態を表示してみます。

```
(parted) p
エラー: /dev/sdc: ディスクラベルが認識できません。
モデル: ATA WDC WD10EZEX-00B (scsi)
ディスク /dev/sdc: 1000GB
セクタサイズ (論理/物理): 512B/4096B
パーティションテーブル: unknown
ディスクフラグ:
(parted)
```

ディスクラベルがないので、「`gpt`」というラベルを付けます。

```
(parted) mklabel gpt
(parted) p
モデル: ATA WDC WD10EZEX-00B (scsi)
ディスク /dev/sdc: 1000GB
セクタサイズ (論理/物理): 512B/4096B
パーティションテーブル: gpt
ディスクフラグ:

番号  開始  終了  サイズ  ファイルシステム  名前  フラグ

(parted)
```

パーティションテーブルのラベルが「`unknown`」から「`gpt`」に変わりました。

それでは、前にメモした交換前のディスクのパーティションと同じになるよう、コマンド「`mkpart`」にて、パーティション分けしていきます。

まずは、1つ目のパーティション

```
(parted) mkpart
パーティションの名前?  []? primary
ファイルシステムの種類?  [ext2]?
開始? 2048s
終了? 13025279s
(parted) 
```

次に、2つ目のパーティション

```
(parted) mkpart
パーティションの名前?  []? primary
ファイルシステムの種類?  [ext2]?
開始? 13025280s
終了? 1951571967s
(parted) 
```

1つ目に「`lvm`」、2つ目に「`raid`」とそれぞれフラグを付けます。

```
(parted) set 1 lvm on
(parted) set 2 raid on
(parted) 
```

作成されたパーティションを確認します。

```
(parted) p
モデル: ATA WDC WD10EZEX-00B (scsi)
ディスク /dev/sdc: 1000GB
セクタサイズ (論理/物理): 512B/4096B
パーティションテーブル: gpt
ディスクフラグ:

番号  開始    終了    サイズ  ファイルシステム  名前     フラグ
 1    1049kB  6669MB  6668MB                    primary  lvm
 2    6669MB  999GB   993GB                     primary  raid
(parted) 
```

交換前のディスクと同じになりましたので、コマンド「`q`」

```
(parted) q
通知: 必要であれば /etc/fstab を更新するのを忘れないようにしてください。
$
```

fdiskでも確認してみます。

```console
$ sudo fdisk /dev/sdc
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

コマンド (m でヘルプ): p

Disk /dev/sdc: 1000.2 GB, 1000204886016 bytes, 1953525168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O サイズ (最小 / 推奨): 4096 バイト / 4096 バイト
Disk label type: gpt
Disk identifier: 16DD4C98-65C8-466A-B4E5-2C73406C8745

#         Start          End    Size  Type            Name
 1         2048     13025279    6.2G  Linux LVM       primary
 2     13025280   1951571967  924.4G  Linux RAID      primary

コマンド (m でヘルプ): q
```

※fdiskは2TB未満まで

`lsblk`コマンドでも確認してみます。

```console
$ sudo lsblk -i
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                   8:0    0 931.5G  0 disk
|-sda1                8:1    0   953M  0 part  /boot
|-sda2                8:2    0   6.2G  0 part
| `-cl-swap         253:1    0  18.6G  0 lvm   [SWAP]
`-sda3                8:3    0 924.4G  0 part
  `-md127             9:127  0   1.8T  0 raid5 /
sdb                   8:16   0 931.5G  0 disk
|-sdb1                8:17   0   6.2G  0 part
| `-cl-swap         253:1    0  18.6G  0 lvm   [SWAP]
`-sdb2                8:18   0 924.4G  0 part
  `-md127             9:127  0   1.8T  0 raid5 /
sdc                   8:32   0 931.5G  0 disk
|-sdc1                8:33   0   6.2G  0 part
`-sdc2                8:34   0 924.4G  0 part
:
```

新規ディスクのraidパーティション（`/dev/sdc2`）をRAIDグループ（`/dev/md127`）に編入します。

```console
$ sudo mdadm --manage /dev/md127 --add /dev/sdc2
mdadm: added /dev/sdc2
```

※addするのは「`/dev/sdc2`」であって、「`/dev/sdc`」ではないことに注意！

mdstatを確認します。

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdc[3] sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [>....................]  recovery =  0.1% (1378176/969142272) finish=163.8min speed=98441K/sec
      bitmap: 7/8 pages [28KB], 65536KB chunk

unused devices: <none>
```

既にリビルドが始まっているようです。

`lsblk`コマンドでも確認してみます。

```console
$ sudo lsblk -i
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                   8:0    0 931.5G  0 disk
|-sda1                8:1    0   953M  0 part  /boot
|-sda2                8:2    0   6.2G  0 part
| `-cl-swap         253:1    0  18.6G  0 lvm   [SWAP]
`-sda3                8:3    0 924.4G  0 part
  `-md127             9:127  0   1.8T  0 raid5 /
sdb                   8:16   0 931.5G  0 disk
|-sdb1                8:17   0   6.2G  0 part
| `-cl-swap         253:1    0  18.6G  0 lvm   [SWAP]
`-sdb2                8:18   0 924.4G  0 part
  `-md127             9:127  0   1.8T  0 raid5 /
sdc                   8:32   0 931.5G  0 disk
|-sdc1                8:33   0   6.2G  0 part
`-sdc2                8:34   0 924.4G  0 part
  `-md127             9:127  0   1.8T  0 raid5 /
:
```

良い感じです。

構築速度を上げる方法として、`/proc/sys/dev/raid/speed_limit_min`の値を、1000から100000くらいに上げる、つまり、

```console
# echo 100000 > /proc/sys/dev/raid/speed_limit_min
```

とすると、動的に変わるらしいです。  

【参考】[mdadmで運用しているraid5のディスク交換 - Qiita](https://qiita.com/t_mozomozo/items/557f29815e7bf8add75b)

約10分後、確認してみるとこんな感じ。

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdc[3] sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [=>...................]  recovery =  6.9% (67368320/969142272) finish=143.6min speed=104604K/sec
      bitmap: 7/8 pages [28KB], 65536KB chunk

unused devices: <none>
```

構築速度が、確かに、98441K/sec から、104604K/sec に上がっています。ちょっと速くなったかな。

～リビルド中…～

```console
$ sudo cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md127 : active raid5 sdc2[3] sdb2[1] sda3[0]
      1938284544 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/8 pages [0KB], 65536KB chunk
```

無事にリビルドされました。

syslogも確認

```console
$ sudo grep md127 /var/log/messages
:
Aug 13 13:38:47 bamboo kernel: md: recovery of RAID array md127
Aug 13 16:29:33 bamboo kernel: md: md127: recovery done.
:
```

リビルドにかかった時間は約170分、ほぼ予定通り。`read error corrected` のエラーも出なくなりました。

【2023/09/20 追記】

実は、問題があって、上の「`lsblk`」コマンドの結果からもわかるように、「`/dev/sdc1`」がボリュームグループになっておらず、SWAPパーティションとして認識されていません。なので、SWAP領域確保に失敗します。次の投稿にその後の解決策を記録しています。

[HDD交換にて消失したswap領域の復旧 \| tamo's archives](recovery-of-lost-swap-partition.html)

