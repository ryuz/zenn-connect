---
title: "ZynqMP用に Devicetree Overlay できる Dockerイメージを buildx で作ってみる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [FPGA, ZynqMPp, Docker]
published: true
---

## 概要

ZynqMP には Cortex-A53 が搭載されていて Linux が動くわけですが、最近は arm64(aarch64)用の Docker も普及してきており大変便利です。

私は ZynqMP 上で OpenCV や riscv-gnu-toolchain を利用したりするのですが、SD カードを作り直すたびにコンパイルしていては大変ですので Docker イメージにしてしまうと便利そうです。

しかしながら、OpenCV のようなライブラリは、ビルド時だけでなく、実行時にも必要となるため、ビルド環境と実行環境の両方をこなせる必要があります。

その際、実行環境としては ZynqMP の PL に bitstream をダウンロードしたり、uio や u-dma-buf などを利用するために、Devicetree Overlay が必要ですし、OpenCV の 表示を X-Window で行ったりも必要です。

加えて、 SD カードと限られたメモリ内でのビルドはいろいろと制約があるので、Docker の [buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/) の機能で、ハイパフォーマンスなPC上でビルドしてしまおうというのがこの記事の趣旨です。


## 作成したビルド環境

今回作成した環境は [こちら](https://github.com/ryuz/jelly/tree/master/docker/zynqmp_jelly)に置いております。

今回は Windows11 + WSL2(Ubuntu 20.04) の環境にて Docker Desktop を用いてビルドを行なっております。


## Devicetree Overlay できるようにする

今回、一番重要なのが Devicetree Overlay できるようにする点ですが、これは Twitter 上で多くの方のお知恵をお借りして解決しました。

以下が、今回作成した docker-compose.yml ですが、

```
version: "3"

services:
  zynqmp_develop:
    build: .
    image: ryuz88/zynqmp_jelly
    container_name: zynqmp_jelly
    ports:
      - 20022:20022
    privileged: true
    volumes:
      - /home/${LOCAL_USER}:/home/${LOCAL_USER}
      - /lib/firmware:/lib/firmware
      - /configfs:/configfs
      - /dev:/dev
    environment:
      LOCAL_USER: ${LOCAL_USER}
      LOCAL_UID: ${LOCAL_UID}
      LOCAL_GID: ${LOCAL_GID}
```

このうち

```
    privileged: true
```

 と volumes の

```
      - /lib/firmware:/lib/firmware
      - /configfs:/configfs
      - /dev:/dev
```

の部分が、最終的に Devicetree Overlay できるようにするのに必要だった項目です。

これでちゃんと、Devicetree Overlay で uio などを増やすと、/dev の下にデバイスが追加されてくれ、privileged によって、/sys/class 以下から情報を受け取ることも可能となるようです。


## ホストと同じユーザーで作業できるようにする

これは、ビルド環境を作る場合の宿命的な部分でもありますが、前処理や後処理で他のツールと連携しないといけないことが多々あります。

ホストの home をマウントして、ホストと同じ権限でファイル操作できると何かと便利そうです。

方法はいくつかあるようですが、結局試してみて、Docker 内でホストと同じ UID、GID を持つユーザーを新規に作る方法に落ち着きました。

下記のような compose.sh という docker-compose を呼び出す前に変数設定するスクリプトを準備したうえで

```
set -eu
cat <<EOT > .env
LOCAL_USER=`whoami`
LOCAL_UID=`id -u`
LOCAL_GID=`id -g`
EOT

docker-compose $@
```

 docker-compose.yml に

```
    environment:
      LOCAL_USER: ${LOCAL_USER}
      LOCAL_UID: ${LOCAL_UID}
      LOCAL_GID: ${LOCAL_GID}
```

や volumes 部分の

```
      - /home/${LOCAL_USER}:/home/${LOCAL_USER}
```

を加え、下記のような entrypoint.sh を

```
#!/bin/bash

USER_NAME=${LOCAL_USER:-user}
USER_ID=${LOCAL_UID:-9001}
GROUP_ID=${LOCAL_GID:-9001}

echo "Starting with UID : $USER_ID, GID: $GROUP_ID"
useradd -u $USER_ID -o -m $USER_NAME --shell /usr/bin/bash
groupmod -g $GROUP_ID $USER_NAME
echo "$USER_NAME:fpga" | chpasswd

gpasswd -a $USER_NAME sudo

/usr/sbin/sshd -D
```

Dokerfile の最後で

```
# entrypoint
COPY ./files/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

とすることで、ユーザーを作成してから sshd を起動するようにしております。

注意点はここでパスワードが固定で設定されてしまう点です。
開発環境がクローズな場合を想定しておりますので、そうでない場合は別途、パスワードの保護や、アクセス制限を行う工夫の追加をご検討ください。


## buildx でのビルド

ビルド自体は Docker Desktop には buildx は含まれているようですので

ビルダーを作ったあと

```
docker buildx create custum-builder
docker buildx use custum-builder
```

```
./buildx.sh
```

とすれば zynqmp_jelly.tar が出来上がります。

もっとも、現在の内容はすでに[こちら](https://hub.docker.com/repository/docker/ryuz88/zynqmp_jelly)に push しておりますので、ZynqMP 側から

```
docker pull ryuz88/zynqmp_jelly:latest
```

としていただければ、利用可能になるはずです。

起動は、利用したいユーザーアカウントで

```
./compose.sh up -d
```

とするだけで、終了時は

```
./compose.sh down
```

とすれば、起動時したユーザー名でポート 20022 で ssh を待ち受けるコンテナが起動します。

その際、オリジナルの home をボリュームとして接続しますので .bashrc も今使っているものが読まれます。

Docker コンテナ内では /.dockerenv がありますので、下記のような感じの if を追加しておくとよいでしょう。

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

これで、SDカードを作り直すたびに OpenCV 等のビルドを行う状況から脱却できると嬉しいです。


