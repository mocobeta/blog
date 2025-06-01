+++
title = "[embcli開発日誌] マルチモーダル埋め込みモデルCLIPで遊ぶ"
date = "2025-06-01"

[taxonomies]
categories = ["Short Posts"]
tags = ["embcli", "python", "oss", "embedding"]

[extra]
cover = "ricecurry.jpeg"
+++

マルチモーダル埋め込みモデル[CLIP](https://github.com/openai/CLIP)をコマンドラインから遊べるembcliプラグイン[embcli-clip](https://pypi.org/project/embcli-clip/)をリリースしました。

## CLIPとは

[CLIP: Connecting text and images](https://openai.com/index/clip/)

2021年にOpenAIが発表した，テキストと画像を同じベクトル空間上で扱えるマルチモーダル埋め込みモデル。より正確には，「画像とテキストのペアをトレーニングデータとしてContrastive Learningを行ったVisual Transformer」とのことです。商用利用可能なオープンソースライセンス（MIT）で公開されています。

## embcli-clipプラグイン

PyPIからインストールできます。

```bash
pip install embcli-clip
```

`embcli`については，以前の記事[embeddingのためのOSSコマンドラインツールを作っている](https://blog.mocobeta.dev/posts/20250518-entry-cli-for-embedding/)を参照ください。

### 画像から埋め込み表現を作る

gingercat.jpeg

![gingercat.jpeg](gingercat.jpeg)

以下のコマンドで，`gingercat.jpeg` という画像から，CLIPでエンコードしたembeddingが作れます。

```bash
emb embed -m clip --image gingercat.jpeg
```

### テキストと画像の類似度を計算する

desc.txt

```
A ginger cat with bright green eyes, lazily stretching out on a sun-drenched windowsill.
```

というテキストと，`gingercat.jpeg`の類似度を計算してみます。（正しくは，テキストをCLIPモデルでエンコードして得られるembeddingと画像をCLIPモデルでエンコードして得られるembeddingのコサイン距離を測っているのですが，長いので「テキストと画像の類似度」と表現してます。）

```bash
emb simscore -m clip -f1 desc.txt --image2 gingercat.jpeg 
0.33982698978267567
```

0.34くらいの類似度・・・といっても妥当なのかどうかわからないので，同じテキストと他の画像の類似度も計算してみます。

blackcat.jpeg

![blackcat.jpeg](blackcat.jpeg)

との距離は

```bash
emb simscore -m clip -f1 desc.txt --image2 blackcat.jpeg 
0.2251996520708121
```

orangehouse.jpeg

![orangehouse.jpeg](orangehouse.jpeg)

との距離は

```bash
emb simscore -m clip -f1 desc.txt --image2 orangehouse.jpeg 
0.16374113114522493
```

purplerose.jpeg

![purplerose.jpeg](purplerose.jpeg)

との距離は

```bash
emb simscore -m clip -f1 desc.txt --image2 purplerose.jpeg 
0.10988502060326195
```

なんとなく直感と合っていそうな値になりました。

### 画像をクエリにしてコーパスを検索する

テキストと画像が同じ空間上に埋め込まれるので，「画像をクエリとしてテキストを検索する」ことができます。

まず，embcliに付属しているサンプルコーパス[dishes-en](https://github.com/mocobeta/embcli/blob/main/packages/embcli-core/src/embcli_core/synth_data/dishes-en.csv)をCLIPでエンコードしてベクトルデータベースにインデックスします。

```bash
# dishes-enコーパスに含まれるテキストを，CLIPでエンコードして，menuというChromaDBコレクションに格納
emb ingest-sample -m clip -c menu --corpus dishes-en
Documents ingested successfully.
Vector store: chroma (collection: menu)
```

検索に使う画像は，（存在しない）カレーライスの画像。

ricecurry.jpeg

![ricecurry.jpeg](ricecurry.jpeg)

```bash
emb search -m clip -c menu --image ricecurry.jpeg 
Found 5 results:
Score: 0.00871434027958996, Document ID: 13, Text: Massaman Curry (Thailand): Rich mild sweet curry with origins from Persia. Contains potatoes peanuts and tender meat like beef or chicken. Complex fragrant spices create a deeply comforting and flavorful dish.
Score: 0.00856720166794799, Document ID: 4, Text: Green Curry (Thailand): Aromatic creamy coconut milk curry vibrant with green chilies and fragrant herbs. Packed with tender meat like chicken and various vegetables. Its spicy exotic taste transports your senses to Thailand with every bite.
Score: 0.008291618090874418, Document ID: 30, Text: Chicken Tikka Masala (India/UK): Pieces of chicken tikka grilled chicken simmered rich creamy spiced tomato gravy. Comforting flavorful dish enjoyed worldwide. Slightly sweet spiced.
Score: 0.008202290799093813, Document ID: 45, Text: Ratatouille (France): Flavorful Provençal vegetable stew eggplant zucchini bell peppers tomatoes onions garlic simmered herbs. Colorful healthy aromatic dish.
Score: 0.008183849397490686, Document ID: 26, Text: Butter Chicken (India): Tender marinated chicken simmered creamy tomato based gravy enriched with butter aromatic spices. Mildly spiced rich comforting. Globally popular Indian dish.
```

それっぽい検索結果です。

なお，画像からChatGPTでキャプションを生成して，生成したキャプションを検索クエリにしても，似たような結果になりました（ランキングは多少変わるものの上位５件中４件が同じ）。

```bash
emb search -m clip -c menu -q "A warm and comforting bowl of creamy chicken curry served with fluffy white rice."
Found 5 results:
Score: 0.03536559881204484, Document ID: 13, Text: Massaman Curry (Thailand): Rich mild sweet curry with origins from Persia. Contains potatoes peanuts and tender meat like beef or chicken. Complex fragrant spices create a deeply comforting and flavorful dish.
Score: 0.03304049320760643, Document ID: 26, Text: Butter Chicken (India): Tender marinated chicken simmered creamy tomato based gravy enriched with butter aromatic spices. Mildly spiced rich comforting. Globally popular Indian dish.
Score: 0.032370656263859175, Document ID: 4, Text: Green Curry (Thailand): Aromatic creamy coconut milk curry vibrant with green chilies and fragrant herbs. Packed with tender meat like chicken and various vegetables. Its spicy exotic taste transports your senses to Thailand with every bite.
Score: 0.030957274204826232, Document ID: 30, Text: Chicken Tikka Masala (India/UK): Pieces of chicken tikka grilled chicken simmered rich creamy spiced tomato gravy. Comforting flavorful dish enjoyed worldwide. Slightly sweet spiced.
Score: 0.0268596218357381, Document ID: 75, Text: Beef Stroganoff (Russia): Sautéed pieces beef served in sauce smetana sour cream often mushrooms onions. Typically served over noodles or rice. Creamy savory comforting.
```

### もっと詳しく

より詳しい使い方はドキュメンテーションを参照ください。

- [embcli Multimodal Usage](https://embcli.mocobeta.dev/#multimodal_usage/)
- [embcli-clip for CLIP Models](https://embcli.mocobeta.dev/#multimodal_model_plugins/#embcli-clip-for-clip-models)