+++
title = "Agents SDKのトークン使用量トラッキング"
date = "2025-12-15"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "agents", "openai"]
+++

`Runner.run()`メソッドの返す`RunResult`または`RunResultStreaming`オブジェクトから，エージェントのトークン使用量が取れます。

[Usage](https://openai.github.io/openai-agents-python/usage/) ドキュメントから引用。

```python
result = await Runner.run(agent, "What's the weather in Tokyo?")
usage = result.context_wrapper.usage

print("Requests:", usage.requests)
print("Input tokens:", usage.input_tokens)
print("Output tokens:", usage.output_tokens)
print("Total tokens:", usage.total_tokens)
```

## エージェントのトークン使用量を構造化ログで記録する

`result.context_wrapper.usage` の中身は [`RequestUsage`](https://openai.github.io/openai-agents-python/ref/usage/) というdataclassになっています。これをたとえば[`structlog`](https://www.structlog.org/en/stable/)でJSON化してさくっとログに流しておくと，後でログ分析やモニタリングツールを使って使用状況の確認や監視がしやすそうです。

```python
# 15-usage-log.py

# JSON形式で構造化ログを出力するための設定
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(indent=2),
    ]
)

logger = structlog.get_logger()


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

        input_items.append({"role": "user", "content": user_input})
        result: RunResult = await Runner.run(agent, input=input_items)
        if result.final_output:
            print(result.final_output)
            input_items.append({"role": "assistant", "content": result.final_output})
        
        # usage log を出力
        usage = result.context_wrapper.usage
        logger.info("Usage log", usage=asdict(usage))


if __name__ == "__main__":
    asyncio.run(main())
```

## ログ出力例

```
$ uv run python ./15-usage-log.py
>> 眠いです
眠いのつらいよね。今の状況はどう？  
- 眠れる場所があるなら10〜20分の仮眠がおすすめ。  
- 難しければ、軽いストレッチと深呼吸、日光を浴びるのも効果的。  
- カフェインは過剰に摂らず、眠る時間帯にも注意してね。  

続けて話す？今の気分や状況を教えて。
{
  "usage": {
    "requests": 1,
    "input_tokens": 28,
    "input_tokens_details": "InputTokensDetails(cached_tokens=0)",
    "output_tokens": 946,
    "output_tokens_details": "OutputTokensDetails(reasoning_tokens=832)",
    "total_tokens": 974,
    "request_usage_entries": [
      {
        "input_tokens": 28,
        "output_tokens": 946,
        "total_tokens": 974,
        "input_tokens_details": "InputTokensDetails(cached_tokens=0)",
        "output_tokens_details": "OutputTokensDetails(reasoning_tokens=832)"
      }
    ]
  },
  "event": "Usage log",
  "level": "info",
  "timestamp": "2025-12-15T14:46:59.830849Z"
}
>> コーヒーいれてください
ごめん、私が実際にコーヒーを入れることはできないんだけど、今すぐ飲める作り方を案内するね。

- インスタントコーヒー
  - カップにコーヒー粉1〜2杯分を入れる
  - 熱湯150mlを注いでよく混ぜる
  - 牛乳や砂糖はお好みで

- ドリップ/フィルター式
  - 中挽きのコーヒー粉約1杯分（7–8g）をフィルターに
  - 200mlのお湯をゆっくり注ぎ、2–3分蒸らしてから抽出

- フレンチプレス
  - 粗挽き8–10gを入れ、熱湯約250mlを注いで4分
  - フレンチプレスをゆっくり押し下げる

手元にどの道具がある？それを教えてくれれば、詳しい手順をさらに簡単にするよ。濃さの好みも教えてね。
{
  "usage": {
    "requests": 1,
    "input_tokens": 153,
    "input_tokens_details": "InputTokensDetails(cached_tokens=0)",
    "output_tokens": 2056,
    "output_tokens_details": "OutputTokensDetails(reasoning_tokens=1792)",
    "total_tokens": 2209,
    "request_usage_entries": [
      {
        "input_tokens": 153,
        "output_tokens": 2056,
        "total_tokens": 2209,
        "input_tokens_details": "InputTokensDetails(cached_tokens=0)",
        "output_tokens_details": "OutputTokensDetails(reasoning_tokens=1792)"
      }
    ]
  },
  "event": "Usage log",
  "level": "info",
  "timestamp": "2025-12-15T14:47:55.068835Z"
}
```

エージェントの出力とログが混在していて見づらいですが，1度のエージェント実行ごとにJSONフォーマットのログが出力されて， `usage` プロパティに詳細なトークン使用量が記録されているのがわかります。（この例ではinputには以前の会話のコンテキストがすべて含まれるので，`input_tokens`は会話が続くと増え続けます。）

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の15日目の記事です。