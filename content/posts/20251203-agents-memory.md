+++
title = "SessionでAgentに記憶をもたせる"
date = "2025-12-03"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "ai-agents", "openai"]
+++

[以前の記事](https://blog.mocobeta.dev/posts/20251201-my-first-agent/)で，「エージェントとは、指示とツールで構成された大規模言語モデル（LLM）」という（OpenAIの）定義を紹介しました。「ツール＋LLM」で十分な場合もありますが，とはいえ，なんらかのヒューマンインターフェースを備えた「人の相手ができるAIエージェント」を作ろうとすると，もうひとつ大事な要素があります。「記憶（Memory）」ですね。

## すべてを忘却するエージェント

まずは，一切の記憶を持たないエージェントを作ってみます。

```python
import asyncio
from agents import Agent, Runner, RunResult, TResponseInputItem


async def main() -> None:
    agent = Agent(
        name="My Agent",
        instructions="日本語で簡潔にフレンドリーに答えてください。",
        model="gpt-5-nano",
    )
    input_items: list[TResponseInputItem] = []

    while True:
        # 会話のループ
        try:
            user_input = input(">> ")
        except (EOFError, KeyboardInterrupt):
            print()
            break
        if not user_input:
            continue
        
        # エージェントへの入力は，毎回ユーザーの発話のみ
        input_items = [{"role": "user", "content": user_input}]

        result: RunResult = await Runner.run(agent, input=input_items)
        if result.final_output:
            print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

動かしてみると，会話が成り立たず，Agentそのものは記憶を持たないワンショットのLLM呼び出しであることがわかります。

```
>> もうすぐクリスマスだね
うん、もうすぐだね！どんな予定？家でのんびり？外でイルミネーション巡り？クリスマスディナーやプレゼント選び、何か手伝えることがあれば教えてね。
>> 家で料理を作るなら何がいいかな
いいね！家でのごはん、手軽で栄養バランスが取れるのが続くポイントだよ。今夜の候補を3つ挙げるね。材料と時間があれば教えてね。

1) 鶏の照り焼き丼（約20分）
- ひと口メモ: 鶏ももを焼いて、しょうゆ・みりん・砂糖でたれを絡めるだけ。ご飯と together。
- ポイント: 仕上げにねぎを少しのせると香りが出る。

2) 野菜たっぷりの豚バラ炒め定食（約20–25分）
- ひと口メモ: 野菜を先に強火で炒め、豚肉を戻してたれで味を整える。あとでご飯にのせても良い。
- ポイント: 末尾にしょうがを少し入れると風味アップ。

3) 麻婆豆腐風ご飯（約15–20分）
- ひと口メモ: 豆腐を崩さずやさしく煮て、市販の麻婆の素を使えば時短楽チン。
- ポイント: 花椒やラー油を好みで調整して辛さを調整。

もし材料や時間の希望があれば教えて。和洋中のどれがいいか、アレルギーや好きな食材を踏まえて、ぴったりのレシピをもう少し絞るよ。
```

## エージェントに記憶をもたせる

記憶をもたせるナイーブなやり方は，過去の会話履歴（ユーザーの発話とアシスタントの応答）をすべてエージェントへの入力に含める方法です。

```python
# 03-memory.py
import asyncio
from agents import Agent, Runner, RunResult, TResponseInputItem


async def main() -> None:
    agent = Agent(
        name="My Agent",
        instructions="日本語で簡潔にフレンドリーに答えてください。",
        model="gpt-5-nano",
    )
    input_items: list[TResponseInputItem] = []

    while True:
        try:
            user_input = input(">> ")
        except (EOFError, KeyboardInterrupt):
            print()
            break
        if not user_input:
            continue
        
        # ユーザーの最新の発話をエージェントへの入力に追加
        input_items.append({"role": "user", "content": user_input})
        result: RunResult = await Runner.run(agent, input=input_items)
        if result.final_output:
            print(result.final_output)
            # エージェントの応答もエージェントへの入力に追加
            input_items.append({"role": "assistant", "content": result.final_output})


if __name__ == "__main__":
    asyncio.run(main())
```

これで，会話が成り立つようになります。

```
$ uv run ./03-memory.py
>> もうすぐクリスマスだね
ほんとだね、もうすぐクリスマス。何か予定ある？楽しい時間を過ごせるといいね🎄
>> 家で料理作るなら何がいいかな
いいね！家でのクリスマスディナーの候補を3つ紹介します。好みや時間で選んでみてね。

1) 簡単＆華やかプラン（約60分）
- 主菜: 鶏もも肉のオーブン焼き ハーブとレモン
- 副菜: ローストポテトと季節野菜
- デザート: 市販のケーキを温めてアイス添え or ブッシュ・ド・ノエル風デザート
- ポイント: 下味を15–30分置くだけで香り良く仕上がる。オーブン任せで楽ちん。

2) 2〜3コース本格プラン（約90–120分）
- 前菜: カプレーゼ or ほうれん草とベーコンのキッシュ
- 主菜: ローストビーフ or 魚のムニエル 白ワインソース
- サイド: じゃがいものクリームグラタン or マッシュポテト、グリル野菜
- デザート: ブッシュ・ド・ノエル or 市販のクリスマスケーキ
- ポイント: 前菜は前日までOK。ソースは別鍋で温めると時短に。

