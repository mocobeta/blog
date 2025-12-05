+++
title = "Hello Agents! - OpenAI Agents SDK"
date = "2025-12-01"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "agents", "openai"]
+++

最近仕事で，AIエージェントの開発に関わっています。現職（現配属チーム）では，エージェントフレームワークとしてOpenAI AgentKitを使っています。

AgentKitはそれ自体が具体的なツールというわけではなく，[ここ](https://platform.openai.com/docs/guides/agents/agent-builder#agentkit)でリストアップされているように，多数のフレームワーク／ツールキットからなるツールキット群で，必要なツールを選んで組み合わせて使う感じです。たとえば，私が触っているのはこのあたりのツール群です。

- [OpenAI Agents Python SDK](https://openai.github.io/openai-agents-python/) と [Responses API](https://platform.openai.com/docs/api-reference/responses)
- [Files API](https://platform.openai.com/docs/api-reference/files)
- [ChatKit Python SDK](https://openai.github.io/chatkit-python/)
- [Evals](https://platform.openai.com/docs/guides/evals)

# Hello, Agents!

本エントリでは，[Agents SDK (Python)](https://openai.github.io/openai-agents-python/) のドキュメントを参考に，初歩的なエージェントを書いて動作させます。

## ところでエージェントって何なの

マーケティング用語としての「AIエージェント」はさておき，Agents SDKのドキュメントでは，

> Agents are the core building block in your apps. An agent is a large language model (LLM), configured with instructions and tools.

> エージェントはアプリの中核となる構成要素です。エージェントとは、指示とツールで構成された大規模言語モデル（LLM）です。

と説明されています。エンジニアリングの視点から見てメンタルモデルが作りやすく，使いやすい定義だと思います。

OpenAI のフレームワークなので，少なくとも現時点では，「LLM」はOpenAIのモデルに限定されます。また，Agents SDKはResponses APIを抽象化したラッパーであるため，LLMのI/FとしてはResponses APIが前提となります。そのためResponses APIのリファレンスも併せて参照すると捗ります。というよりは，Responses APIを知らないとHello World以上のことはできないため，嫌でも詳しくなれます。

[2025/12/05追記]

[LiteLLM](https://docs.litellm.ai/docs/) を介して，OpenAI以外のモデルを使うことができる，とご指摘をいただきました。ありがとうございます。

> この初日の記事で OpenAI のモデルのみとなっていましたが、実は他のモデルも併用できます（ただし、その場合は handoff したいなら Responses API と混ぜないとか、当然 OpenAI の組み込みツールは使えないなどの制約はあります）
>
> https://openai.github.io/openai-agents-python/ja/models/litellm/

from Kazuhiro Seraさん [https://bsky.app/profile/seratch.bsky.social/post/3m77dr6kamk2x](https://bsky.app/profile/seratch.bsky.social/post/3m77dr6kamk2x)

また，[このドキュメント](https://openai.github.io/openai-agents-python/ja/models/)によると，Responses API以外にChat Completions APIも使えるみたいです。

## Hello Agents!

Agents SDKは，複数のエージェントを協調動作させるマルチエージェントシステムのためのツールキットですが，まずは，ひとつのエージェントだけからなる簡単なアプリを作ります。

以下のコードは`man`コマンドをツールとして使い，日本語でやさしくLinuxコマンドの使い方を教えてくれるエージェントです。

デモ用の[REPL utility](https://openai.github.io/openai-agents-python/repl/)が用意されていて，とりあえず対話的に動かして試行錯誤する時はこれを使うのが便利です。

```python
import asyncio
from agents import Agent, run_demo_loop, function_tool
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
    await run_demo_loop(agent, stream=False)


if __name__ == "__main__":
    asyncio.run(main())
```

`Agent` の初期化コードを見ると，最初に引用した定義どおり，指示（`instructions`）とツール（`tools`）とモデル（`model`）を指定していることがわかります。

`@function_tool` デコレータは，普通のPythonの関数をエージェントが使えるツールとして登録するためのおまじないです。docstringはツールの説明としてエージェントが参照するので，サボらずに書きます。

`export OPENAI_API_KEY="sk-..."` でOpenAI APIキーを環境変数に設定して，実行。

```bash
$ uv run python ./01-getting-started.py
 > tarコマンドでbzip2ファイルを解凍したい
.tar.bz2 を tar で解凍するには -j オプションを使います。

基本例
- 現在のディレクトリに展開: tar -xjf archive.tar.bz2
- 展開時に詳細を表示: tar -xvjf archive.tar.bz2  または tar -xjf archive.tar.bz2 -v
- 特定のディレクトリに展開: tar -xjf archive.tar.bz2 -C /path/to/dir

内容を確認したい場合
- アーカイブの中身を一覧表示: tar -tjf archive.tar.bz2  または tar -tvjf archive.tar.bz2

補足
- .tar.bz2 以外にも .tbz2 など同様に -j で解凍できます。
- 長いオプション版: tar --extract --bzip2 --file archive.tar.bz2

もし他にも知りたい状況（例えば特定ファイルだけ抽出したい等）があれば教えてください。
```

指示に書いたように，プレーンテキストでコマンド例を交えた回答が返ってきました。

LLMの知識の範囲で回答せずに，本当にツールを呼んでるの？というのは，`run_demo_loop()` のオプションで `stream=True` を指定して，tool callのログを出力させると確認できます。

```bash
 > duでホームディレクトリの容量を確認したい

[Agent updated: Linux command helper]

[tool called]

[tool output: DU(1)                                                                                 User Commands                                                                                 DU(1)
...略...
]

以下のコマンドでホームディレクトリの容量を知れます。

- 合計を人間が読みやすい形式で表示
  du -sh ~
  例: du -sh /home/youruser

- ホーム配下のトップレベルの各ディレクトリの容量を知りたい場合
  du -h --max-depth=1 ~ | sort -hr

- 特定のディレクトリだけを合計したい場合（例: /home/youruser）
  du -sh /home/youruser

補足
- -s / --summarize は合計のみを表示、-h / --human-readable は読みやすい単位で表示
- さらに細かい階層を表示したい場合は --max-depth=N を使います（N に階層数を入れる）
- 追加の除外が必要なら --exclude=PATTERN を使えます

この回答は du の基本的な使い方に基づいています。必要なら他のオプション例も示します。
```

## 存在しないコマンドについて聞いてみる

嘘を生成しないか確認するために，存在しないコマンド `hmm` について聞いてみます。

```bash
 > hmmコマンドの使い方

[Agent updated: Linux command helper]

[tool called]

[tool output: hmm コマンドは存在しません。]
hmm コマンドは標準的な Linux には存在しません。現環境で実行しても見つからず、man ページもありません。したがって hmm の「使い方」を直接案内することはできません。

まずは以下を試して、コマンド名が正しいか確認してください。
- which hmm
- type -a hmm
- command -v hmm

もし何かのパッケージの一部として存在する想定なら、パッケージを探してからインストールします。例:
- Debian/Ubuntu 系: apt-cache search hmm または apt-cache pkgnames | grep -i hmm
- Red Hat/Fedora 系: dnf search hmm または yum search hmm

hmm が何をするコマンドなのか教えてください（用途・ソフトウェア名など）。正確な名称や背景が分かれば、具体的な使い方と代替案を案内します。

補足として、一般的なコマンドの使い方の確認方法も併記します。
- ヘルプを表示: コマンド名 --help
- マニュアルを読む: man コマンド名
- 例: あるファイルを検索するコマンドの基本例を示すときは、例の形で書く（例: grep "文字列" file.txt など）
```

### エージェントの用途と関係ないことを聞いてみる

linuxコマンドと関係ない質問をしてみます。指示（instructions）に書いたように，用途から外れた質問には「答えない」ことが確認できます。

```bash
 > 人類が次に月に行くのはいつ

[Agent updated: Linux command helper]
その質問はLinuxコマンドの使い方には関係ないためお答えできません。Linuxコマンドの使い方について質問があれば、具体例とともにお手伝いします（例: ls、grep、cat など）。
```

----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の1日目の記事です。