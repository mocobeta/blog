+++
title = "LanceDBのRust APIで全文検索，ベクトル検索，ハイブリッド検索をする"
date = "2025-08-09"

[taxonomies]
categories = ["Short Posts"]
tags = ["lancedb", "search", "embedding", "rust"]
+++

[LanceDB](https://lancedb.github.io/lancedb/)の[Rust API](https://docs.rs/lancedb/latest/lancedb/index.html)を使って，全文検索，ベクトル検索，ハイブリッド検索を試します。

[LanceDBのPythonSDKで全文検索，ベクトル検索，ハイブリッド検索をする](../20250808-lancedb-python-search/)のRust版です。

公式ドキュメントにRustのサンプルは少ないですが，GitHubリポジトリの[examples](https://github.com/lancedb/lancedb/tree/main/rust/lancedb/examples)があります。

## 準備

Cargo.tomlに依存関係を追加します。`arrow-array`のバージョンは，LanceDBが依存しているバージョンに合わせます。また，embeddingにSentence Transformersを使用するために，`lancedb`の`sentence-transformers` featureを有効にします。

```toml
[dependencies]
arrow-array = "55.2.0"
arrow-schema = "55.2.0"
futures = "0.3.31"
lance-index = "0.32.0"
lancedb = { version = "0.21.2", features = ["sentence-transformers"] }
tokio = { version = "1", features = ["full"] }
```

テーブルを作成して，初期データを投入します。テーブルスキーマは，Apache Arrow Schemaで定義します。

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // データベースに接続（または新規作成）
    let uri = "data/testdb";
    let db = connect(uri).execute().await?;

    // sentence-transformersモデルをデータベースに登録
    let embedding = Arc::new(SentenceTransformersEmbeddings::builder().build()?);
    db.embedding_registry()
        .register("sentence-transformers", embedding.clone())?;

    // テーブルスキーマ定義
    let schema = Arc::new(Schema::new(vec![Field::new(
        "movie",
        DataType::Utf8,
        false,
    )]));

    // 初期データ
    let data = StringArray::from_iter_values(vec![
        "The Godfather is Francis Ford Coppola's epic crime drama often cited as one of the greatest films ever made.",
        "The Shawshank Redemption is a powerful and uplifting prison story that consistently tops audience polls.",
        "Citizen Kane, directed by Orson Welles, is celebrated for its revolutionary cinematic techniques.",
        "Pulp Fiction is Quentin Tarantino's non-linear crime film and a landmark of independent cinema.",
        "2001: A Space Odyssey is Stanley Kubrick's visually stunning and philosophical science fiction epic.",
        "The Dark Knight, directed by Christopher Nolan, is hailed for its dark themes and iconic villain.",
        "Schindler's List is Steven Spielberg's poignant and powerful historical drama about the Holocaust.",
        "Seven Samurai is Akira Kurosawa's influential epic that set the template for the modern action film.",
        "Casablanca is a timeless romantic drama known for its iconic lines and performances.",
        "Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.",
    ]);
    let batch = RecordBatchIterator::new(
        RecordBatch::try_new(schema.clone(), vec![Arc::new(data)])
            .into_iter()
            .map(Ok),
        schema.clone(),
    );

    // "movie"カラムに対してembeddingを定義。embeddingの格納先は"embeddings"カラム
    let embdef = EmbeddingDefinition::new("movie", "sentence-transformers", Some("embeddings"));
    // テーブル作成
    let table = db
        .create_table("best_movies", batch)
        .add_embedding(embdef)?
        .execute()
        .await?;

    Ok(())
}
```

## 全文検索用のインデックス作成

`Table.create_index`メソッドで，`movie`カラムに対して全文検索インデックスを作成します。`index`引数に`Index::FTS`を指定すると全文検索インデックスが作成されます。

```rust
    table
        .create_index(&["movie"], Index::FTS(FtsIndexBuilder::default()))
        .execute()
        .await?;
```

## (オプショナル) ANNインデックス作成

`Table.create_index`メソッドで，`embeddings`カラムに対してANNインデックスを作成します。`index`引数に`Index::IvfFlat`を指定するとInverted File Flatインデックスが作成されます。ここでは，ドキュメント数が少ないのでコメントアウト。

```rust
//    table
//        .create_index(
//            &["embeddings"],
//            Index::IvfFlat(IvfFlatIndexBuilder::default()),
//        )
//        .execute()
//        .await?;
```

## 検索する

`Table.query`メソッドを使用して検索を行います。結果は，Arrowの`RecordBatch`で返されます。

### 全文検索

`Query.full_text_search`メソッドに全文検索クエリ`FullTextSearchQuery`を指定します。

```rust
    println!("Full-text search results:");
    let fts_query = FullTextSearchQuery::new("history".to_string());
    let results = table
        .query()
        .full_text_search(fts_query.clone())
        .limit(3)
        .execute()
        .await?
        .try_collect::<Vec<_>>()
        .await?;
    for batch in results {
        let column = batch
            .column_by_name("movie")
            .unwrap()
            .as_any()
            .downcast_ref::<StringArray>()
            .unwrap();
        for v in column.iter() {
            println!("{}", v.unwrap());
        }
    }
```

```
Full-text search results:
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
```

"history"という検索キーワードを含む映画だけがヒットする。

### ベクトル検索

`Query.nearest_to`メソッドにクエリembeddingを指定するとベクトル検索になります。Python版の場合，クエリembeddingは，カラムに紐づけたembedding functionで暗黙的に計算されますが，Rust版だと自分で計算する必要があります。

```rust
    println!("\nVector search results:");
    // クエリembeddingの計算
    let vector_query = embedding
        .compute_query_embeddings(Arc::new(StringArray::from_iter_values(once("history"))))?;

    let results = table
        .query()
        .nearest_to(vector_query.clone())?
        .limit(3)
        .execute()
        .await?
        .try_collect::<Vec<_>>()
        .await?;
    for batch in results {
        let column = batch
            .column_by_name("movie")
            .unwrap()
            .as_any()
            .downcast_ref::<StringArray>()
            .unwrap();
        for v in column.iter() {
            println!("{}", v.unwrap());
        }
    }
```

```
Vector search results:
Casablanca is a timeless romantic drama known for its iconic lines and performances.
The Dark Knight, directed by Christopher Nolan, is hailed for its dark themes and iconic villain.
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
```

ふんわり，"history"に関連しそうな映画がヒットする。なぜか，[Python版](https://blog.mocobeta.dev/posts/20250808-lancedb-python-search/#bekutorujian-suo)と検索結果が違う・・・。

### ハイブリッド検索

ハイブリッド検索のサンプルはドキュメントにもAPIリファレンスにも見つからなかったので，ソースコードを見ながら実装してみました。

`Query.nearest_to`メソッドと`Query.full_text_search`メソッドでそれぞれクエリembeddingと全文検索クエリを渡して，`execute`の代わりに`execute_hybrid`メソッドを呼び出すことでハイブリッド検索ができるようです。APIが少し落ち着かない感じ。

```rust
    println!("\nHybrid search results:");
    let results = table
        .query()
        .nearest_to(vector_query)?
        .full_text_search(fts_query)
        .limit(3)
        .execute_hybrid(QueryExecutionOptions::default())
        .await?
        .try_collect::<Vec<_>>()
        .await?;
    for batch in results {
        let column = batch
            .column_by_name("movie")
            .unwrap()
            .as_any()
            .downcast_ref::<StringArray>()
            .unwrap();
        for v in column.iter() {
            println!("{}", v.unwrap());
        }
    }
```

```
Hybrid search results:
Parasite is Bong Joon-ho's satirical thriller that made history at the Academy Awards.
Casablanca is a timeless romantic drama known for its iconic lines and performances.
The Dark Knight, directed by Christopher Nolan, is hailed for its dark themes and iconic villain.
```

返ってくる結果セットはベクトル検索と一緒だけど，"history"という検索キーワードを含む映画が1位にヒットしている。