---
title: "【コミュファ光】ルーター広告(RA)のRDNSSを偽装して内部DNSを通知する"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["コミュファ光", "ipv6"]
published: true
---
# 概要

HGWのルーター広告(以降RA)のRDNSSを偽装して内部DNSを通知する方法のまとめである。

RAを偽装してリレーするPythonスクリプトとICMPv6のACLが設定出来るようなL2スイッチングハブ(以降HUB)により実現する。Pythonスクリプトを動かすホストはHGWとHUBの両方を接続するため、Ethernetが2ch必要となる。

また、LANをHUBの下にぶら下げる必要がありHGWのWIFIはオフにするしかないため、WIFIが必要であればHUBの下流にWIFI APが必要になる。

![全体図](/images/ra-rdnss-spoofing-overall.png)

# 何に困ってRDNSSを偽装しようとしたか

私が契約しているコミュファ光のHGWではRDNSSやDHCPv6が切ることが出来ず、IPv6を有効にするとHGWのDNSがIPv6ホストに通知されてしまいHGWのDNSフォワードを使うことになり、結果的に外部DNSのサーバーを参照することが強制される。うちの環境ではIPv4では内部DNSのサーバーがいて内部サーバーの名前解決をしていたが、同様のことがIPv6では出来ず困ってしまいRDNSSの偽装することを考えた。

コミュファから提供された我が家のHGWはHP620という機種で、このHGWのDNSフォワードの動きをもう少し詳しく見てみると、HGWにIPv4でDNSの問い合わせをするとHGWに設定したDNSへフォワードされるが(IPv4のDNSはHGWに設定出来る)、IPv6だとIPv4のDNSに問い合わせはされず外部のDNSへ問い合わせされてしまうようだ。

IPv4で出来ることがIPv6が出来ないというのも変な話なので、HGWのIPv6のDNSアドレスの通知を止めたり変更することが出来ないがこれは不具合ではないのかコミュファに問い合わせてみたが、他のHGWでも同様でありどうしようもないと回答あり。コミュファ光には法人向けにビジネスコミュファというブランドもあるが、WEBの情報を見る限りは期待は出来無さそう(ビジネスって何だろう)。

## PPPoEブリッジ

コミュファ光はIPv4とIPv6のどちらともPPPoE接続であり、HGWでPPPoEブリッジを有効とすることも出来るため、最初に考えたのはPPPoEブリッジにして好きなルーターを使えばよいのではないかという方法だ。しかし、コミュファではPPPoEの接続IDとパスワードがIPv4しか公開されておれず、IPv6接続をしたいのであればHGWをそのまま使うしかない。

この話には抜け道もあるようでHGWがNEC製であれば以下の記事の方法でIPv6の接続IDとパスワードが手に入るようだ。

https://qiita.com/nabeyaki/items/25263f35166d9255c4ea

この方法が使えればRDNSSやDHCPv6をどうするかは自分で用意したルーター次第となる。だが、コミュファから提供された我が家のHGWはHP620という韓国製でこの方法は使えなかった。

## 他の手段

HGWを使うしか無いとなるとRAリレーしつつRDNSSオプションのアドレスを上書き出来るようなルーター(というかブリッジ？)を用意すれば解決しそうだが、コンシューマ向けのルーターでは難しそうである。調べたり少し試した限りはOpenWrtのodhcpdであれば出来そうではあった。

もしくはRAの上書きは出来なくてもPBRが設定出来るようなL3スイッチングハブが用意出来ればHGWに向けられた通信を内部DNSに向けることも出来そうである。

困ったことにコミュファ光を10Gbpsで契約してしまっているので10Gbps関係のハードを追加するのは消費電力も気になるし、悩んだ挙句今回はこの記事のような方法を取ることにした。

# どのようにしてRDNSSを偽装するか

まず、定期送信されたRAをHUBでは拒否するが、RAリレーホストで受信してHUBの下流側へ偽装したRAをリレーする。このときRAリレーホストでは以降のルータ要請(以降RS)受信時に使うため、RAを保存しておく。

