+++
title = "LinuxでシステムクロックをNTPサーバーと同期する"
date = "2025-02-24"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "linux"]
+++

今日知った初心者殺しのトリビア：Arch Linuxは，インストール時のデフォルト設定だとシステムクロックがリモート時刻サーバーと同期されない。

```
sudo timedatectl set-ntp true
```

で，NTP同期を有効にする。

`timedatectl` コマンドの使い方は

[【timedatectl】コマンドでタイムゾーンとシステムクロックを設定 - suer TIL](https://atsum.in/linux/timedatectl/)

に詳しい。

時刻同期されていないことに気づかずに，半年くらい使っていたよ。いろいろ大丈夫だったんか。