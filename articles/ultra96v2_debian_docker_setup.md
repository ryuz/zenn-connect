---
title: Docker を使える Ultra96V2 用 Debian 環境構築
emoji: 🍊
type: tech
topics: [FPGA, ZynqMP, Ultra96, Debian]
published: true
---


## 概要

以前 Qiita の方に[Ultra96V2 Debian環境構築メモ](https://qiita.com/Ryuz/items/7498f2f8a7ab838aa196)という記事を書いておりますが、SDカードの作成に Linux マシン(もしくは Virtual Boxなどの仮想環境)が必要だったり、Docker を使えるようにするのが大変だったりと、いろいろ手間がありましたので、Windows しか環境が無い場合でももう少しお気軽にセットアップできないか実験的に取り組んでみました。


## イメージの取得

試しに作成してみた SD カードイメージを下記においておきます。

-  [ZynqMP-FPGA-Linux-v2021.1.1-Ultra96V2-Docker-20220622.zip](https://onedrive.live.com/download?cid=E643EA309C96C6F6&resid=E643EA309C96C6F6%2142127&authkey=AF2eo7gjuQptjSs)


私の OneDrive に置いているものなので、そのうち無くなるかもしれませんがひとまずの実験ということでご了承ください。

Dokcer 起動のためのカーネルコンフィギュレーション修正と、RPU(Cortex-R5)認識の為の修正のみ行っており、初回起動時に自動的にSDカードのサイズに合わせたパーティーション拡張を行うようにしております。

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

最初に アカウント fpga でログインしてパッケージのインストールを行います。

しかしながら、なぜか gcc-10 だとエラーが出るため 先に gcc-9 を入れて切り替えます。

```
sudo apt update
sudo apt install -y gcc-9
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 10
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9  9
```

下記で、切り替えメニューが出ますので gcc-9 を選びます。

```
sudo update-alternatives --config gcc
```

その後、下記のようにパッケージをインストールします。

```
cd /home/fpga/debian
sudo dpkg -i linux-image-5.4.0-xlnx-v2020.2-zynqmp-fpga_5.4.0-xlnx-v2020.2-zynqmp-fpga-3_arm64.deb
sudo dpkg -i linux-headers-5.4.0-xlnx-v2020.2-zynqmp-fpga_5.4.0-xlnx-v2020.2-zynqmp-fpga-3_arm64.deb
sudo dpkg -i u-dma-buf-5.4.0-xlnx-v2020.2-zynqmp-fpga_3.2.4-0_arm64.deb
sudo dpkg -i fclkcfg-5.4.0-xlnx-v2020.2-zynqmp-fpga_1.7.2-1_arm64.deb
```


もしカーネル 5.10.0 を使う場合は下記になります。

```
cd /home/fpga/debian
sudo dpkg -i linux-image-5.10.0-xlnx-v2021.1-zynqmp-fpga_5.10.0-xlnx-v2021.1-zynqmp-fpga-4_arm64.deb
sudo dpkg -i linux-headers-5.10.0-xlnx-v2021.1-zynqmp-fpga_5.10.0-xlnx-v2021.1-zynqmp-fpga-4_arm64.deb
sudo dpkg -i u-dma-buf-5.10.0-xlnx-v2021.1-zynqmp-fpga_3.2.4-0_arm64.deb
sudo dpkg -i fclkcfg-5.10.0-xlnx-v2021.1-zynqmp-fpga_1.7.2-1_arm64.deb
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


## ホスト名設定(お好みで)

他にも Zybo など複数のボードを持っている人は hostname が同じだと紛らわしいので名前変更しています。

今回は

```
sudo nano /etc/hostname
```

として debian-fpga を debian-ultra96 に変更しました。

変更して再起動すると sudo するたびに「sudo: unable to resolve host」と出たので

```
sudo sh -c 'echo 127.0.1.1 $(hostname) >> /etc/hosts'
```

としております。


## ユーザー作成(お好みで)

fpga というユーザー名のままでもいいのですが、github を使う場合などもあるので自分の名前でユーザーを作ることも可能です。

```
sudo adduser <username>
```

sudo権限もつけておきます

```
sudo gpasswd -a <username> sudo
```


## Samba設定(お好みで)

Windowsからの利用が楽なように Samba の設定をします。

```
sudo pdbedit -a <username>
sudo nano /etc/samba/smb.conf
```

smb.conf は [homes] セクションで

```
read only = no
```

にしておけばフルアクセスできます。

さらに [global] のセクションに

```
unix extensions = no
wide links = yes
```

を足すと、シンボリックリンクの先もアクセスできるようです。

以下でサービスを再起動します。

```
sudo systemctl restart smbd nmbd
```

## NFSの設定(お好みで)

以前[こちら](https://ryuz.hatenablog.com/entry/2021/06/04/221256)に記事を書きましたが、NASサーバーなどがあれば、NFSを設定しておくと、SDカードの容量や寿命の問題を緩和出来て便利です。

```
sudo apt -y install nfs-common 
sudo apt -y install autofs 
```

でインストールして

```
sudo nano /etc/auto.master
```

として末尾に

```
/-    /etc/auto.mount
```

を追加

```
sudo mkdir /mnt/nfs 
sudo nano /etc/auto.mount 
```

で

```
/mnt/nfs -fstype=nfs,rw XX.XX.XX.XX:/nfsshare
```

みたいな感じで、共有フォルダを指定

```
sudo systemctl restart autofs 
```



## Docker のインストール

以前[こちら](https://ryuz.hatenablog.com/entry/2022/03/24/140650)に書いた記事を踏襲して設定を行います。

次に Docker をインストールします。

```
sudo apt update
sudo apt install -y docker.io
sudo apt install -y docker-compose
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo systemctl restart docker
```

下記のようにすると次回ログインから sudo 不要で実行できます。

```
sudo usermod -aG docker <username>
```

試しに

```
sudo docker run hello-world
```

として動けばＯＫです。


なお、Docker を動かす為に、オリジナルの uEnv.txt を下記のように修正しております。

```
linux_boot_args_systemd=systemd.unified_cgroup_hierarchy=0
```


## Jelly の sample を Docker で動かしてみる

拙作の ZynqMP用のDocker イメージとして、[zynqmp_jelly](https://hub.docker.com/repository/docker/ryuz88/zynqmp_jelly)というものを作成しており、OpenCV や riscv コンパイラなど、[Jelly](https://github.com/ryuz/jelly.git) のサンプルのビルドに必要なものを一式収めたイメージを用意しております。


```
git clone https://github.com/ryuz/jelly.git
cd jelly/docker/zynqmp_jelly
./compose.sh up -d
```

これで、zynqmp_jelly イメージが実行され(初回のみ少し長いダウンロードあり)、ポート 20022 で ssh 接続を待ち受けるようになる。

ssh 接続はイメージを立ち上げたのと同じアカウント名で、パスワードが fpga として接続できるようにしている。

Dockerイメージ内は、ホストのユーザーID で home をボリュームとしてマウントしているので、そのまま作業できる。

Docker イメージ用のパスの追加が必要だが、 home の .bashrc もそのまま読まれるので

```
if [ -f /.dockerenv ]; then
  export PATH="/opt/opencv4/bin:$PATH"
  export PKG_CONFIG_PATH="/opt/opencv4/lib/pkgconfig:$PKG_CONFIG_PATH"
  export LD_LIBRARY_PATH="/opt/opencv4/lib:$LD_LIBRARY_PATH"

  export PATH="/opt/opencv3/bin:$PATH"
  export PKG_CONFIG_PATH="/opt/opencv3/lib/pkgconfig:$PKG_CONFIG_PATH"
  export LD_LIBRARY_PATH="/opt/opencv3/lib:$LD_LIBRARY_PATH"

  export PATH="/opt/grpc/bin:$PATH"
  export PATH="/opt/riscv/bin:$PATH"
fi
```

などを追加しておくと便利です。

事前にREADMEに従って別PCで bit ファイルを合成してコピーしておく必要がありますが、例えば下記のようにすれば u-dma-buf のサンプルが実行できます。


```
cd jelly/projects/ultra96v2_udmabuf_sample/app/
make run
```


