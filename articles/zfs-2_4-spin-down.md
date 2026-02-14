---
title: "OpenMediaVault 8でZFSを2.4.0にしたらHDDがスピンダウンしなくなった話"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenMediaVault", "ZFS", "Proxmox"]
published: true
---

# 概要

OpenMediaVault 8でProxmox KernelでOpenZFSを使用しhd-idleを使ってHDDをスピンダウンさせていたところ、ZFSが2.4.0に更新されたタイミングでHDDがスピンダウンしなくなってしまった。

```
$ dpkg -l | egrep 'zfs|6.17'
ii  libzfs7linux:amd64                  2.4.0-pve1                           amd64        OpenZFS filesystem library for Linux - general support
ii  omv-zfs-provider-pve                8.0.0                                amd64        OMV ZFS provider (Proxmox)
ii  openmediavault-zfs                  8.0.4                                amd64        OpenMediaVault plugin for ZFS
ii  proxmox-kernel-6.17                 6.17.9-1                             all          Latest Proxmox Kernel Image
ii  proxmox-kernel-6.17.4-2-pve-signed  6.17.4-2                             amd64        Proxmox Kernel Image (signed)
ii  proxmox-kernel-6.17.9-1-pve-signed  6.17.9-1                             amd64        Proxmox Kernel Image (signed)
ii  zfs-initramfs                       2.4.0-pve1                           all          OpenZFS root filesystem capabilities for Linux - initramfs
ii  zfs-zed                             2.4.0-pve1                           amd64        OpenZFS Event Daemon
ii  zfsutils-linux                      2.4.0-pve1                           amd64        command-line tools to manage OpenZFS filesystems
```

恐らく以下のfeatureをオンにしたタイミングから発生。

```
2026-02-08.08:49:08 [txg:12514994] set feature@block_cloning_endian=enabled
2026-02-08.08:49:08 [txg:12514995] set feature@physical_rewrite=enabled
```

解決に随分手間取ったので記事にした。

# 起きていたこと

auditdを入れvfs経由のアクセスや/dev/sdxへのアクセスを監視してもそれらしいアクセスはないのにzpool iostatで見ていると10分毎に約1MBのアクセスが発生。

```
zpool iostat -v <pool名> 5
```

# 対策

spa_note_txg_timeというパラメータが600となっていたが、これが悪さをしていた模様。

```
$ cat /sys/module/zfs/parameters/spa_note_txg_time
600
```

以下のように1年に設定したところ10分毎のアクセスはなくなった。

```
$ cat /etc/modprobe.d/zfs.conf
options zfs spa_note_txg_time=31557600
```

`sudo update-initramfs -u`等を忘れずに。

# 終わりに

技術的な背景はよく分からず記事にしている。

Geminiに相談していたところ`grep -r "600" /sys/module/zfs/parameters/`で10分のパラメータ無いんだっけ？みたいな話になり出てきたパラメータをgoogleで検索したら以下のredditがヒット。

私と同じ状況のようなのでスレッドの解決策をそのままやってみたら治ったという話。

https://www.reddit.com/r/zfs/comments/1q75x2u/disks_no_longer_sleeping_since_zfs_240/

