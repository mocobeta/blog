+++
title = "LanceDBのPythonSDKで全文検索，ベクトル検索，ハイブリッド検索をする"
date = "2025-08-08"

[taxonomies]
categories = ["Short Posts"]
tags = ["lancedb", "search", "embedding", "python"]
+++

[LanceDB](https://lancedb.github.io/lancedb/)の[Python SDK](https://lancedb.github.io/lancedb/python/python/)を使って，全文検索，ベクトル検索，ハイブリッド検索を試します。

## 準備

まずはLanceDBのテーブルを作成します。LanceDBのテーブルは，[いろいろな方法で作成できます](https://lancedb.github.io/lancedb/guides/tables/)が，ここではembeddingモデルがシームレスに組み込めるPydantic方式で定義します。

```python
import lancedb
from lancedb.embeddings import get_registry
from lancedb.pydantic import LanceModel, Vector

# データベースに接続（または新規作成）
db = lancedb.connect("data/mylance.db")

# sentence-transformersモデルを取得
model = get_registry().get("sentence-transformers").create(name="all-MiniLM-L6-v2")

# 映画のタイトルと，そのembeddingを持つテーブルを定義
# LanceModelは特殊なPydanticモデルで，LanceDBのテーブルスキーマを定義する。
# "Vector"タイプが黒魔術になっていて，こう書くと暗黙的にソースフィールドからembeddingが計算されて，vectorフィールドに格納される。
class BestMovies(LanceModel):
    vector: Vector(model.ndims()) = model.VectorField()  # embeddingフィールド
    movie: str = model.SourceField()  # embeddingのソースとなるテキストフィールド

# 初期データ
data = [
    {"movie": "The Godfather is Francis Ford Coppola's epic crime drama often cited as one of the greatest films ever made."},
    {"movie": "The Shawshank Redemption is a powerful and uplifting prison story that consistently tops audience polls."},
    {"movie": "Citizen Kane, directed by Orson Welles, is celebrated for its revolutionary cinematic techniques."},
    {"movie": "Pulp Fiction is Quentin Tarantino's non-linear crime film and a landmark of independent cinema."},
    {"movie": "2001: A Space Odyssey is Stanley Kubrick's visually stunning and philosophical science fiction epic."},
    {"movie": "The Dark Knight, directed by Christopher Nolan, is hailed for its dark themes and iconic villain."},
    {"movie": "Schindler's List is Steven Spielberg's poignant and powerful historical drama about the Holocaust."},
    {"movie": "Seven Samurai is Akira Kurosawa's influential epic that set the template for the modern action film."},
    {"movie": "Casablanca is a timeless romantic drama known for its iconic lines and performances."},
    {"movie": "Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards."}
]

# テーブル名，スキーマ，初期データを指定してテーブルを作成
table = db.create_table("best_movies", schema=BestMovies, data=data)
```

## 全文検索用のインデックス作成

`Table.create_fts_index`メソッドで，全文検索インデックス（＝転置インデックス）を作成します。

```python
table.create_fts_index("movie")
```

## (オプショナル) ANNインデックスを作成

`Table.create_index`メソッドで，近似最近傍検索用のインデックスを作成します。`index_type`に`IVF_FLAT`を指定するとInverted File Flatインデックスが作成されます。ここでは，ドキュメント数が少ないのでコメントアウト。

```python
#table.create_index(metric="cosine", vector_column_name="vector", index_type="IVF_FLAT")
```

## 検索する

`Table.search`メソッドで検索を行います。`query_type`に`fts`（全文検索），`vector`（ベクトル検索），`hybrid`（ハイブリッド検索）を指定できます。

### 全文検索

`queyr_type`に`fts`を指定すると，全文検索になります。

```python
print("Full-text search results:")
fts_results = table.search("history", query_type="fts").limit(3).to_pydantic(model=BestMovies)
for result in fts_results:
    print(result.movie)
```

```bash
Full-text search results:
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
```

"history"という検索キーワードを含む映画だけがヒットする。

### ベクトル検索

`query_type`に`vector`を指定すると，ベクトル検索になります。

```python
print("\nVector search results:")
vector_results = table.search("history", query_type="vector").limit(3).to_pydantic(model=BestMovies)
for result in vector_results:
    print(result.movie)
```

```bash
Vector search results:
Schindler's List is Steven Spielberg's poignant and powerful historical drama about the Holocaust.
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
The Godfather is Francis Ford Coppola's epic crime drama often cited as one of the greatest films ever made.
```

ふんわり，"history"に関連しそうな映画がヒットする。

### ハイブリッド検索

ベクトル検索の文脈での「ハイブリッド検索」とは，全文検索とベクトル検索を組み合わせて，より精度の高い検索結果を得ることを指します。`query_type`に`hybrid`を指定するとハイブリッド検索になります。

```python
print("\nHybrid search results:")
hybrid_results = table.search("history", query_type="hybrid").limit(3).to_pydantic(model=BestMovies)
for result in hybrid_results:
    print(result.movie)
```

```bash
Hybrid search results:
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
Schindler's List is Steven Spielberg's poignant and powerful historical drama about the Holocaust.
The Godfather is Francis Ford Coppola's epic crime drama often cited as one of the greatest films ever made.
```

ベクトル検索の時と，若干違う結果になります。"history"という検索キーワードを含む映画が1位にヒットしている。

ハイブリッド検索の裏側では，全文検索とベクトル検索をそれぞれ独立に実行したあと，２つの検索結果をマージして，`RRF`リランカーでリランクが行われます。そのあたりの詳細は，また別の機会に。