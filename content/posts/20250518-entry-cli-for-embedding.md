+++
title = "embeddingのためのOSSコマンドラインツールを作っている"
date = "2025-05-18"
description = "毎週のように新しいモデルが発表されるLLMほどではないが，embedding model（埋め込み表現モデル）にもたくさんのモデルがある。埋め込み表現と，そのメジャーなアプリケーションであるベクトル検索のリサーチがてら，いろいろなembeddingを手軽に試せるembcliというOSSを作っている。"

[taxonomies]
categories = ["Long Posts"]
tags = ["embcli", "python", "oss", "embedding"]
+++

毎週のように新しいモデルが発表されるLLMほどではないが，embedding model（埋め込み表現モデル）にもたくさんのモデルがある。埋め込み表現と，そのメジャーなアプリケーションであるベクトル検索のリサーチがてら，いろいろなembeddingを手軽に試せる`embcli`というOSSを作っている。

GitHub: [embcli - CLI for Embeddings](https://github.com/mocobeta/embcli)

ライセンス: Apache License v2

設計思想や使い勝手は，さまざまなLLMをコマンドラインから呼べる[llm](https://github.com/simonw/llm)というツールにインスパイアされていて，ざっくりそのembedding版を目指している。なお[llmもembeddingsをサポートしていて](https://llm.datasette.io/en/stable/embeddings/index.html)，とはいえembedding modelのサポートはLLMよりは薄いので，embeddingの取扱いに特化したニッチツールがあっても良いかな...と思ったのがきっかけ。

ソフトウェアとしてはまだ粗々なものの，一通りのベースらしきものができたところでPyPIに初期バージョンを登録したので，紹介エントリを書く。

# インストールと使い方

たとえば[SentenceTransformer](https://sbert.net/index.html)モデルを試す場合，[embcli-sbert](https://pypi.org/project/embcli-sbert/)を`pip`でインストールする。

```bash
pip install embcli-sbert
```

依存を最小限にするためモデルごとにパッケージを公開していて，他にはOpenAIモデル用の[embcli-openai](https://pypi.org/project/embcli-openai/), Cohereモデル用の[embcli-cohere](https://pypi.org/project/embcli-cohere/)などがある。

サポートしているモデルの一覧は[README](https://github.com/mocobeta/embcli)を参照ください。プロプライエタリのモデルを使うにはAPIキーが必要です。

## embコマンド

`embcli-<model>`には`emb`というコマンドラインツールが含まれていて，サポートされるコマンドは

```bash
$ emb --help
Usage: emb [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  collections        List collections in the vector store.
  delete-collection  Delete a collection from the vector store.
  embed              Generate embeddings for the provided text or file...
  ingest             Ingest documents into the vector store.
  ingest-sample      Ingest example documents into the vector store.
  models             List available models.
  search             Search for documents in the vector store for the query.
  simscore           Calculate similarity score between two texts.
  vector-stores      List available vector stores.
```

で確認できる。各コマンドのヘルプも`emb <command> --help`で表示される。

主なコマンドと，その使い方の例を紹介していく。

## インストール済みモデルをリストする

`emb models`で，インストールされているモデルのリストを表示する。

```bash
$ emb models
SentenceTransformerModel
    Vendor: sbert
    Models:
    * sentence-transformers (aliases: sbert)
    See https://sbert.net/docs/sentence_transformer/pretrained_models.html for available models.
    Model Options:
```

ここでは`embcli-sbert`のみインストールしているので，`SentenceTransformerModel`だけが表示されている。他のパッケージをインストールすると，環境にインストールされているすべてのモデルが表示される。

## 埋め込み表現を生成する

`emb embed` コマンドで埋め込み表現を生成する。`--model` または `-m` オプションで，使うモデルを指定する。

```bash
$ emb embed -m sbert/all-MiniLM-L12-v2 "hello, embedding"
[0.010096956044435501, -0.07646115869283676, 0.07836000621318817, -0.017467740923166275, -0.08471577614545822, -0.08252009749412537, -0.021875102072954178, -0.04600825533270836, 0.002717185765504837, 0.04721956327557564, 0.05318789556622505, -0.011195648461580276, 0.07706273347139359, -0.02276509813964367, ...
```

## 2つの埋め込み表現の類似度（または距離）を計算する

`emb simscore` コマンドで，2つのセンテンスから生成した埋め込み表現の類似度または距離を計算する。`--similarity`または`-s`オプションで，計算にどのメトリクスを使うかを指定する。2つの埋め込み表現のdot productを計算する場合：

```bash
$ emb simscore -m sbert/all-MiniLM-L12-v2 -s dot "The cat drifts toward sleep." "Sleep dances in the cat's eyes."
0.6868138043884893
```

## セマンティックサーチ（ベクトル検索）

`embcli-<model>`には[Chroma](https://github.com/chroma-core/chroma)と，いくつかの小さなサンプルコーパスが付属していて，簡易的にembeddingのインデクシングと検索を試すことができる。手元にインデックスしたいテキストがある場合は，自前のコーパスも利用できる。

`emb ingest-sample`コマンド（または，手元にインデックスしたい自前のコーパスがすでにある場合は`emb ingest`コマンド）で，テキストコーパスから埋め込み表現を生成して，ローカルファイルシステム上のChromaデータベースにインデックスする。

`--collectoin`または`-c`オプションでコレクション名を指定し，`--corpus`でサンプルコーパス名を指定する。たとえば[dishes-ja](https://github.com/mocobeta/embcli/blob/main/packages/embcli-core/src/embcli_core/synth_data/dishes-ja.csv)コーパスをインデックスする場合

```bash
$ emb ingest-sample -m sbert/paraphrase-multilingual-MiniLM-L12-v2 -c menu-ja --corpus dishes-ja
Documents ingested successfully.
Vector store: chroma (collection: menu-ja)
Persist path: ./chroma
```

インデックス済みのコレクションは，`emb search`コマンドで検索できる。「おすすめのデザートは？」というクエリで検索すると，

```bash
$ emb search -m sbert/paraphrase-multilingual-MiniLM-L12-v2 -c menu-ja -q "おすすめのデザートは？"
Found 5 results:
Score: 0.08250103935120566, Document ID: 44, Text: クレープ（フランス）：非常に薄く繊細なパンケーキ。軽い生地で、甘くフルーツジャムやチョコレート、またはチーズ、ハム、野菜を詰めて提供されます。多用途で美味しい。
Score: 0.07426561166383495, Document ID: 85, Text: カンノーリ（イタリア）：シチリアの伝統的なデザート。筒状の生地を揚げたシェルに、甘いクリーム状のフィリング（リコッタチーズ）を詰めたお菓子。カリッとしたシェルと滑らかなフィリングが特徴です。
Score: 0.07029977647833369, Document ID: 81, Text: ストロープワッフル（オランダ）：薄いワッフル生地にキャラメルシロップを挟んだ焼き菓子。もちもちとした甘さで、温かい飲み物と一緒に楽しむことが多い。
Score: 0.059633586841592826, Document ID: 5, Text: マカロン（フランス）：軽いアーモンドメレンゲのシェルに、濃厚なクリームやガナッシュのフィリングを詰めた、色鮮やかで繊細なスイーツ。サクサクのシェルが柔らかくモチモチした内側に溶け込み、食感と視覚の絶妙なハーモニーを楽しめます。
Score: 0.05821842435995467, Document ID: 10, Text: ティラミス（イタリア）：コーヒーに浸したレディフィンガー、マスカルポーネチーズ、泡立てた卵、ココアパウダーの層からなる贅沢なデザート。クリーミーで濃厚な香りのこのデザートは、その名前の通り「元気を出そう」という意味で、純粋な至福の味わいを提供します。
```

軽量なSentenceTransformerモデルを使って日本語のテキストをインデックス／検索すると，正直イマイチな印象。多言語に強いCohere（プロプラ）のモデルなんかだと結構いい感じになります。

# 開発ステータスとこれから

開発ステータスはpre-Alphaで，これからやりたいことは

- ドキュメンテーション
- ローカルモデルのサポート拡充（特にllama.cpp/GGUF）
- CLIPなどマルチモーダルモデルのサポート
- Quantizationのサポート（int8, binary, など）
- サポートしているモデルのドキュメンテーションや原著論文を読んで解説を書く
- ベクトル検索以外のダウンストリームタスク（クラスタリング，文書分類，他）のサポート
- ベクトル検索で使う各種ベクトルデータベースのサポート

など。

勉強のためという漠としたイメージで始めたOSSプロジェクトですが，面白そう・便利かもと思った方はお好みのモデルのパッケージをインストールしてみてください。ドキュメンテーションがTODOとなっていますが（汗），READMEとコマンド付属のヘルプで大体の使い方はすぐに掴めると思います。[GitHub Issues](https://github.com/mocobeta/embcli/issues)でもフィードバック・リクエストを受け付けています。

# おまけ：設計ポリシーなど

[pluggy](https://pluggy.readthedocs.io/en/latest/)を使い，プラッガブルで拡張性の高い設計にしている。また，`uv workspace`を活用することで，コアライブラリと各種プラグインをモノレポで管理しつつ，依存ツリーをパッケージごとに管理しつつ，複数モデル（パッケージ）をインストールしても依存ツリーの一貫性が保たれるようにした。

このあたりの詳細は以前に書いたブログ[uv workspacesでスッキリ作るPythonモノレポ](https://blog.mocobeta.dev/posts/20250427-entry-monorepo-with-uv-workspaces/)と[uv workspacesとpluggyで作る，プラッガブルなPythonエコシステム](https://blog.mocobeta.dev/posts/20250429-entry-pluggy-uv-workspace/)で紹介しています。