![RA](/images/ra-rdnss-spoofing-ra.png)

なお、私が使用しているHUBではACLをインバウンドしか設定できないため、偽装したRAがHGW側にも流れてしまうという問題があるが、うちのHGWの場合は実害無さそうである。

次にRSを送信したホストに対してRAリレーホストで保存していてRAを元にRAの応答を返す。現在のPythonスクリプトの実装はRAリレーホストのHUB側のEthernetのI/F(RAを送信する方)でRSを受信している。

![RS](/images/ra-rdnss-spoofing-rs.png)

RAと同じくACLでインバウンドしか設定できないとRSがHGW側にも流れてしまいHGWがRAを応答することにはなるが、RAはHUBで拒否してしまうので実害はない。

## 問題点や懸念

今回の方法にはいくつかの問題点や懸念がある。

1. 現在のPythonスクリプトはRAリレーホストの起動時にいい加減なタイミングで開始する想定であり、RSに対してのRAは受信済みの状態でそのRAを保存出来ていない状態で開始してしまっていると思われる。そのため、RAリレーホストがRSに対してのRAを送信出来るようになるのはHGWの定期送信のRAを受信して以降になってしまう。HP620の場合はRAの定期送信が約40秒と十分短いため私は許容しているが、スクリプト起動時にHGWに対してRSを送信する処理を追加したほうがよいかもしれない

1. 偽装したRAはEthernetアドレスが本来のRAと比較するとRAリレーホストになってしまう。このRAがIPv6ホストに受け入れられるかは実装次第である。Linux, Windows11, Android, iOSで試す限りは今のところ動作しているが、OSのセキュリティが強化されていくと今後は拒否されてしまう可能性はある

1. 現在のPythonスクリプトではRAリレーホストのHUB側のEthernetのI/F(RAを送信する方)にIPv6アドレスが付与されない

1. 現在のPythonスクリプトではプリフィックス長が64ビットであることを前提としている

# 実践

## HUBの設定

HUBではHGWからのRAとDHCPv6のインバウンドを拒否するように設定する。私が使用しているMS510TXMではshow running-configで確認すると以下のような設定をしている。

```
ipv6 acl "from_HGW"
 sequence 1 deny protoKey 58 proto 58 sip any sport any 0 0 0 0 dip any dport any 0 0 0 0 icmp 1 134 0 frag 0 routing 0 tos any 0 0 asq any mirror any redirect any matchEvery 0 logging 0 
 sequence 2 deny protoKey 17 proto 17 sip any sport any 0 0 0 0 dip any dport 0 0 546 0 0 icmp any any any frag 0 routing 0 tos any 0 0 asq any mirror any redirect any matchEvery 0 logging 0 
 sequence 5 permit protoKey 0 proto 0 sip any sport any 0 0 0 0 dip any dport any 0 0 0 0 icmp any any any frag 0 routing 0 tos any 0 0 asq any mirror any redirect any matchEvery 1 logging 0 
```

sequence 1がRAの拒否、sequence 2がDHCPv6の拒否、sequence 5が全部許可である。

## RAリレーホスト

RAを偽装してリレーするホストでは以下のようなPythonするクリプトを用意する。Debianではpython3-scapyパッケージのインストールが必要

