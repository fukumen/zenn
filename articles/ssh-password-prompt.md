---
title: "sshでパスワード認証のパスワードがKDEウォレットに保存できなかったのをなんとかする話"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ssh"]
published: true
---
# 困った話

NetgearのMS510TXMはsshで接続出来るが、公開鍵認証に対応しておらずパスワード認証しかできない。
sshでnetgearに接続しようとするとksshaskpassにパスワードを求められパスワードを保存するチェックボックスが表示された。
一見このチェックボックスをチェックしておけばKDEウォレットにパスワードが保存されて次回接続時にはパスワード無しで接続できるかと思ったが、再度パスワードを要求されてしまった。

# 原因と対策

ssh時のコンソールの表示をよく見るとksshaskpassから以下のような出力がされていた。

```
$ ssh admin@netgear
ksshaskpass: Unable to parse phrase "(admin@netgear) Password:"
```

ksshaskpassの実装を調べるとパスワード認証のプロンプトの文字列から識別子を取り出す処理があるが、Netgearのプロンプトの形式に対応していないことの警告だった（ダブルクォーテーションの間が送られてきたプロンプトの文字列）。

プロンプトの文字列をコマンドの引数から受け取っているだけなので、この文字列を変換するラッパーのスクリプトを用意し特殊なプロンプト形式を標準的な形式に変換してksshaskpassに渡してやる。
これによりKDEウォレットにパスワード保存ができるようにする。

# 環境

* Ubuntu 25.04
* KDE Plasma 6.3.4

# 対策方法

以下のようなシェルスクリプトを用意して実行属性をつけておく。

```:${HOME}/bin/ksshaskpass_netgear
#! /bin/sh

# ex) "(admin@netgear) Password:"
HOST_USER=$(echo "$1" | sed -E 's/^\((.*)\) Password:$/\1/')

if [ ! -z "$HOST_USER" ]; then
        /usr/bin/ksshaskpass "(${HOST_USER})'s password: "
else
        /usr/bin/ksshaskpass $1
fi
```

Netgearに接続するときはSSH_ASKPASS環境変数に用意したシェルスクリプト指定してsshを起動してやる。

```
$ SSH_ASKPASS=${HOME}/bin/ksshaskpass_netgear ssh admin@netgear
```
KDEなのでksshaskpassを使っているが他のssh-askpassでも同じようにできるはずだが、当然変換先のプロンプトの形式はそのソフトが対応している形にしないと駄目。
（例はopenssh標準的なプロンプトのようなのでよほどそのままでいけると思いますが）

長くて毎回打ちたくないのでaliasを用意してのがおすすめ。