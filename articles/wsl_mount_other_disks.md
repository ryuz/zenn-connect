---
title: "WSL2で他のディスクをマウント"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [WSL]
published: true
---

## 同じPCの別の vhdx をマウントする

### Microsoft Store から入手した WSL を使う

おもに[こちら](https://docs.microsoft.com/ja-jp/windows/wsl/wsl2-mount-disk)の情報。


ディスクの管理から vhdx ファイルは作成可能なので、例えば Dドライブに ext4.vhdx のようなファイルを作って、PowerShell から

```
wsl --mount "D:\ext4.vhdx" --vhd --bare
```

とすれば、WSL 内に表れるので fdisk などでパーティーションを作成して mkfs.ext4 などのコマンドでフォーマットしておく。

以降は PowerShell から

```
wsl --mount "D:\ext4.vhdx" --vhd --partition 1 --name work-disk
```

のような感じで、 /mnt/wsl/work-disk などにマウントできる模様。ただ起動時に自動マウントさせるのがまだうまくいっていない。


### Microsoft Store から入手した WSL を使わない

ディスクの管理から接続した vhd ファイルは PowerShell から下記のコマンドで参照できるので、

```
GET-CimInstance -query "SELECT * from Win32_DiskDrive"
```

ディスク番号を調べて、例えば6番目のディスクになっていたら下記のような感じで

```
wsl --mount \\.\PHYSICALDRIVE6 --bare
```

とか

```
wsl --mount \\.\PHYSICALDRIVE6 --partition 1 --name work-disk
```

でマウントできる。

ディスクの管理から接続は PC を再起動すると切れるので、自動接続したい場合は
[こちら](https://www.sumaho-mation.com/vhd-auto/)の情報が参考になりそうです。

vhdmount.txt を以下のように用意しておき

```
sel vdisk file="d:\ext4.vhd"
attach vdisk
```

automount.bat を下記のように準備して、

```
diskpart -s D:\vhdmount.txt
```

タスクスケジューラを用いて、スタートアップ時をトリガに起動するように設定するとよいようです。


## NAS をマウントする

### NFS でマウント

おもに[こちら](https://scratchpad.jp/ubuntu-on-windows11-6/)を参考に、NSFでのマウントを試みてみました。


```
sudo mkdir -p /run/sendsigs.omit.d/rpcbind
sudo /etc/init.d/rpcbind start
sudo /etc/init.d/nfs-common start
```

我が家にあるのは QNAP の TS-219P+ ですが、/etc/fstab に、下記のような感じで書き足せば、マウントできるようです。


```
XX.XX.XX.XX:/nfsshare          /mnt/nfs         nfs rw,auto 0 0
```


### その他の方法

ほかにも[こちら](https://scratchpad.jp/ubuntu-on-windows11-6/)などによると、FTPFSやsshfsなども候補になりそうに思いました。

