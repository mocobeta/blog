+++
title = "エージェントで画像やPDFファイルを解析する - Files APIとAgents SDKとResponses API"
date = "2025-12-05"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "agents", "openai"]

[extra]
cover = "cookies1.jpg"
+++

最近のLLMはマルチモーダルが当たり前になりつつありますが，Agents SDK（というよりは，その裏側のResponses API）もファイルを入力として受け取ることができます。

## Agents SDK / Responses APIでファイルを扱う

Responses APIで，プロンプトと一緒に画像ファイルを入力に与えるコード例がこちらのドキュメントにあります：
[Images and vision](https://platform.openai.com/docs/guides/images-vision)

ファイルの与え方には，

1. URLで指定する
2. base64エンコードした文字列を直接与える
3. Files APIでアップロードしたファイルのIDを与える

の3通りがあります。何度も同じファイルを解析したり，大きいファイルを扱う可能性がある場合は，[Files API](https://platform.openai.com/docs/api-reference/files)に事前にファイルをアップロードしておくのが無難かなあ，と思います。

## 画像ファイルをアップロードしてエージェントで解析する

ファイルは一度に複数個渡せるので，エージェントに2枚の画像を解析させてみます。

cookie1.jpg

![cookie1.jpg](./cookies1.jpg)

cookie2.jpg
![cookie2.jpg](./cookies2.jpg)

```python
# 05-image-agent.py
from openai import OpenAI
from agents import Agent, Runner, TResponseInputItem

client = OpenAI()

# アップロードする画像ファイルのパス
image_files = [
    "./images/cookies1.jpg",
    "./images/cookies2.jpg"
]

file_ids = []
for image_file in image_files:
    # 画像ファイルをFiles APIでアップロード
    with open(image_file, "rb") as f:
        result = client.files.create(
            file=f,
            purpose="vision",
        )
        file_ids.append(result.id)

# ファイル情報を取得してアップロードが成功したことを確認
for file_id in file_ids:
    result = client.files.retrieve(file_id)
    print(f"Retrieved file info: {result}")

# 画像解析エージェント
agent = Agent(
    name="Image Analysis Agent",
    instructions="あなたは画像を解析する専門家です。画像についての質問に答えてください。",
    model="gpt-5-mini",
)

# 入力アイテムのリスト。Responses APIへの入力と同じ形式で渡す
# TRsponseInputItem 型はTypedDictになっていて，どんなプロパティが必要はソースコードを参照すると良い
input_items: list[TResponseInputItem] = [
    {
        "role": "user",
        "content": [
            {"type": "input_text", "text": "クッキーは全部で何個写っている？"},
            {
                "type": "input_image",  # 画像ファイルは input_image タイプを使う
                "file_id": file_ids[0],
                "detail": "auto",
            },
            {
                "type": "input_image",
                "file_id": file_ids[1],
                "detail": "auto",
            },
        ],
        "type": "message",
    }
]
result = Runner.run_sync(agent, input=input_items)

print(result.final_output)

# アップロードしたファイルを削除
for file_id in file_ids:
    client.files.delete(file_id)
```

レスポンス

```bash
$ uv run python ./05-image-agent.py
Retrieved file info: FileObject(id='file-6fschKQqi1pfjqMQ2VfVeo', bytes=1638461, created_at=1764943216, filename='cookies1.jpg', object='file', purpose='vision', status='processed', expires_at=None, status_details=None)
Retrieved file info: FileObject(id='file-Sr872vwyDyQQ9udhD3HZB8', bytes=3512374, created_at=1764943221, filename='cookies2.jpg', object='file', purpose='vision', status='processed', expires_at=None, status_details=None)
全部で22個です。ジンジャーブレッドが15個、皿のクッキーが7個で合計22個になります。
```

正解...！

ちなみに，この例では`gpt-5-mini`を使いましたが，モデルを`gpt-5-nano`に変えると，「32個です。」のように数え間違いをしていました。画像やファイルを正しく扱うにはモデル選択に気をつけるのが良さそう。

## PDFファイルをアップロードしてエージェントで解析する

PDFファイルを扱う具体例はドキュメントにはなさそうですが，画像ファイルと同じ要領でいけます。

[レーザーテック株式会社の2026年第1四半期の決算短信PDF](https://ssl4.eir-parts.net/doc/6920/tdnet/2705316/00.pdf)を解析させてみたコード。

```python
# 05-pdf-agent.py
from openai import OpenAI
from agents import Agent, Runner, TResponseInputItem

client = OpenAI()

# アップロードするPDFファイルのパス
pdf_file = "./pdf/kessan_tanshin.pdf"

# PDFファイルをFiles APIでアップロード
with open(pdf_file, "rb") as f:
    result = client.files.create(
        file=f,
        purpose="user_data",
    )
    file_id = result.id

# ファイル情報を取得してアップロードが成功したことを確認
result = client.files.retrieve(file_id)
print(f"Retrieved file info: {result}")

# 画像解析エージェント
agent = Agent(
    name="PDF Analysis Agent",
    instructions="あなたはPDFを解析する専門家です。PDFについての質問に答えてください。",
    model="gpt-5-mini",
)

# 入力アイテムのリスト
# TRsponseInputItem 型はTypedDictになっていて，どんなプロパティが必要はソースコードを参照する
input_items: list[TResponseInputItem] = [
    {
        "role": "user",
        "content": [
            {"type": "input_text", "text": "四半期の売上高と経常利益と前年同四半期からの増減率を抽出して"},
            {
                "type": "input_file",  # 画像以外のファイルは input_file タイプを使う
                "file_id": file_id,
            },
        ],
        "type": "message",
    }
]
result = Runner.run_sync(agent, input=input_items)

print(result.final_output)

# アップロードしたファイルを削除
client.files.delete(file_id)
```

レスポンス

```bash
$ uv run python ./05-pdf-agent.py
Retrieved file info: FileObject(id='file-9CKDUHEaJe9FPnC1aE1PmM', bytes=238611, created_at=1764944203, filename='kessan_tanshin.pdf', object='file', purpose='user_data', status='processed', expires_at=None, status_details=None)
以下を抽出しました（単位：百万円、増減率は前年同四半期比）。

- 2026年6月期 第1四半期（2025/7/1–2025/9/30）
  - 売上高：54,171（前年同四半期比 +47.5%）
  - 経常利益：27,141（前年同四半期比 +113.2%）

- 2025年6月期 第1四半期（2024/7/1–2024/9/30）
  - 売上高：36,737（前年同四半期比 △22.3%）
  - 経常利益：12,728（前年同四半期比 +16.5%）

必要ならCSV形式や差額（百万円・増加倍数）も出力しますか？
```

PDFのレイアウトや作成方法によると思いますが，フォーマットがほぼ決まっている決算短信のようなファイルからの情報抽出は正確にできるようです。

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の5日目の記事です。