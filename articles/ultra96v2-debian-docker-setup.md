# Ultra96V2 環境を Docker で作ってみる

## 概要


## イメージの取得

試しに作成してみた SD カードイメージを[こちら](https://onedrive.live.com/download?cid=E643EA309C96C6F6&resid=E643EA309C96C6F6%2142124&authkey=ADTriYzkOZAZdCg)に置いておきます。

私の OneDrive に置いているものなので、そのうち無くなるかもしれませんがひとまずの実験ということでご了承ください。

Dokcer 起動のためのカーネルコンフィギュレーション修正と、RPU(Cortex-R5)認識の為の修正のみ行っております。

現在  v2021.1.1 を元に作成を行っており、このパッケージには 

- 5.4.0-xlnx-v2020.2-zynqmp-fpga
- 5.10.0-xlnx-v2021.1-zynqmp-fpga

の２つのカーネルが入っているようですが、5.10.0 でRPUの認識がうまくいかなかったため、デフォルトで 5.4.0-xlnx-v2020.2-zynqmp-fpga が起動するようにしております。


## イメージの書き込みと起動

ファイルを解凍して、出てきたファイルをddコマンドや[ Win32DiskImager](https://forest.watch.impress.co.jp/docs/review/1067836.html)などでSDカードに焼き込んでください。

Ultra96V2 に差し込んで起動すると debian が起動します。

この状態で、[オリジナルの配布](https://github.com/ikwzm/ZynqMP-FPGA-Linux)から大きな手は加えていませんので、まずは UART などのシリアルから wifi などのネットワーク設定を行ってください。

ネットーワーク設定を後回しにする場合は date コマンドで日付の設定だけは行ってください。タイムスタンプが不整合を起こすと以降の処理で不都合が起こる場合があります。

初期状態でアカウントは fpga パスワードは fpga となっているようです。

## 基本パッケージのインストール

最初に アカウント fpga でログインして下記の操作を行います。

```
cd /home/fpga/debian
sudo dpkg -i linux-image-5.4.0-xlnx-v2020.2-zynqmp-fpga_5.4.0-xlnx-v2020.2-zynqmp-fpga-3_arm64.deb
sudo dpkg -i linux-headers-5.4.0-xlnx-v2020.2-zynqmp-fpga_5.4.0-xlnx-v2020.2-zynqmp-fpga-3_arm64.deb
sudo dpkg -i fclkcfg-5.4.0-xlnx-v2020.2-zynqmp-fpga_1.7.2-1_arm64.deb
sudo dpkg -i u-dma-buf-5.4.0-xlnx-v2020.2-zynqmp-fpga_3.2.4-0_arm64.deb
```

(なぜかエラーになるので調査中です)

もしカーネル 5.10.0 を使う場合は下記になります。

```
cd /home/fpga/debian
sudo dpkg -i linux-image-5.10.0-xlnx-v2021.1-zynqmp-fpga_5.10.0-xlnx-v2021.1-zynqmp-fpga-4_arm64.deb
sudo dpkg -i linux-headers-5.10.0-xlnx-v2021.1-zynqmp-fpga_5.10.0-xlnx-v2021.1-zynqmp-fpga-4_arm64.deb
sudo dpkg -i fclkcfg-5.10.0-xlnx-v2021.1-zynqmp-fpga_1.7.2-1_arm64.deb
sudo dpkg -i u-dma-buf-5.10.0-xlnx-v2021.1-zynqmp-fpga_3.2.4-0_arm64.deb
```

ここで一度システムをリブートしておくとよいかもしれません。

```
sync
sudo reboot
```

その他、システムの更新も一度しておいた方が良いでしょう。

```
sudo apt update
sudo apt -y upgrade
```

adduser でユーザーを追加したり、samba を入れたり、必要なことがあれば実施しておくとよいでしょう。


## Docker のインストール

まず、インストールに必要な curl を入れます。

```
sudo apt update
sudo apt -y upgrade
sudo apt install -y curl
```

次に Docker をインストールします。

```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

下記のようにすると sudo 不要で実行できます。

```
sudo usermod -aG docker <username>
```

(以降、今後執筆予定)