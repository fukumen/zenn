---
title: "Proxmox 8.1でGPU passthrough(cezanne編)"
emoji: "👋"
type: "tech"
topics: ["proxmox"]
published: false
---

# 概要

VMにOVMFを使ってのdGPUのパススルーはネット上に情報も多くあり苦労なく動作したが、iGPUのパススルーに相当はまったので設定のまとめ。複数の情報を繋ぎ合わせないと上手く行かなかった。

以降の説明は3パターンの設定例。
1. iGPU+dGPU
1. dGPUのみ
1. iGPUのみ

iGPUを使うときはinitrdでのvfioの設定が余分に必要。

dGPUのみのときはそれらは不要だがカーネルのコマンドラインで「initcall_blacklist=sysfb_init」が必要。
あと、dGPUのみの環境は潰してしまったので若干情報があやふや（vfioのあたり）。

なお、AMD Ryzen 5 5600GのiGPUでは上手く行ったが同様の設定をIntel i5-12400のiGPUでも試したところ上手く行かずエラー43になってしまう。

# 環境

## 共通
* Proxmox 8.1.4
* proxmox-kernel-6.5.11-7-pve-signed
* Windows 11(23H2)
* VMはOVMFでq35

## iGPU+dGPU
* CPU: AMD Ryzen 5 5600G
* M/B: ASUS TUF GAMING B550M-PLUS(BIOS:3404)
* GPU: AMD Radeon RX 6600 XT

## dGPUのみ
* CPU: AMD Ryzen 5 5600G
* M/B: ASUS TUF GAMING B550M-PLUS(BIOS:3404)
* GPU: AMD Radeon RX 6600 XT

## iGPUのみ
* CPU: AMD Ryzen 5 5600G
* M/B: Asrock A520M Pro4(BIOS:3.20)

# 設定

## BIOS
IOMMUとSVMをイネーブルにする。

あと、dGPUを使う場合はiGPU側をプライマリになるように設定しておくのとマルチモニタをイネーブルにしておく。

![UEFI](/images/proxmox-gpu-passthrough-cezanne-UEFI.png =500x)
*dGPUを使う場合の設定*

## カーネルのコマンドライン

設定例の「amd_pstate=passive consoleblank=60」はGPUのパススルーとは無関係。

### iGPU+dGPUとiGPUのみ

```:/etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt pcie_acs_override=downstream,multifunction amd_pstate=passive consoleblank=60"
```

### dGPUのみ

```:/etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init amd_pstate=passive consoleblank=60"
```

## vfio

以下のvfio設定の後、リブートしlspci -vによりパススルーしたいiGPUやdGPUにvfio-pciがロードされていることを確認すること。

### iGPU+dGPU

設定後、「update-initramfs -u」しておくこと。（全バージョンを更新したときは-k allも付与）

```:/etc/initramfs-tools/modules
vfio
vfio_iommu_type1
vfio_pci
```

vfio_virqfdはカーネルの6.2からvfioに統合されたそうで指定不要。

```:/etc/modprobe.d/vfio.conf
options vfio-pci ids=1002:1638,1002:1637 disable_vga=1
```
idsのところはパススルーするiGPUを指定する。

### dGPUのみ

設定不要のようです。
もし駄目なら「/etc/initramfs-tools/modules」はiGPU+dGPUと同様の設定をしてみる？

あと、複数のdGPUが搭載されているようなときは「/etc/modprobe.d/vfio.conf」や「/etc/initramfs-tools/scripts/init-top/bind_vfio.sh」のような設定をしてやることで切り離すdGPUをコントロールできるはず。
そのときはカーネルのコマンドラインの「initcall_blacklist=sysfb_init」は要らなさそう。

### iGPUのみ

「/etc/initramfs-tools/modules」と「/etc/modprobe.d/vfio.conf」はiGPU+dGPUと同様の設定が必要。

また、iGPUのみのときは以下の設定が必要。

```:/etc/initramfs-tools/scripts/init-top/bind_vfio.sh
#!/bin/sh

PREREQ=""
prereqs()
{
        echo "$PREREQ"
}

case $1 in
prereqs)
        prereqs
        exit 0
        ;;
esac

#replace with the pci buses you wish to override and bind to the vfio-pci driver.
devices="0000:05:00.0 0000:05:00.1"

bpath=/sys/bus/pci
driver=vfio-pci

for d in $devices; do
        echo $driver > "$bpath/devices/${d}/driver_override"
        echo $d > "$bpath/drivers/vfio-pci/bind"
done

modprobe -i vfio-pci

exit 0
```
devicesのところはパススルーするiGPUを指定する。

