+++
title = "ChatKitの会話履歴を覗く"
date = "2025-12-09"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "agents", "openai", "chatkit"]
+++

[ChatKitの会話履歴の構造](https://blog.mocobeta.dev/posts/20251208-chatkit-thread/)の続きです。[ChatKitで添付ファイルをエージェントに渡して解析させる](https://blog.mocobeta.dev/posts/20251207-file-attachment-chatkit/)で作ったデモチャットアプリの会話履歴をSQLiteのデータベースに保存して，どんなデータが保存されるか，中身を覗いてみます。

SQLiteに会話履歴を保存するには，`Store`の実装を[python/samples/demos/chatkit-integration/store.py](https://github.com/microsoft/agent-framework/blob/551219cb559387164d0fbad99d8ebe7664e77471/python/samples/demos/chatkit-integration/store.py)に入れ替えればOKです。

## 会話履歴のテーブルスキーマ

ChatKitで永続化層のスキーマは特に定められていないですが，上記`store.py`の実装だと以下のようなテーブルが作られます。

```sql
sqlite> .schema
CREATE TABLE threads (
                id TEXT PRIMARY KEY,
                user_id TEXT NOT NULL,
                created_at TEXT NOT NULL,
                data TEXT NOT NULL
                );
CREATE TABLE items (
                id TEXT PRIMARY KEY,
                thread_id TEXT NOT NULL,
                user_id TEXT NOT NULL,
                created_at TEXT NOT NULL,
                data TEXT NOT NULL
                );
CREATE TABLE attachments (
                id TEXT PRIMARY KEY,
                user_id TEXT NOT NULL,
                data TEXT NOT NULL
                );
```

各テーブルに`data`というTEXT型のカラムがあり，ここにPydanticの`BaseModel`をJSONシリアライズした文字列が保存されます。（もう少しかっちり管理するなら，モデルクラスの定義に沿って正規化もできそうです。ただ細かく作り込みすぎるとライブラリのバージョンアップ時に困る可能性もありそうなので，ゆるくJSONで持っておくくらいがいいのかもしれません。）

## 会話履歴のデータを見てみる

UIからエージェントに3つの質問（ひとつは画像ファイルつき）を投げて，どんなデータが保存されるか確認しました。

**質問**

- 「ホットウイスキーの作り方」
- 「おつまみも用意して」
- ウイスキーが数本写っている画像をアップロード＋「どれがいい？」

### `threads`テーブル

```sql
sqlite> select * from threads;
id            user_id    created_at                  data                                                        
------------  ---------  --------------------------  ------------------------------------------------------------
thr_59884bea  demo_user  2025-12-09T22:21:12.056309  {"thread":{"title":null,"id":"thr_59884bea","created_at":"20
                                                     25-12-09T22:21:12.056309","status":{"type":"active"},"metada
                                                     ta":{}}}
```

一つの会話（スレッド）に対して一レコードが保存されます。`data`カラムには`ThreadMetadata`モデルをJSONシリアライズしたデータ保存されています。

### `items`テーブル

```sql
sqlite> select * from items;
id                                                      thread_id     user_id    created_at                  data                                                         
------------------------------------------------------  ------------  ---------  --------------------------  -------------------------------------------------------------
msg_3c175243                                            thr_59884bea  demo_user  2025-12-09T22:21:12.097481  {"item":{"id":"msg_3c175243","thread_id":"thr_59884bea","cre 
                                                                                                             ated_at":"2025-12-09T22:21:12.097481","type":"user_message", 
                                                                                                             "content":[{"type":"input_text","text":"ホットウイスキーの作 
                                                                                                             り方"}],"attachments":[],"quoted_text":"","inference_options 
                                                                                                             ":{"tool_choice":null,"model":null}}}                        

wf_895180e7                                             thr_59884bea  demo_user  2025-12-09T22:21:14.205787  {"item":{"id":"wf_895180e7","thread_id":"thr_59884bea","crea 
                                                                                                             ted_at":"2025-12-09T22:21:14.205787","type":"workflow","work 
                                                                                                             flow":{"type":"reasoning","tasks":[],"summary":{"duration":1 
                                                                                                             6},"expanded":false}}}                                       

msg_027a6e2744a9c974006938225a944881a0b6c93db0363c875e  thr_59884bea  demo_user  2025-12-09T22:21:36.880355  {"item":{"id":"msg_027a6e2744a9c974006938225a944881a0b6c93db 
                                                                                                             0363c875e","thread_id":"thr_59884bea","created_at":"2025-12- 
                                                                                                             09T22:21:36.880355","type":"assistant_message","content":[{" 
                                                                                                             annotations":[],"text":"ホットウイスキー（お湯割り）の基本\n 
                                                                                                             \n材料\n- ウイスキー 30–45ml\n- お湯 90–120ml（70〜80℃目安） 
                                                                                                             \n- お好みで：はちみつ小さじ1〜2、レモン（皮or薄切り）、シナ 
                                                                                                             モン、ジンジャー、クローブ\n\n作り方\n1) マグを温めてお湯を捨
                                                                                                             てる。  \n2)（甘くするなら）先にはちみつを入れる。  \n3) お湯
                                                                                                             →ウイスキーの順に注ぎ、軽く混ぜる。  \n4) レモンやスパイスを 
                                                                                                             添えて出来上がり。\n\nコツ\n- 比率はウイスキー:お湯＝1:2〜1: 
                                                                                                             3が飲みやすい。  \n- 沸騰直後は避けて、70〜80℃くらいで香りよ 
                                                                                                             く。  \n- アルコールは直火や電子レンジで直接加熱しない（お湯 
                                                                                                             だけ温めて使う）。  \n- スモーキー系ははちみつ/生姜、バーボン
                                                                                                             はオレンジ皮やシナモン、日本のブレンデッドは柚子皮が相性◎。\ 
                                                                                                             n\nアレンジ（ホットトディ）\n- ウイスキー45ml＋はちみつ小さじ
                                                                                                             1＋レモン果汁10ml＋お湯120ml＋クローブ少々。","type":"output 
                                                                                                             _text"}]}}                                                   

msg_e24c74f3                                            thr_59884bea  demo_user  2025-12-09T22:30:36.510572  {"item":{"id":"msg_e24c74f3","thread_id":"thr_59884bea","cre 
                                                                                                             ated_at":"2025-12-09T22:30:36.510572","type":"user_message", 
                                                                                                             "content":[{"type":"input_text","text":"おつまみも用意して"} 
                                                                                                             ],"attachments":[],"quoted_text":"","inference_options":{"to 
                                                                                                             ol_choice":null,"model":null}}}                              

wf_5dfe6bb3                                             thr_59884bea  demo_user  2025-12-09T22:30:37.995799  {"item":{"id":"wf_5dfe6bb3","thread_id":"thr_59884bea","crea 
                                                                                                             ted_at":"2025-12-09T22:30:37.995799","type":"workflow","work 
                                                                                                             flow":{"type":"reasoning","tasks":[],"summary":{"duration":2 
                                                                                                             3},"expanded":false}}}                                       

msg_0ad21c3bd88db17700693824956d0081a0a9043e0f50e92a68  thr_59884bea  demo_user  2025-12-09T22:31:13.193679  {"item":{"id":"msg_0ad21c3bd88db17700693824956d0081a0a9043e0 
                                                                                                             f50e92a68","thread_id":"thr_59884bea","created_at":"2025-12- 
                                                                                                             09T22:31:13.193679","type":"assistant_message","content":[{" 
                                                                                                             annotations":[],"text":"ホットウイスキーに合うおつまみ案（手 
                                                                                                             軽＆うまい）\n\nすぐ出せる\n- ミックスナッツ（黒胡椒 or 燻製 
                                                                                                             塩をひと振り）\n- チーズ盛り（チェダー/ブルー/カマンベール） 
                                                                                                             ＋はちみつ\n- ドライフルーツ＋ビターチョコ（カカオ70％前後） 
                                                                                                             \n- スモークサーモン or 合鴨スモーク＋レモン\n- オリーブ＆ピ 
                                                                                                             クルス\n- いぶりがっこ×クリームチーズ（のせるだけ）\n\n5〜10 
                                                                                                             分で作れる\n- はちみつバターナッツ：ナッツをバター小さじ1とは
                                                                                                             ちみつ小さじ1で炒め、塩ひとつまみ。\n- クリームチーズの黒胡椒
                                                                                                             はちみつ：クラッカーにのせ、黒胡椒ガリッとはちみつ少々。\n-  
                                                                                                             きのこのバター醤油ソテー：仕上げに黒胡椒とレモン皮すりおろし 
                                                                                                             。\n- ベーコン巻きデーツ（or プルーン）：巻いて焼き、仕上げに
                                                                                                             黒胡椒。\n- 焼きリンゴ（薄切り）：バターで軽く焼き、シナモン 
                                                                                                             少々。\n\n相性の目安\n- スモーキー系ウイスキー：スモークナッ 
                                                                                                             ツ、ブルーチーズ、生姜、燻製おつまみ。\n- バーボンや甘め：焼 
                                                                                                             きリンゴ、キャラメル系、シナモン、ピーカンナッツ。\n- シェリ 
                                                                                                             ー樽：ドライフルーツ、ダークチョコ、ナッツ、ブルーチーズ。\n 
                                                                                                             - 日本のブレンデッド：柚子皮、いぶりがっこ、昆布系おかき。\n 
                                                                                                             \nコツ\n- 一口サイズ＆塩気は控えめに。温かいおつまみを少しず 
                                                                                                             つ出すと香りが引き立ちます。飲みすぎ注意でゆっくりどうぞ。", 
                                                                                                             "type":"output_text"}]}}                                     

msg_31f064ac                                            thr_59884bea  demo_user  2025-12-09T22:40:41.996965  {"item":{"id":"msg_31f064ac","thread_id":"thr_59884bea","cre 
                                                                                                             ated_at":"2025-12-09T22:40:41.996965","type":"user_message", 
                                                                                                             "content":[{"type":"input_text","text":"どれがいい？"}],"att 
                                                                                                             achments":[{"id":"file-8sjtmgkvjCawY4FS8HJNrq","name":"jp_wh 
                                                                                                             isky.jpg","mime_type":"image/jpeg","upload_url":null,"type": 
                                                                                                             "image","preview_url":"https://127.0.0.1/files/file-8sjtmgkv 
                                                                                                             jCawY4FS8HJNrq/preview"}],"quoted_text":"","inference_option 
                                                                                                             s":{"tool_choice":null,"model":null}}}                       

wf_f20d397f                                             thr_59884bea  demo_user  2025-12-09T22:40:45.403890  {"item":{"id":"wf_f20d397f","thread_id":"thr_59884bea","crea 
                                                                                                             ted_at":"2025-12-09T22:40:45.403890","type":"workflow","work 
                                                                                                             flow":{"type":"reasoning","tasks":[],"summary":{"duration":2 
                                                                                                             6},"expanded":false}}}                                       

msg_0fb7ceeceb39108800693826f7d494819d9bcfbc24350dcf42  thr_59884bea  demo_user  2025-12-09T22:41:18.625854  {"item":{"id":"msg_0fb7ceeceb39108800693826f7d494819d9bcfbc2 
                                                                                                             4350dcf42","thread_id":"thr_59884bea","created_at":"2025-12- 
                                                                                                             09T22:41:18.625854","type":"assistant_message","content":[{" 
                                                                                                             annotations":[],"text":"ホットでいくならこの順がおすすめ\n\n 
                                                                                                             - まろやか系：右端のHibiki（ハーモニー系）。はちみつ少々＋レ 
                                                                                                             モン皮で◎\n- フルーティ系：宮城峡（ボトル左から2番目）。ドラ 
                                                                                                             イフルーツ＆ビターチョコと相性よし\n- スモーキー系：余市（い 
                                                                                                             ちばん左）。生姜＋はちみつ、おつまみは燻製ナッツ\n\n注意\n-  
                                                                                                             白州18・山崎・イチローズモルトは香りが繊細で貴重。まずはスト 
                                                                                                             レート or 少量加水で、ホットは別のボトルでどうぞ。\n\n比率は 
                                                                                                             1:2〜1:3、お湯は70〜80℃目安。  \nスモーキーにする？それともフ
                                                                                                             ルーティにする？","type":"output_text"}]}}
```


ユーザーの発話，アシスタントの応答がすべて個別のレコードとして保存されています。`data`カラムには`ThreadItem`モデルをJSONシリアライズしたデータが保存されています。

`ThreadItem`には種別があり，ユーザーメッセージは`user_message`タイプ，アシスタントメッセージは`assistant_message`タイプになっています。画像を添付した時のユーザーメッセージには，`attachments`というプロパティがついているのが見えます。また，よく見るとアシスタントのメッセージの前に，`workflow`というタイプのレコードが挟まっています。これはLLMの推論処理のログのようです。

`ThreadItem`はコード上Union typeになっていて，今回確認できたタイプ以外に，tool useに対応するものなどさまざまな種別の`ThreadItem`がみえます。

```python
# chatkit/types.py より抜粋
# openai-chatkit==1.4.0
ThreadItem = Annotated[
    UserMessageItem
    | AssistantMessageItem
    | ClientToolCallItem
    | WidgetItem
    | WorkflowItem
    | TaskItem
    | HiddenContextItem
    | SDKHiddenContextItem
    | EndOfTurnItem,
    Field(discriminator="type"),
]
"""Union of all thread item variants."""
```

### `attachments`テーブル

```sql
sqlite> select * from attachments;
id                           user_id    data                                                        
---------------------------  ---------  ------------------------------------------------------------
file-8sjtmgkvjCawY4FS8HJNrq  demo_user  {"attachment":{"id":"file-8sjtmgkvjCawY4FS8HJNrq","name":"jp
                                        _whisky.jpg","mime_type":"image/jpeg","upload_url":null,"typ
                                        e":"image","preview_url":"https://127.0.0.1/files/file-8sjtm
                                        gkvjCawY4FS8HJNrq/preview"}}
```

画像ファイルを一つアップロードしたので，それに対応するレコードが作成されています。`data`カラムには`Attachment`モデルをJSONシリアライズしたデータが保存されています。

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の9日目の記事です。