3) 伝統的に豪華プラン（約120分前後）
- 主菜: ローストチキン or ローストターキー風の鶏肉
- 副菜: マッシュポテト、グラタン、芽キャベツのソテー
- デザート: ブッシュ・ド・ノエル or クリスマスケーキ
- ポイント: オーブンの温度を合わせて同時進行。準備を分担すると楽。

もしよければ、:
- 人数は何人？ 
- オーブン・コンロの状況は？（オーブン1つでOKか、同時進行できるか）
- 和食寄り/洋食寄り、アレルギーや避けたい食材は？
- 今ある材料や予算感

の情報を教えて。希望に合わせて、買い物リストと簡単レシピを具体的に作るよ。
```

## Sessionを使って会話履歴を管理する

ナイーブなやり方だと，デモとしては十分ですが，さまざまな実用上の懸念が浮かびます。プロセスが落ちたら記憶喪失になるとか，履歴が長くなるとコンテキストウィンドウを圧迫しそうとか。なので，会話履歴を管理するための機構が必要となります。

会話履歴の管理機能を自作して，自力でこういった懸念を解決することもできますが，Agents SDKには，[Session](https://openai.github.io/openai-agents-python/sessions/#memory-operations)という会話履歴のストレージ＆管理機能があります。

off-the-shelfで用意されているSessionがいくつかありますが，さくっと試せるのはSQLiteをバックエンドに使う `SQLiteSession` です。以下のコードは，先ほどのエージェントに `SQLiteSession` を組み込んだ例です。

```python
# 03-session.py
import asyncio
from agents import Agent, Runner, RunResult, SQLiteSession


async def main() -> None:
    agent = Agent(
        name="My Agent",
        instructions="日本語で簡潔にフレンドリーに答えてください。",
        model="gpt-5-nano",
    )

    # "conversation_1" というIDの会話履歴を作成（または取得）する
    # SQLiteファイルは "agent_memory.db" に保存される
    session = SQLiteSession(session_id="conversation_1", db_path="agent_memory.db")
    while True:
        try:
            user_input = input(">> ")
        except (EOFError, KeyboardInterrupt):
            print()
            break
        if not user_input:
            continue
        
        # エージェントを実行するときに session を渡す
        # 自前での履歴管理は不要
        result: RunResult = await Runner.run(agent, input=user_input, session=session)
        if result.final_output:
            print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

これで，`conversation_1` というIDの会話履歴が `agent_memory.db` というSQLiteファイルに保存されます。永続化しているので，会話を中断（プロセスを終了）して再度起動すると，以前の会話履歴が復元されて会話が続けられます。

初回の会話

```
$ uv run ./03-session.py
>> もうすぐクリスマスだね
ほんとにもうすぐだね！何か予定はある？プレゼント選びやデコレーション、クリスマスのレシピや映画のおすすめもあるよ。
>> 家で料理作るなら何がいいかな
いいね！家で作るなら、こんなラインナップがおすすめだよ。人数・好み・器具を教えてね。

1) 手軽にクリスマス風3品セット（約30分）
- 鶏もも肉のオーブン焼き（ローズマリーとにんにく）
- にんじんとじゃがいものロースト
- デザートは温かいベリーソースをかけたアイス

2) しっかり派の本格派セット（約60分）
- ローストチキンまたはサーモンのハーブ焼き
- ロースト野菜とマッシュポテト
- 簡単デザート：りんごのクラフティ or カスタードプリン

3) 和風・体に優しいセット（約40分）
- 鰤の照り焼き or 鯖の味噌煮
- きのことほうれん草の和風スープ
- 柚子シャーベットやみかんのシロップ煮

もしよければ、人数・予算・好き嫌い・オーブンの有無を教えて。あなたにぴったりの買い物リストと作り方も用意するよ。
```

`Ctrl+D` でプロセス終了して，再度起動。

```
$ uv run ./03-session.py
>> プレゼントに添えるメッセージカードの案を考えてほしい
いいね！すぐ使えるカード文をいくつか用意したよ。相手との関係やトーンで選んで、名前や贈り物の名前を入れてアレンジしてね。

- 友達・カジュアル
  1) いつもありがとう！素敵なクリスマスを過ごしてね。来年も一緒に笑おう！
  2) サプライズはこれだけど、気持ちは大きさは無限大。楽しいクリスマスを！

- 恋人・パートナー
  1) 君と過ごすクリスマスが今年一番のプレゼント。ずっとそばにいてね。愛してる。
  2) メリークリスマス。いつもありがとう、これからもよろしくね。

- 家族
  1) 家族みんなで過ごせる幸せ。素敵なクリスマスを一緒に過ごそうね。
  2) いつも支えてくれてありがとう。良いクリスマスと新しい年を迎えよう。

- 仕事・同僚・上司
  1) 今年もお世話になりました。素敵なクリスマスとよいお年を。来年もよろしくお願いします。
  2) 感謝をこめて。良いクリスマスと充実した新年をお迎えください。

- 子ども向け
  1) サンタさんへ。これからも楽しく過ごしてね。素敵なクリスマスを！

- 万能・短い
  1) 心をこめて。素敵なクリスマスを。
  2) Merry Christmas（日本語と英語を混ぜても可）。素敵な時間をどうぞ。
```

Sessionは，会話履歴を自動保存する他に，履歴から不要な部分を削除したり，また会話以外の情報（アイテム）を保存したりする管理機能を備えています。また，バックエンドとしてはさまざまなRDBの他，カスタム実装もできるようです。

今日はここまで。Sessionデータベースの中身やSession管理方法についてはまた今度追ってみようと思います。

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の3日目の記事です。