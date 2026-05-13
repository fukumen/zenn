---
title: "Deskflow(Flatpak版)がWaylandセッションでクリップボードの共有出来ない問題の回避策"
emoji: "🖱️"
type: "tech"
topics: ["deskflow", "flatpak", "wayland", "linux"]
published: true
---

## はじめに

Flatpak 版 [Deskflow](https://github.com/deskflow/deskflow) の v1.26.0 時点では Wayland セッションのクリップボードの共有に `wl-clipboard` を使う必要があるが、Flatpak 版に含まれておらずうまく機能しない。

**PR #9431 (Inputcapture clipboard integration)** にて `wl-clipboard` に依存せずに Wayland のネイティブポータル (`xdg-desktop-portal`) を使用したクリップボード統合の実装を進めているようだが、待ってられないので `wl-clipboard` を追加した Flatpak のカスタムビルドを行う回避策を作成したので紹介する。

修正済みのソースコードは以下のリポジトリに push しており、git cloneしてビルドするだけ。

* [fukumen/deskflow (fix-wl-clipboard-flatpak ブランチ)](https://github.com/fukumen/deskflow/tree/fix-wl-clipboard-flatpak)

## 環境

Ubuntu 25.10 の KDE Plasma セッションで動作を確認。

## ローカルでのビルドとインストール手順

`flatpak-builder` を使ってユーザー領域にカスタムビルドをインストールする。

```bash
# リポジトリをクローン（今回はフォークしたリポジトリを使用）
git clone https://github.com/fukumen/deskflow.git
cd deskflow
git checkout fix-wl-clipboard-flatpak

# ビルドに必要なSDKをインストール
flatpak install flathub org.kde.Sdk//6.10 org.kde.Platform//6.10

# ローカル（ユーザー権限）でビルド＆インストール
flatpak-builder --user --install --force-clean build-dir deploy/linux/flatpak/org.deskflow.deskflow.yml
```

インストール後、以下のコマンドで起動するとカスタム版（`user` インストール）が優先して起動する。

```bash
flatpak run org.deskflow.deskflow
```

## 更新履歴

* 2026/5/13 masterブランチから作成

