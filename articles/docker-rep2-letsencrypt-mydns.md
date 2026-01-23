---
title: "Dockerでrep2+Let's Encrypt+MyDNS" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["rep2", "letsencrypt", "mydns", "docker"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

:::message
dockerコンテナをgithubに公開(2026/1/23時点)
<https://github.com/fukumen/docker-rep2>を参照
:::

# 概要

既存の以下の物や情報をベースにDockerでrep2+Let's Encrypt+MyDNSな環境作成をしたまとめ(2023/12/24時点)。

|場所|内容|
|-|-|
|<https://github.com/mikoim/p2-php>|rep2 expack 全部入り for PHP 8.x by （´・ω・） ｽ|
|<https://github.com/pen/docker-rep2/tree/php8>|mikoimさんのphp8対応版をフライングでコンテナ化したものです。|
|<https://qiita.com/SogoK/items/8bcf92356dcb39621a56>|【ポート開放なし】MyDNS.JPのワイルドカード証明書を簡単に取得する【Docker】|
|<https://egg.5ch.net/test/read.cgi/software/1659739569/63-93>|postとtgrepの文字コード問題の修正|

# 構築した環境

* OS: Debian 12
* Docker: OS付属ではなくdocker.comのを使用

# Let's Encrypt+MyDNS

HTTP-01チャレンジを使うのであればOS標準のcertbotでよいが、DNS-01チャレンジを使いたかったので[ポート開放なし】MyDNS.JPのワイルドカード証明書を簡単に取得する【Docker】](https://qiita.com/SogoK/items/8bcf92356dcb39621a56)を参考にした。

このDockerコンテナでホストの/etc/letsencryptを作成・更新してやり、後述のdocker-rep2のDockerコンテナで/etc/letsencryptに用意された証明書を参照するようにする。

なお、提供されているソースコードを使うとAlpine Linuxのバージョンが上がってPHPのバージョンが上がっていて上手く行かなかった。

## 修正が必要な箇所

1. Dockerfileのapk add行のphp7をphp81に置き換える
1. DirectEdit-master/txtdelete.phpとtxtregist.phpの/usr/bin/phpを/usr/bin/php81に置き換える

あと、ホスト側のcronでrenew_cert.shを動かす場合は-itオプションを消す必要があった。

# rep2本体

mikoimさんのphp8対応版+5chのrep2スレ(Part69)の情報ではいくつか文字化けが残ったので(私がスレを追いきれていないだけかもしれないが)、場当たり的に以下を追加で修正した。

* 板検索
* 板内のスレ検索
* スレ内の文字列フィルタ
* （なにか一個忘れている気がする）

<https://github.com/fukumen/p2-php/tree/php8-merge-mbstring>

# docker-rep2

penさんのdocker-rep2からの変更は以下の二箇所のみ。

* 前述のrep2本体を文字化け修正版を参照するように変更
* h2o.confにSSLの設定を追加

h2o.confにLet's Encryptの証明書のパスが含まれているので以下をクローンして修正してdocker buildが必要。

<https://github.com/pen/docker-rep2/tree/php8>

## 手順

まずrootfs/etc/h2o.confの以下の行の「your-domain」の部分を実際のドメインに置き換える。

    certificate-file: /etc/letsencrypt/live/your-domain/fullchain.pem
    key-file:  /etc/letsencrypt/live/your-domain/privkey.pem

次に以下のようにしてdocker buildしてやる。

    $ docker build . -t rep2

## docker-compose.yml

以下、サンプル

```yaml
version: '2'
services:
  rep2php8
    image: rep2:latest
    volumes:
      - ./rep2-data:/ext
      - /etc/letsencrypt:/etc/letsencrypt
    ports:
      - "10088:443"
    restart: unless-stopped
    environment:
      NCPX_USER_AGENT: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:99.0) Gecko/20100101 Firefox/99.0
      NCPX_ENABLE_SSL_CONNECTION: 1
      NCPX_TIMEOUT: 120
```


