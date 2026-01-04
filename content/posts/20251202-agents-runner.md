+++
title = "Agents SDKのRunnerとStreaming出力"
date = "2025-12-02"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "ai-agents", "openai"]
+++

# Agentを実行する

[昨日のエントリ](https://blog.mocobeta.dev/posts/20251201-my-first-agent/)では，エージェント実行してLLMのアウトプットを出力する部分をREPL utility (`run_demo_loop`)に任せていましたが，今回はその部分を自分で書いてみます。

## ドキュメントなど

[Running Agents](https://openai.github.io/openai-agents-python/running_agents/)のドキュメントにあるように，エージェントを実行するには`Runner.run()`, `Runner.run_sync()`, `Runner.run_streamed()`を使います。`run()`は最終的なLLMのアウトプットだけ受け取るメソッド，`run_sync()`はその同期版で，`run_streamed()`は，ツール呼び出しなど途中のイベントもストリーミング（Pythonのジェネレータ）で受け取るメソッドです。実用上は途中経過も取りたいケースがほとんどだと思うので，使うのは`run_streamed()`になるでしょう。

`run_streamed()` が返すのは `RunResultStreaming` というオブジェクトで，このオブジェクトの [stream_events()](https://openai.github.io/openai-agents-python/ref/result/#agents.result.RunResultStreaming.stream_events) メソッドがイベントのジェネレータを返します。ループを回してイベントを受け取り，イベントの種類に応じた処理を行う流れ。

どんなイベントがあるか，またイベントオブジェクトがどんなデータを持っているかは，ドキュメントを参照しつつ，とはいえ詳細はソースコードを見ながら＆デバッグしながら取れる情報を確認していく感じになります。ドキュメントが追いついていない点も多く，どこまでがイベントの仕様でどこからが実装の詳細なのかはよくわからない。イベント周りは変更も頻繁そうなので，SDKのアップグレードのたびに影響を確認しないといけなそう。SDKのコードはOSSだから自己責任でどうぞってことでしょうか。

## RunnerとRunResultStreamingのコード

具体的な実装は， `repl.py` の `run_demo_loop()` のコードを参考にします。

[src/agents/repl.py](https://github.com/openai/openai-agents-python/blob/7a14b4b65b3e9ee4b208afa0b09aa91c240170e5/src/agents/repl.py)

標準入力からユーザーの質問を受け取って，エージェントを1回だけ実行して，イベントをループ処理するコードはこう書けます。

```python
import asyncio
from agents import Agent, function_tool, Runner, RunResultStreaming, TResponseInputItem, RawResponsesStreamEvent, RunItemStreamEvent, AgentUpdatedStreamEvent
from openai.types.responses.response_text_delta_event import ResponseTextDeltaEvent
import subprocess


@function_tool
def run_man_command(cmd: str) -> str:
    """man コマンドを実行するツール"""
    if subprocess.run(["which", cmd], capture_output=True).returncode == 1:
        return f"{cmd} コマンドは存在しません。"
    res = subprocess.run(["man", cmd], capture_output=True, text=True).stdout
    if not res:
        return f"{cmd} コマンドのマニュアルは見つかりません。"
    return res


INSTRUCTIONS = """
あなたはLinuxコマンドラインインターフェースの専門家です。Linuxコマンドの使い方を教えることができます。
ユーザーからLinuxコマンドの使い方について質問されたら、`run_man_command`ツールを使ってmanページを参照し、適切な回答を提供してください。

以下の制約条件を守ってください：
- 回答は日本語で行ってください。
- コマンド例を示してください。
- markdown形式ではなく、プレーンテキストで回答してください。
- 回答は簡潔にまとめてください。
- ツールの用途外のことを聞かれた場合は答えないでください。
"""


async def main() -> None:
    agent = Agent(
        name="Linux command helper", instructions=INSTRUCTIONS, model="gpt-5-nano", tools=[run_man_command]
    )
    try:
        user_input = input(">> Linuxコマンドについて何でも聞いてください。\n")
    except (EOFError, KeyboardInterrupt):
        print()
        return

    # 入力アイテムのリスト
    input_items: list[TResponseInputItem]  # type hint 必須
    input_items = [{"role": "user", "content": user_input}]

    # エージェントの実行（ストリーミング）
    result: RunResultStreaming = Runner.run_streamed(agent, input=input_items)
    async for event in result.stream_events():
        # ループ本体
        # イベントの種類に応じて処理を分岐
        if isinstance(event, RawResponsesStreamEvent):
            if isinstance(event.data, ResponseTextDeltaEvent):
                print(event.data.delta, end="", flush=True)
        elif isinstance(event, RunItemStreamEvent):
            if event.item.type == "tool_call_item":
                print(f"\n[Tool called] {event.item.raw_item.name}", flush=True)
            elif event.item.type == "tool_call_output_item":
                print(f"\n[Tool output received]", flush=True)
            else:
                pass
        elif isinstance(event, AgentUpdatedStreamEvent):
            print(f"\n[Agent updated] {event.new_agent.name}", flush=True)
    print()


if __name__ == "__main__":
    asyncio.run(main())
```

実行すると，①Agentが起動されて，②ツールの呼び出しが行われて，③ツールのアウトプットを受け取って，④最終的な回答を返す，までの流れが確認できます。

```bash
$ uv run python ./02-runner-streamed.py
>> Linuxコマンドについて何でも聞いてください。
findでPNGファイルだけリストする

[Agent updated] Linux command helper

[Tool called] run_man_command

[Tool output received]
以下のコマンドでPNGファイルだけを再帰的にリストできます。

- 基本形（現在のディレクトリ以下を検索）
  find . -type f -iname "*.png"

- 出力をファイル名だけにしたい場合
  find . -type f -iname "*.png" -printf "%f\n"

- 特定のディレクトリから検索したい場合
  find /path/to/dir -type f -iname "*.png"

- 出力をNUL区切りで扱いやすくしたい場合（xargs 等と組み合わせると安全）
  find . -type f -iname "*.png" -print0

ポイント
- -type f はファイルのみを対象にする
- -iname は大文字小文字を区別せずにマッチする
- -print は通常省略しても動作しますが、明示することもできます
```

このプログラムは答えを返すと終了します。対話エージェントを作るには，whileループで囲みつつ，これまで会話した内容を覚えておくためのメモリー処理が必要になります。メモリーの扱いについてはまた別途。

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の2日目の記事です。