```:/root/ra.py
#! /usr/bin/python3

from scapy.all import *

master = "enp1s0"
slave = "enp6s18"
dns = ["be24:11ff:fed7:5e1b","be24:11ff:fe26:9901"]
searchlist = ["example.net"]
pkt2 = 0

def sniff_ra(pkt):
    global master
    global slave
    global dns
    global searchlist
    global pkt2

    if pkt.haslayer(ICMPv6ND_RA) and pkt[Ether].src != ifaces.dev_from_name(slave).mac:
        ether = Ether()
        ether.src = ifaces.dev_from_name(slave).mac
        ether.dst = "33:33:00:00:00:01"
        ip = IPv6()
        ip.src = pkt[IPv6].src
        ip.dst = pkt[IPv6].dst
        ra = ICMPv6ND_RA()
        ra.M = 0
        ra.O = 0
        ra.chlim = pkt[ICMPv6ND_RA].chlim
        ra.routerlifetime = pkt[ICMPv6ND_RA].routerlifetime
        if pkt.haslayer(ICMPv6NDOptPrefixInfo):
            prefix = ICMPv6NDOptPrefixInfo()
            prefix.prefixlen = pkt[ICMPv6NDOptPrefixInfo].prefixlen
            prefix.prefix = pkt[ICMPv6NDOptPrefixInfo].prefix
            prefix.validlifetime = pkt[ICMPv6NDOptPrefixInfo].validlifetime
            prefix.preferredlifetime = pkt[ICMPv6NDOptPrefixInfo].preferredlifetime
            rdnss = ICMPv6NDOptRDNSS()
            rdnss.lifetime = pkt[ICMPv6NDOptRDNSS].lifetime
            rdnss.dns = []
            for d in dns:
                rdnss.dns.append(pkt[ICMPv6NDOptPrefixInfo].prefix[:-1] + d)
            dnssl = ICMPv6NDOptDNSSL()
            dnssl.lifetime = pkt[ICMPv6NDOptRDNSS].lifetime
            dnssl.searchlist = searchlist
        lladdr = ICMPv6NDOptSrcLLAddr()
        lladdr.lladdr = pkt[ICMPv6NDOptSrcLLAddr].lladdr
        if pkt.haslayer(ICMPv6NDOptPrefixInfo):
            pkt2=(ether/ip/ra/prefix/rdnss/dnssl/lladdr)
        else:
            pkt2=(ether/ip/ra/lladdr)
        sendp(iface=slave, x=pkt2, verbose=0)
    elif pkt.haslayer(ICMPv6ND_RS) and pkt2 != 0:
        ether = Ether()
        ether.src = ifaces.dev_from_name(slave).mac
        ether.dst = pkt[Ether].src
        ip = IPv6()
        ip.src = pkt2[IPv6].src
        ip.dst = pkt[IPv6].src
        ra = ICMPv6ND_RA()
        ra.M = 0
        ra.O = 0
        ra.chlim = pkt2[ICMPv6ND_RA].chlim
        ra.routerlifetime = pkt2[ICMPv6ND_RA].routerlifetime
        if pkt2.haslayer(ICMPv6NDOptPrefixInfo):
            prefix = ICMPv6NDOptPrefixInfo()
            prefix.prefixlen = pkt2[ICMPv6NDOptPrefixInfo].prefixlen
            prefix.prefix = pkt2[ICMPv6NDOptPrefixInfo].prefix
            prefix.validlifetime = pkt2[ICMPv6NDOptPrefixInfo].validlifetime
            prefix.preferredlifetime = pkt2[ICMPv6NDOptPrefixInfo].preferredlifetime
            rdnss = ICMPv6NDOptRDNSS()
            rdnss.lifetime = pkt2[ICMPv6NDOptRDNSS].lifetime
            rdnss.dns = []
            for d in dns:
                rdnss.dns.append(pkt2[ICMPv6NDOptPrefixInfo].prefix[:-1] + d)
            dnssl = ICMPv6NDOptDNSSL()
            dnssl.lifetime = pkt2[ICMPv6NDOptRDNSS].lifetime
            dnssl.searchlist = searchlist
        lladdr = ICMPv6NDOptSrcLLAddr()
        lladdr.lladdr = pkt2[ICMPv6NDOptSrcLLAddr].lladdr
        if pkt2.haslayer(ICMPv6NDOptPrefixInfo):
            pkt3=(ether/ip/ra/prefix/rdnss/dnssl/lladdr)
        else:
            pkt3=(ether/ip/ra/lladdr)
        sendp(iface=slave, x=pkt3, verbose=0)

sniff(iface=master, filter="icmp6", prn=sniff_ra)
```

