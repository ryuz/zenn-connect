---
title: Kria KV260 環境構築メモ
emoji: 😸
type: tech
topics: [FPGA, ZynqMP, Kria, KV260]
published: true
---

## 概要

[Kria KV260](https://japan.xilinx.com/products/som/kria/kv260-vision-starter-kit.html)用の環境を構築した際の個人的メモです(備忘録)。

同じような使い方をする人の参考になれば幸いです。

以前 Qiita の方に[Ultra96V2 Debian環境構築メモ](https://qiita.com/Ryuz/items/7498f2f8a7ab838aa196)という記事を書いておりますが、最終的に似たような環境を KV260 に作るのが目的です。


## Ubuntu の SDカード作成

[こちら](https://ubuntu.com/download/amd-xilinx)から、SDカードのイメージを取得して、dd コマンド(Linuxの場合)や、Win32DiskImager(Windowsの場合) でSDカードに焼き込みます。

私が使ったバージョンは iot-kria-classic-desktop-2004-x03-20211110-98.img.xz でした。

SDが出来上がったら、ボードに差し込んで電源を入れれば起動します。

初期アカウント ubuntu の 初期パスワードは ubuntu のようなので、ログインするとパスワードを変えるように促されるので設定します。


## CUI/GUI の切り替え 

GUI使わないという人はさっさとCUIに切り替えてしまう手もあるかもしれません(お好みで)。

```
# CUIにする
sudo systemctl set-default multi-user.target

# GUIにする
sudo systemctl set-default graphical.target
```

## ユーザーを作る 

デフォルトのユーザー(ubuntu)をそのまま使ってもよいのですが、新規に作る場合は。

```
sudo adduser <username>
```

でユーザーを作って


```
sudo gpasswd -a <username> sudo
```

で sudo 権限を付けておきましょう。


## fclkcfg と u-dma-buf のインストール

ZynqMP開発の定番となっている ikwzm氏の[fclkcfg](https://github.com/ikwzm/fclkcfg)と[u-dma-buf](https://github.com/ikwzm/udmabuf)をインストールします。

```
git clone --recursive --depth=1 -b v1.7.2 https://github.com/ikwzm/fclkcfg-kmod-dpkg
cd fclkcfg-kmod-dpkg
sudo debian/rules binary
cd ..
sudo dpkg -i fclkcfg-5.4.0-1017-xilinx-zynqmp_1.7.2-1_arm64.deb 


git clone --recursive --depth=1 -b v3.2.3 https://github.com/ikwzm/u-dma-buf-kmod-dpkg
cd u-dma-buf-kmod-dpkg
sudo debian/rules binary
cd ..
sudo dpkg -i u-dma-buf-5.4.0-1017-xilinx-zynqmp_3.2.3-0_arm64.deb
```

## RPU (Cortex-R5) を認識させる

詳しい記事は[こちら](https://ryuz.hatenablog.com/entry/2022/05/04/100016)に書いております。

下記の方法で現状のデバイスツリーを取得します。

```
sudo dtc /sys/firmware/fdt 2> /dev/null > system.dts
```

取得した system.dts に対して、まず interrupt-controller@f9010000 のセクションを探して "gic:" を追記しました。

```
    gic: interrupt-controller@f9010000 {
```

次に symbols の直前に下記を追記しました。

```
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        rproc_0_dma: rproc@0x7ed40000 {
            no-map;
            compatible = "shared-dma-pool";
            reg = <0x0 0x7ed40000 0x0 0x100000>;
        };
        rproc_0_reserved: rproc@0x7ed00000 {
            no-map;
            reg = <0x0 0x7ed00000 0x0 0x40000>;
        };

        rproc_1_dma: rproc@0x7ef00000 {
            no-map;
            compatible = "shared-dma-pool";
            reg = <0x0 0x7ef00000 0x0 0x40000>;
        };
        rproc_1_reserved: rproc@0x7ef40000 {
            no-map;
            reg = <0x0 0x7ef40000 0x0 0x100000>;
        };
    };

    zynqmp-rpu {
        compatible = "xlnx,zynqmp-r5-remoteproc-1.0";
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        core_conf = "split";
        r5_0: r5@0 {
            #address-cells = <2>;
            #size-cells = <2>;
            ranges;
            memory-region = <&rproc_0_reserved>, <&rproc_0_dma>;
            pnode-id = <0x7>;
            mboxes = <&ipi_mailbox_rpu0 0>, <&ipi_mailbox_rpu0 1>;
            mbox-names = "tx", "rx";
            tcm_0_a: tcm_0@0 {
                reg = <0x0 0xFFE00000 0x0 0x10000>;
                pnode-id = <0xf>;
            };
            tcm_0_b: tcm_0@1 {
                reg = <0x0 0xFFE20000 0x0 0x10000>;
                pnode-id = <0x10>;
            };
        };
        r5_1: r5@1 {
            #address-cells = <2>;
            #size-cells = <2>;
            ranges;
            memory-region = <&rproc_1_reserved>, <&rproc_1_dma>;
            pnode-id = <0x8>;
            mboxes = <&ipi_mailbox_rpu1 0>, <&ipi_mailbox_rpu1 1>;
            mbox-names = "tx", "rx";
            tcm_1_a: tcm_1@0 {
                reg = <0x0 0xFFE90000 0x0 0x10000>;
                pnode-id = <0x11>;
            };
            tcm_1_b: tcm_1@1 {
                reg = <0x0 0xFFEB0000 0x0 0x10000>;
                pnode-id = <0x12>;
            };
        };
    };
    
    zynqmp_ipi1 {
        compatible = "xlnx,zynqmp-ipi-mailbox";
        interrupt-parent = <&gic>;
        interrupts = <0 29 4>;
        xlnx,ipi-id = <7>;
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        // APU<->RPU0 IPI mailbox controller
        ipi_mailbox_rpu0: mailbox@0xff990000 {
            reg = <0x00 0xff990600 0x00 0x20>,
                  <0x00 0xff990620 0x00 0x20>,
                  <0x00 0xff9900c0 0x00 0x20>,
                  <0x00 0xff9900e0 0x00 0x20>;
            reg-names = "local_request_region",
                "local_response_region",
                "remote_request_region",
                "remote_response_region";
            #mbox-cells = <1>;
            xlnx,ipi-id = <1>;
        };
    };

    zynqmp_ipi2 {
        compatible = "xlnx,zynqmp-ipi-mailbox";
        interrupt-parent = <&gic>;
        interrupts = <0 30 4>;
        xlnx,ipi-id = <8>;
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        // APU<->RPU0 IPI mailbox controller
        ipi_mailbox_rpu1: mailbox@ff3f0b00 {
            reg = <0x00 0xff3f0b00 0x00 0x20>,
                  <0x00 0xff3f0b20 0x00 0x20>,
                  <0x00 0xff3f0940 0x00 0x20>,
                  <0x00 0xff3f0960 0x00 0x20>;
            reg-names = "local_request_region",
                    "local_response_region",
                    "remote_request_region",
                    "remote_response_region";
            #mbox-cells = <1>;
            xlnx,ipi-id = <2>;
        };
    };
```

下記のように dtc でコンパイルして、FAT領域にコピーすれば完了です。

```
dtc -I dts -O dtb system.dts -o system.dtb
sudo cp system.dtb /boot/firmware/
```

リブートして /sys/class/remoteproc/ の下にプロセッサが２つ見えていれば成功です。

## bootgenを入れる

PLを使うアプリで必要となるので bootgenも入れておきます。

```
git clone https://github.com/Xilinx/bootgen
cd bootgen/
make
sudo cp bootgen /usr/local/bin/
```

## Docker を入れる

公式のUbuntu イメージは比較的簡単にDockerを入れることができるので、今回は環境構築に活用します。

```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install docker-compose
```

下記のように特手ユーザーに権限を付けると sudo が不要になります。

```
sudo usermod -aG docker <username>
```


## Samba を入れる

必須ではありませんが、Windowsユーザーの場合は入れておくと何かと便利かと思います。

```
sudo apt update
sudo apt install -y samba
sudo nano /etc/samba/smb.conf
```

として、/etc/samba/smb.conf を編集します。

[home]セクションのコメントアウトをすべて外して、

```
read only = no
```

とします。また [global] のセクションに

```
unix extensions = no
wide links = yes
```

を足すと、シンボリックリンクの先もアクセスできるようです。

ユーザーを追加は下記のコマンドで行うことができ

```
sudo pdbedit -a <username>
```

次のようにして再起動して反映可能です。

```
sudo systemctl restart smbd nmbd
```
