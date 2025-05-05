+++
title = "ZolaのブログテンプレートにOGPメタタグをつける"
date = "2025-05-05"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "zola", "blogging"]
+++

このブログは[Zola](https://www.getzola.org/)という静的サイトジェネレータと，Zolaコミュニティのブログテーマ[serene](https://www.getzola.org/themes/serene/)で作っている。

Zolaもsereneもミニマルな思想で作られているためか，SNS用の[OGPメタタグ](https://ogp.me/)などはデフォルトでは生成されない。Blueskyなどにポストしたときに少しさびしい感じなので，sereneテーマを少しカスタマイズしてOGPメタタグをつけることにした。

Zolaテーマのカスタマイズ方法は，[Customizing a theme](https://www.getzola.org/documentation/themes/extending-a-theme/)に書かれている。一つのテンプレートファイルを丸ごと置き換えで良い場合は簡単で，書き換えたいテンプレートファイルを`/template`以下にコピーしてきて編集すれば，デフォルトのテンプレートファイルよりそちらが優先されるようになる。たとえば今回はsereneテーマの`post.html`を変更したいので，

1. `/themes/serene/templates/post.html`を`/templates/post.html`にコピー
2. `post.html`に，↓のような感じでOGPメタタグを追加する

```html
# templates/post.html
...
{% if page.description %}
<meta name="description" content="{{ page.description }}">
<meta property="og:description" content="{{ page.description }}">
{% endif %}
...
<meta property="og:title" content="{{ page.title }} | {{ config.title }} by @{{ config.extra.id }} ">
<meta property="og:type" content="website">
<meta property="og:url" content="{{ page.permalink }}">
{% if page.extra.cover %}
<meta property="og:image" content="{{ page.permalink }}{{ page.extra.cover }}">
{% else %}
<meta property="og:image" content="/img/search-system-book-penguin.jpg">
{% endif %}
```

[https://github.com/mocobeta/blog/blob/main/templates/post.html](https://github.com/mocobeta/blog/blob/main/templates/post.html)

`og:image` は，ブログポスト内にカバーイメージがある場合はそれを使い，なければデフォルトの画像にフォールバックするようにしている。

テンプレートのシンタックスはJinja2と大体同じで馴染みやすい。

Blueskyに投稿するとこんな感じ。

なお，デフォルトのカバー画像は[「検索システムー実務者のための開発改善ガイドブック」](https://www.lambdanote.com/products/ir-system)の表紙絵です。ペンギンかわいい。

