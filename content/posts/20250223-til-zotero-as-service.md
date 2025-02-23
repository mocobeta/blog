+++
title = "Zoteroをデーモンで起動する"
date = "2025-02-23"

[taxonomies]
categories = ["Posts"]
tags = ["til", "linux", "zotero"]
+++

読む/読んだ論文やテクニカルレポートを保存するのに， [Zotero](https://www.zotero.org/) を使っている。Zoteroには便利なブラウザプラグインがあって，ブラウザからワンクリックでZoteroに連携してくれるのだけれど，プラグインが動作するためにはZoteroアプリが起動していないといけない。たぶんWindowsやMacでは，パソコンのスタートアップ時に起動させておく設定が簡単にできるのだけど，Linuxデスクトップだと，ブラウザプラグインのために毎回アプリを起動する必要があって面倒だった。

[Running zotero in background via
Trapped inside a tölva](https://edoput.it/2021/07/22/background-zotero.html)

で，ヘッドレスモードのZoteroをサービスとして稼働させておく方法が紹介されている。

ひとつ困るのが，

> The only issue is that now launching the GUI will complain that ``zotero is already running’’ but I will take that over the alternative.

で，バックグラウンドですでに起動しているzoteroプロセスがあると，GUIアプリが立ち上がってくれない。

とりあえずナイーブに，zotero GUI アプリを起動するスクリプトとzoteroサービスを起動するスクリプトを `~/.local/bin` に置いて使うことにした。

```sh
# ~/.local/bin/zotero
#!/bin/sh

# stop current zotero process
systemctl --user stop zotero
sleep 1

# start zotero application
/usr/bin/zotero
```

```sh
# ~/.local/bin/zotero-start
#!/bin/sh

systemctl --user start zotero
```

運用はこう

- ログイン直後: バックグラウンドプロセスでヘッドレスzoteroが自動起動
- GUIアプリを使う時: `~/.local/bin/zotero` を実行
- GUIアプリ終了後: `~/.local/bin/zotero-service` を（気づいたタイミングで）叩いてバックグラウンドサービスを起こす

（そのうち，バックグラウンドプロセスが落ちてたら自動起動させるようにしたい）

