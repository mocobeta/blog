+++
title = "Agents SDKはどうやってPython関数（Function tool）をLLMのtoolにしているのか"
date = "2025-12-18"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "ai-agents", "openai"]
+++

Agents SDKでは，[`@function_tool`デコレータ](https://openai.github.io/openai-agents-python/tools/#function-tools)を普通のPython関数に付与すると，[Responses APIのFunction](https://platform.openai.com/docs/guides/function-calling?api-mode=responses)としてシームレスに登録できます。このマジックがどう実装されているのか，ソースコードを覗いてみました。

[Automatic argument and docstring parsing](https://openai.github.io/openai-agents-python/tools/#automatic-argument-and-docstring-parsing)で詳細に説明されていますが，`@function_tool`デコレータを付与すると，デコレータの実行時にまず

- 関数のシグネチャ（関数名，引数，型アノテーション）
- 関数のdocstring

を解析して，[`FuncSchema`](https://openai.github.io/openai-agents-python/ref/function_schema/#agents.function_schema)オブジェクトが作られ，その情報をもとに[`FunctionTool`](https://openai.github.io/openai-agents-python/ref/tool/#agents.tool.FunctionTool)オブジェクトが作られます。そしてResponses APIの呼び出し時に，`FunctionTool`オブジェクトから`tools`パラメータが導出されるという流れ。

## Function toolの実装を覗く

`FuncSchema`のコードは[src/agents/function_schema.py](https://github.com/openai/openai-agents-python/blob/main/src/agents/function_schema.py)で，inspectionを駆使して関数シグネチャ，docstringを抽出してResponses APIで使いやすいモデルに変換しています。マジックの主要な部分はだいたいここで行われていているようです。普段こういうメタなコードをあまり書かないので勉強になるし面白い。

`FuncSchema`から`FunctionTool`を作っているのは[function_toolデコレータ](https://github.com/openai/openai-agents-python/blob/56680f43d636fec36ae2ccb6dab681a813f2f5cf/src/agents/tool.py#L710)。

さらに，`FunctionTool`をResponses APIの`tools`パラメータに変換しているのは[このあたり](https://github.com/openai/openai-agents-python/blob/56680f43d636fec36ae2ccb6dab681a813f2f5cf/src/agents/models/openai_responses.py#L463)でした。


----

これは [Agents SDK+αのTipsを一人で書いていくアドカレ Advent Calendar 2025](https://adventar.org/calendars/12523)の18日目の記事です。（Tipsではなくなってきた...）