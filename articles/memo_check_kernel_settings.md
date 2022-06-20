---
title: カーネルの設定確認(備忘録)
emoji: 🍕
type: tech
topics: [FPGA, ZynqMP]
published: true
---


## カーネル設定を覗く

```
zcat /proc/config.gz
```


## デバイスツリーを覗く


```
sudo dtc /sys/firmware/fdt 2> /dev/null
```