## iGPUのROM

### iGPU+dGPUとiGPUのみ

iGPUのROMファイルなしでOVMFのVMを起動しようとするとdmesgにROMが読めないようなエラーが出るのでROMを用意する必要あり。
ネット上でよく見かける「/sys/bus/pci/devices/hogehoge/rom」をダンプする方法は使えない模様。
（だから上記のエラーが出るのだと思いますが）

自分のM/BのBIOSを用意してWindows上で[UBU](https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/30357)に食わせます（UBUを展開したディレクトリにBIOSを置いてubu.batを叩くだけ）。
解析が終わったら「2」を選んだ後「s」を選ぶとvbios_1638.datとAMDGopDriver.efiが取得出来る。
（1638の方はiGPU毎に別ファイルがあるのでiGPUに合わせたものを取得する）

その後、AMDGopDriver.efiは[edk2-BaseTools-win32](https://github.com/tianocore/edk2-BaseTools-win32)のEfiRomを使用して変換が必要。

```
EfiRom.exe -f 0x1002 -i 0xffff -e AMDGopDriver.efi
```

UBUにより取得出来たvbios_1638.datとEfiRomにより取得出来たAMDGopDriver.romを「/usr/share/kvm/」に格納しておく。

あと、OVMFではなくSeabiosで使うならこの面倒なROMファイルの用意はなしでも行けるようです。

### dGPUのみ

6600 XTではROMなしで問題ありませんでした。

## VMのパススルーの設定例

iGPUのパススルーをする場合、後述のRadeonResetBugFixがないとダメそうなのでWindowsのインストールの前にパススルーを設定するのではなく、以下のようにしてやる必要があり。

1. ビデオ規定でWindowsをインストール
1. 必要であればvitioのドライバをインストール
1. リモートデスクトップを有効にしてビデオをnoneに変更
1. iGPUのパススルーを設定
1. ビデオカードのドライバとRadeonResetBugFixをインストール

### iGPU+dGPU

```:/etc/pve/qemu-server/<iGPU VM ID>.conf
hostpci0: 0000:0a:00.0,pcie=1,romfile=vbios_1638.dat,x-vga=1
hostpci1: 0000:0a:00.1,pcie=1,romfile=AMDGopDriver.rom
```

```:/etc/pve/qemu-server/<dGPU VM ID>.conf
hostpci0: 0000:03:00.0;0000:03:00.1,pcie=1,x-vga=1
```
PCIバスのところはパススルーするdGPUやiGPUを指定する。

### dGPUのみ

「/etc/pve/qemu-server/<dGPU VM ID>.conf」を参照。

### iGPUのみ

「/etc/pve/qemu-server/<iGPU VM ID>.conf」を参照。

## RadeonResetBugFix

### iGPU+dGPUとiGPUのみ

[RadeonResetBugFix](https://oomza.cutegay.software/inga-lovinde/RadeonResetBugFix/releases)をインストールする。

「RadeonResetBugFixService.exe install」するときはcmdやPowerShellの管理者モードで実行する必要あり。また「RadeonResetBugFixService.exe」がそのままサービスに登録されるので永続的な場所に追いて実行する必要あり。

### dGPUのみ

6600 XTではRadeonResetBugFixが無くても問題ありませんでした。

# 参考

* [https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) とその日本語版
* https://pve.proxmox.com/wiki/PCI_Passthrough
* https://github.com/isc30/ryzen-7000-series-proxmox
* https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a
* https://www.reddit.com/r/VFIO/comments/16mrk6j/amd_7000_seriesraphaelrdna2_igpu_passthrough/
* man initramfs-tools

# 問題点

HDMI接続だとiGPUはOVMFの画面がで表示されなかったり、デスクトップの表示開始がRadeonResetBugFix頼みになってしまう模様。
DisplayPort接続だと再起動時は問題あるがシャットダウン→起動だと問題なさそう。
M/BのUEFIやその設定によってUEFIやgrub表示時の解像度が変わるようなのでそれらによるような気もする？

# おまけ

IntelのiGPUもそのうち何とかしてみたいが、ネットで探してもiGPUは情報少なめだったり古かったり上手く行かないみたいな物し見つからず、一旦ギブアップ。

SR-IOVなら丁寧なまとめ記事がある。
→ https://www.derekseaman.com/2023/11/proxmox-ve-8-1-windows-11-vgpu-vt-d-passthrough-with-intel-alder-lake.html

なお、事務作業をしたいだけであればiGPUのパススルーを頑張らなくてもDisplayLinkのUSBビデオアダプタを使えば簡単にUSBパススルー出来るのでお金で解決するという方法もある。
