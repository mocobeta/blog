+++
title = "Arch LinuxからブラザーのDCP-J528Nプリンターを使う"
date = "2025-02-24"

[taxonomies]
categories = ["Posts"]
tags = ["til", "linux"]
+++

ブラザーのプリンターを持っているのだけれど，Linux用ドライバのダウンロードページには，Debian系とRedHat系向けのインストーラーしかなく，その他のディストリビューションからは使えないのかと諦めていた。

ちゃんと調べてみると（Perplexity AIに聞いた），私が持っている型番のプリンターは[AirPrintという，ドライバーレスで印刷ができるAppleの規格に対応していて](https://faq.brother.co.jp/app/answers/detail/a_id/12493/~/airprint%E3%81%AE%E4%BD%BF%E3%81%84%E6%96%B9)，LinuxでもCUPS経由で使うことができた。

AirPrintに対応しているプリンターなら，ドライバーインストール不要で，ネットワーク経由で印刷ができるようになる。Appleさまさまだ。

コマンド

```
sudo pacman -S cups
sudo systemctl enable --now cups

sudo usermod -aG lp <your-user>  # これはもしかするといらないのかも

sudo lpadmin -p AirPrint -E -v "ipp://PRINTER_IP/ipp/print" -m everywhere

sudo pacman -S system-config-printer
system-config-printer # 一覧で，「AirPrint」が認識されているはず
```

参考リンク

[Perplexity AIの検索結果 "arch linux brother printer driver"](https://www.perplexity.ai/search/arch-linux-brother-printer-dri-LtmA059aSlKHW4M4el1cGQ)

[Arch Brother printer driver install - Arch Linux BBS](https://bbs.archlinux.org/viewtopic.php?id=276550)

[Debian から AirPrint 対応プリンタに印刷する - @tsuchm](https://qiita.com/tsuchm/items/783bcb07c6a0a4e1db63)

