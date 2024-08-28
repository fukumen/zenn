---
title: "【Ubuntu 24.04】セキュアブートONでmodprobe nvidiaに失敗するようになった話"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ubuntu"]
published: true
---
# 概要

TUF GAMING Z790-PLUS WIFI D4のBIOSをIntel 第13/14世代Coreの対策マイクロコード「0x129」に対応した1663に更新したところmodprobe nvidiaに失敗するようになってしまったときの復帰手順である。

具体的には以下のようなメッセージが表示されカーネルモジュールのロードに失敗する。
```
modprobe: ERROR: could not insert 'nvidia': Key was rejected by service
```

# 環境

Ubuntu 24.04の標準カーネルとNVIDIAのドライバを使用した環境。

* Ubuntu 24.04
* linux-headers-6.8.0-41-generic
* nvidia-driver-535

# 手順

一旦、UEFIでセキュアブートOFFに設定しmodprobe nvidiaに成功することを確認し、再度セキュアブートONに設定しUbuntuのブートは成功するがmodprobe nvidiaには失敗する状態から開始した。

まずはセキュアブートに成功しているかを確認する。

```
$ sudo mokutil --sb-state
SecureBoot enabled
```

NVIDIAのドライバを一旦削除する。

```
$ sudo apt-get purge nvidia*
```

MOKの証明書をインポートする。

```
$ sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
input password: パスワードを入力
input password again: パスワードを入力
```

その後再起動するとPerform MOK managementが立ち上がるので10秒以内に何かキーを押してやり、Enroll MOKを選択しパスワードを入力する。具体的には https://nanbu.marune205.net/2021/12/ubuntu-secure-boot.html?m=1 を参考に実施。

Perform MOK managementから再起動後にNVIDIAのドライバをインストールする。

```
$ sudo ubuntu-drivers autoinstall
```

この状態でmodprobe nvidia出来るようになった。

# 参考

* https://qiita.com/abehiro/items/81af20369099a303b855
