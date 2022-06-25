---
title: "Vivado のプロジェクトからファイルリスト取得"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [fpga, vivado]
published: true
---

例えば以下のようなファイルを file_list.tcl として準備して

```tcl
open_project "XXXX.xpr"

set f [open "file_list.txt" w]
foreach fname [get_files] {
  puts $f $fname
}
close $f

close_project
```

下記のように実行

```
vivado -m64 -mode batch -source file_list.tcl
```

