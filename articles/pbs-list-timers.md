---
title: "ayufan/proxmox-backup-serverで証明書の自動更新する方法"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["proxmox"]
published: true
---
# 概要

Proxmox Backup Server(以降pbs)のunofficialなDockerイメージ https://github.com/ayufan/pve-backup-server-dockerfiles では証明書の自動更新が動作してくれない。

pbsの証明書の自動更新はsystemdのタイマーで/usr/lib/x86_64-linux-gnu/proxmox-backup/proxmox-daily-updateを実行することで実現されているがsystemdを使うように構成されていないために起きているようです。

![systemdのタイマー](/images/pbs-list-timers.png)
*systemdのタイマー*

# 対応方法

今回はホスト側のcronでコンテナ内のproxmox-daily-updateを一日一回起動することで対応する。
（cronをインストールしておくこと）

まず、cronで起動するシェルスクリプトを用意する。今回の環境では「/home/ubuntu/docker-pbs」にdocker-compose.ymlを格納しているディレクトリであり、そのディレクトリにupdate.shを用意しておく。

```:/home/ubuntu/docker-pbs/update.sh
#!/bin/sh

cd /home/ubuntu/docker-pbs

docker-compose exec -T pbs /usr/lib/x86_64-linux-gnu/proxmox-backup/proxmox-daily-update > /dev/null 2>&1
```

シェルスクリプトを用意したら適当な所有者と権限にしておくこと（ここではユーザubuntu、権限0755とした）。

次にcron.dにファイルを用意する。今回の環境では「/etc/cron.d/pbs_update」に作成した。

```:/etc/cron.d/pbs_update
49 17 * * *	root	/home/ubuntu/docker-pbs/update.sh
```

上記だと毎日17時49分に実行される。

狙った時刻にテストすることが容易なため上記のようにしたが、本来はcron.dailyにシェルスクリプトを入れた方が良さそう。

# 制約事項

proxmox-daily-updateは証明書の更新以外にapt updateが実行される。apt updateの参照先が正規リポジトリになっているようなのでサブスクリプションがないとapt updateに失敗しタスクログに警告が残る。