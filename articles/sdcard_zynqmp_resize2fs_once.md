---
title: ZynqMP用SDカードの自動パーティーション拡張に挑戦してみる
emoji: ⛳
type: tech
topics: [FPGA, ZynqMP]
published: true
---


## 概要

先日、[ZynqMP 用の SDイメージを SD カードを使わずに作る](https://ryuz.hatenablog.com/entry/2022/06/17/154424)というのをやってみた。

これで、Linux環境が無い人にも[ Win32DiskImager](https://forest.watch.impress.co.jp/docs/review/1067836.html)などでSDカードのイメージ共有がやりやすくなった。

折角なので、そこからさらに、Raspberry PI のように初回起動時にSDカードサイズにあわせて自動調整する機能を考えたい。

## resize2fs_once スクリプト

Raspberry PI の場合[ここ](https://github.com/RPi-Distro/pi-gen/blob/master/stage2/01-sys-tweaks/files/resize2fs_once)にあるようなスクリプトを使っているようだ。

調べてみると既に[なひたふ氏がZynq用に踏み込んだ詳細](http://nahitafu.cocolog-nifty.com/nahitafu/2019/08/post-2df6c8.html)をされていた。

そのまままねさせて頂いて、resize2fs_once に parted の行を1行追加させて頂いて、下記のようなスクリプトとした。

```resize2fs_once
#!/bin/sh
### BEGIN INIT INFO
# Provides:          resize2fs_once
# Required-Start:
# Required-Stop:
# Default-Start: 3
# Default-Stop:
# Short-Description: Resize the root filesystem to fill partition
# Description:
### END INIT INFO
. /lib/lsb/init-functions
case "$1" in
  start)
    log_daemon_msg "Starting resize2fs_once"
    ROOT_DEV=$(findmnt / -o source -n) &&
    parted -s /dev/mmcblk0 unit s "resizepart 2 -1s" &&
    resize2fs $ROOT_DEV &&
    update-rc.d resize2fs_once remove &&
    rm /etc/init.d/resize2fs_once &&
    log_end_msg $?
    ;;
  *)
    echo "Usage: $0 start" >&2
    exit 3
    ;;
esac
```

## img の加工

以前の[この記事](https://ryuz.hatenablog.com/entry/2022/06/17/154424)と同様に
/mnt/usb2 にイメージの第二パーティーション(ext4)がマウントされている前提で進める。

まず下記のように上のファイル(resize2fs_once)をコピーして実行権限を付ける。

```
sudo cp resize2fs_once /mnt/usb2/etc/init.d/
sudo chmod 755         /mnt/usb2/etc/init.d/resize2fs_once
```

そして、chroot で中に入って parted コマンドのインストールと起動予約を行う。

```
sudo mv /mnt/usb2/etc/resolv.conf    /mnt/usb2/etc/resolv.conf.org
sudo cp /etc/resolv.conf             /mnt/usb2/etc
sudo cp /usr/bin/qemu-aarch64-static /mnt/usb2/usr/bin

sudo chroot /mnt/usb2/

apt update
apt install -y parted
update-rc.d resize2fs_once defaults
exit

sudo mv /mnt/usb2/etc/resolv.conf.org /mnt/usb2/etc/resolv.conf
sudo rm /mnt/usb2/usr/bin/qemu-aarch64-static
```
