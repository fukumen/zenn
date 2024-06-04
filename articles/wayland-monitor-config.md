---
title: "【Wayland】【Ubuntu 24.04】コマンドラインによるマルチモニタとシングルモニタの切り替え"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wayland", "gnome", "ubuntu"]
published: true
---
# 概要

Windowsでは「Windowsキー+P」でマルチモニタとシングルモニタの切り替えが可能だが、Ubuntuでも同様のことが出来ないか調べたところgnomeに同様のショートカットキーが無さそうと分かった。

解像度変更のシェルスクリプトについて以下の記事があったので、この記事ではこれをベースにシェルスクリプトを作成する。
https://qiita.com/QiitaYkuyo/items/7c8762e8fd5b077a8aa4

作成したシェルスクリプトをgnomeで任意のショートカットキーに割り当てることでマルチモニタとシングルモニタの切り替えを実現する。

# シェルスクリプト

```:monitor.sh
#!/bin/sh

serial="$(gdbus call \
    --session \
    --dest org.gnome.Mutter.DisplayConfig \
    --object-path /org/gnome/Mutter/DisplayConfig \
    --method org.gnome.Mutter.DisplayConfig.GetResources | awk '{print $2}' | tr -d ',')"

if [ "$1" = "s" ]; then
    gdbus call --session --dest org.gnome.Mutter.DisplayConfig --object-path /org/gnome/Mutter/DisplayConfig --method org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig \
        $serial 1 '[
            (0, 0, 1.25, 0, true, [("DP-1", "5120x2160@59.999", [] )] )
        ]' '[]' > /dev/null
elif [ "$1" = "m" ]; then
    gdbus call --session --dest org.gnome.Mutter.DisplayConfig --object-path /org/gnome/Mutter/DisplayConfig --method org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig \
        $serial 1 '[
            (0, 0, 1.25, 0, true, [("DP-1", "5120x2160@59.999", [] )] ), 
            (4096, 0, 1.25, 0, false, [("HDMI-2", "3840x2160@60.000", [] )] )
        ]' '[]' > /dev/null
elif [ "$1" = "g" ]; then
    gdbus call --session --dest org.gnome.Mutter.DisplayConfig --object-path /org/gnome/Mutter/DisplayConfig --method org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig \
        $serial 1 '[
            (0, 0, 1, 0, true, [("DP-1", "5120x2160@59.999", [] )] )
        ]' '[]' > /dev/null
else
    gdbus call \
        --session \
        --dest org.gnome.Mutter.DisplayConfig \
        --object-path /org/gnome/Mutter/DisplayConfig \
        --method org.gnome.Mutter.DisplayConfig.GetCurrentState \
        | tr ',' '\n' | egrep '\(\(|@' | tr -d "[('"
fi
```
「5120x2160@59.999」や「3840x2160@60.000」の記載されている行がモニタの設定部分になる。

見ての通りコマンドのオプションに与えた一文字で処理分けしており、何をやろうとしているかは以下の通り。
- sのときはシングルモニタ（モニタの設定は1行）
- mのときはマルチモニタ（モニタの設定は2行）
- gのときはシングルモニタでスケールを1（モニタの設定は1行）
- オプション無しのときはモニタの設定詳細をするときに使用するとコネクタ名と解像度の一覧を表示（後述）

## $serialの後ろの1

$serialの後ろの1は、1のときに一時的な変更、2だと永続的な変更になる。

1を指定すると ~/.config/monitors.xml は反映されないので再ログイン後は元の設定に戻る。

2を指定すると変更を反映するか聞かれて「変更を反映」を選択すると ~/.config/monitors.xml に反映されるので再ログイン時にも設定が反映された状態になる。

## モニタの設定詳細

```
            (0, 0, 1.25, 0, true, [("DP-1", "5120x2160@59.999", [] )] ), 
            (4096, 0, 1.25, 0, false, [("HDMI-2", "3840x2160@60.000", [] )] )
```

- 1つ目と2つ目の値はX座標とY座標の指定。3つ目で指定するスケールに1以外を指定するときは解像度にスケールを反映した値にする必要あり（例：5120/1.25=4096）
- 3つ目の値はスケール値。ディスプレイ設定で任意倍数のスケーリングをONにしていると1.25とか1.5が指定できる
- 4つ目の値は画面の回転のようです（1:90度、2:180度、3:240度のようだが未確認）
- 5つ目の真偽値はプライマリか否か（trueのときプライマリ）
- 「DP-1」のところはコネクタ名の指定（詳細後述）
- 「5120x2160@59.999」のところは解像度とリフレッシュレートの指定（詳細後述）

## コネクタ名と解像度の一覧

シェルスクリプトをオプション無しで実行すると以下のようなリストが表示される。

```
 DP-1
 5120x2160@59.999
 5120x2160@29.995
 3840x2160@59.997
 2560x1440@59.951
（中略）
HDMI-2
 3840x2160@60.000
 3840x2160@59.940
 3840x2160@50.000
 3840x2160@30.000
 （以下略）
```

「DP-1」や「HDMI-2」がコネクタ名なのでこれを指定する。それらの下に一覧になっているのが解像度とリフレッシュレートなのでこの中から選んで指定する。

参考にした記事ではGetResourcesメソッドの結果を使っているが、私の環境ではリフレッシュレート部分の指定が駄目のようで上手く行かなかった（小数点以下の桁数の関係？）。GetCurrentStateメソッドで取得した@がついてる文字列の方を使うと上手く行ったので、この記事ではそれを紹介している。

# ショートカット

gnomeのキーボードの設定の中に独自のショートカットが指定出来るので今回用意したシェルスクリプトを指定する。

うちでは以下のような設定にしてみた。

- シングルモニタ：「monitor.sh s」を「Ctrl+Super(Windowsキー)+s」
- マルチモニタ：「monitor.sh m」を「Ctrl+Super(Windowsキー)+m」
- シングルモニタでスケールを1：「monitor.sh g」を「Ctrl+Super(Windowsキー)+g」

# 参考元やメモ

- 概要欄に記載した参考にした記事：https://qiita.com/QiitaYkuyo/items/7c8762e8fd5b077a8aa4
- さらにその大元：https://smile-jsp.hateblo.jp/entry/2021/05/04/223356
- ApplyMonitorsConfigするpythonスクリプトでGetCurrentStateメソッドで取得した情報の理解に助かった：https://gist.github.com/strycore/ca11203fd63cafcac76d4b04235d8759
- gnome-monitor-configを使う方法があるようでこれを使ったほうが早かったかも？：https://qiita.com/QiitaYkuyo/items/816e6440793a3f3a1a14
- gnome-monitor-configにorg.gnome.Mutter.DisplayConfigの各メソッドの引数の説明らしきものがあったので参考になった：https://github.com/jadahl/gnome-monitor-config/blob/master/src/org.gnome.Mutter.DisplayConfig.xml
