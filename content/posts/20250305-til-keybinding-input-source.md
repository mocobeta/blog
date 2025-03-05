+++
title = "Windows向けUS配列キーボードをLinux/MacOSで切り替えて使う時に日本語/英語入力ソース切替えショートカットを統一したい（Emacsキーバインド派向け）"
date = "2025-03-05"

[taxonomies]
categories = ["Posts"]
tags = ["til", "linux", "macOS"]
+++

こまかすぎてタイトルが何を言っているのかわからなくなったけど，ピンとくる人には伝わるのではないか・・・。

### 結論

`<Alt>-space`(Linux), `<Opt>-space`(MacOS)に落ち着いた。

### もう少しこまかいモチベーションの説明

- 仕事用の，会社から貸与されているPCはMacOS, 個人PCはWindows PCにLinuxをインストールしたもの
- 個人所有のWindows向けUS配列キーボード（REALFORCE）を仕事用PCと個人PCで共用したい
- 2つのPCのキーバインドやショートカットに，なるべく差異がないようにしたい（Windows特有またはMac特有のキーに依存しないようにしたい）
- VSCodeのキーバインドを[Emacs Friednly Keymap](https://marketplace.visualstudio.com/items?itemName=lfs.vscode-emacs-friendly)でEmacsにしていて，エディタのキーバインドとなるべくコンフリクトしないようにしたい

### ファイナルアンサー？

しばらくはこれで妥協できそう。ものすごく使いやすいというわけではないので，どうしても慣れなければ変えるかも。

### その他

VSCodeで，Altを押すとカーソルのフォーカスがメニューにあたるという困った事態に遭遇した。いつの間にか（余計な）仕様がVSCodeに入っていたようで，

[[VSCode]フォーカスがメニューに移動してしまう](https://www.codelab.jp/blog/?p=3303)

を参考にAltでのフォーカス移動を無効化して解決。

### 試して却下したショートカット候補

- `<Ctrl>-space`: 日本語入力をサポートしているMacOSでUS配列キーボードを選んだ時の，デフォルトの入力ソース切り替えショートカット（のひとつ）。問題は，同じショートカットがEmacsの範囲選択開始に割り当てられていること。Emacsキーバインドが体に染み付いている体には，このセマンティクスを今さら変えるのはとても辛い。
- `<Shift>-space`: Windows系だと入力ソース切り替えによく使われるショートカット。MacOSだと`<Shift>`単独ではショートカットキーに指定できない（`<Command>`など他の修飾キーと組み合わせると使える）。
