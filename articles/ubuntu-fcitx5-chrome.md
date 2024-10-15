---
title: "Ubuntu 24.10+fcitx5+Chromeでインプットメソッドの変換候補ウィンドウの表示位置がおかしい問題の対処法"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ubuntu", "fcitx5", "Google Chrome"]
published: true
---
# 概要

うちの以下の環境でインプットメソッドの変換候補ウィンドウの表示位置がおかしな位置になってしまう問題が起きていた。

・Ubuntu 24.10
・Waylandセッション
・fcitx5
・Google Chrome バージョン129
・Preferred Ozone platform: X11（Waylandだと問題無し）

# (その場しのぎの)対処法

うちの環境では.local/share/applications/google-chrome.desktopのコマンドオプションに–-gtk-version=4オプションがついていた。これを外してやりgtk 3が使われるようにしてやると問題が解消した。

Snap版Chromiumの場合、–-gtk-version=4オプション無しなら問題なし、–-gtk-version=4オプションありだとそもそも日本語入力が出来ず。

# その他

Ubuntu 24.04で同様の問題が起きていてUbuntu 24.10へアップグレードしても駄目で、その後あれこれ試した結果の対処法なので24.04で有効かは不明。

# おまけ

私がPreferred Ozone platformをX11で使っているのはWindow Resizerという拡張機能を使うため。

そのうちgtk 3もX11も捨てられるだろうしその場しのぎにしかなっていない。