以下の変数の値は環境に合わせて修正が必要。

### master

RAリレーホストのHGW側のEthernetのI/F名を設定する。

### slave

RAリレーホストのHUB側のEthernetのI/F名を設定する。

### dns

内部DNSサーバーのリンクローカルアドレス(fe80::で始まるIPv6アドレス)を確認してその下位64ビットを設定する。

RDNSSオプションが不要であればこの設定は不要であり、rdnss変数の設定しているところをコメントアウトしてしまい、(ether/ip/ra/prefix/rdnss/dnssl/lladdr)の式からrdnssを外せばよい。

現在のPythonスクリプトでは内部DNSサーバーが一番古典的なModified EUI-64(RFC4861)でアドレスを決定していることを想定している。具体的には64ビットのプレフィックスに下位64ビットを繋げているだけである。

### searchlist

検索ドメインリストを設定する。

DNSSLオプションが不要であればこの設定は不要であり、dnssl変数の設定しているところをコメントアウトしてしまい、(ether/ip/ra/prefix/rdnss/dnssl/lladdr)の式からdnsslを外せばよい。

### 注意点

当初、RAリレーホストをProxmoxのCTでやろうとしたが、仮想ブリッジ経由だとCTでもVMでも何故かRSが受信出来なかった(ただし、RA受信前なら受信出来る？ tcpdumpで見ているとホスト側では受信出来ているのでRSはマルチキャストが全ノードではなく全ルータなのが影響してそう？)。

よくわからないので、HGW側のEthernetを今回の目的以外で使うことは無さそうなので専有してしまうことにした。具体的にはRAリレーホストをVMにして仮想ブリッジではなくHGW側のEthernetをパススルーすることで対応した。LXCでEthernetのパススルーも出来るようだが、ProxmoxのCTでの方法がわからず今回はVMで妥協した(もし出来たら別記事を作成する予定)。

# Windows11でのRDNSS

Windows11では以下にあるようにIPv4とIPv6がDHCPという設定になっているとRDNSSは無視されてしまうようだ。

https://learn.microsoft.com/en-us/answers/questions/884756/after-update-from-windows-10-to-windows-11-ipv6-rf

私が試した限りはIPv4が固定＋IPv6がDHCPでも同様なようでRDNSSは無視されてしまうようだ。

ただ、この状態でもDNSの名前解決がIPv4になるというだけであり、IPv6の名前解決に支障があるわけでもないので実害はないので私は許容しているが、気になるのであれば別途DHCPv6サーバーを用意するしかなさそうだ。その場合、今のPythonスクリプトではRAのOフラグを0としているので1に変更するべきである。

# 感想

私の場合、LAN内はIPv4の方が使い勝手もよいのでIPv6を使いたいのはWANだけなのに、とんでもなく苦労してしまった。

# おまけ

HGWのWIFIが使えなくなるのでWIFI APを追加する必要があり、WIFI7の製品も出てきているのでWIFI7のELECOM WRC-BE94XSを購入してAPモードで使うことにした(4ストリームはいらないので2ストリームの物で2.5Gbpsハブもあるのでこれにした)。

だが、このWRC-BE94XSがAPモードでもDHCPv6サーバーの応答をしてしまいWIFI APがDNSサーバーとして通知されてしまうという問題があり、結局やりたいことが出来ないじゃんみたいなことになってしまうと分かった。Wiresharkのパケットを添えてELECOMのサポートに相談したところ、修正していただけるということで2024年8月頃には修正されたファームが提供されるとのことだ。

この問題はWindows11 PCで試す限りは再現したりしなかったりで正直良くわからないが、RAのOフラグを0にしているのに問題発生時はWindowsがDHCPv6のInfomation-requestメッセージを送信してしまうときに起きるようなのでWindows側の動作も変な気がする。