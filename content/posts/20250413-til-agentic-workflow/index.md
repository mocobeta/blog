+++
title = "Andrew Ng氏による\"Agentic workflowのためのデザインパターン\"で理解するAI agent"
date = "2025-04-13"

[taxonomies]
categories = ["Posts"]
tags = ["til", "llm", "AI agent"]
+++

## AI agentのためのデザインパターン

流行のAI agentについて，重い腰を上げて学んでいる。断片的な最新情報とバズワードの洪水に圧倒されながら，できるだけ体系的な理解を求めて調べていて，AI agent/Agentic workflowのためのフレームワーク・デザインパターンをAndrew Ng氏が与えているのを見つけた。

- (video) [What's next for AI agentic workflows ft. Andrew Ng of AI Fund](https://www.youtube.com/watch?v=sal78ACtGTc)

スライドの中では，「Agentic Reasong Design Pattern」と紹介されているもので，Agentic workflowは，Reflection, Tool Use, Planning, Multi-agent collabolationの４つのパターン（カテゴリ）に分類されると説明されている。1年前に公開された動画なので，周回遅れかもしれないけれど，2025年現在でも十分にワークする思考のフレームワークだと思う。

[LLMエージェントのデザインパターン、Agentic Design Patternsを理解する (via 株式会社ログラス テックブログ)](https://zenn.dev/loglass/articles/b9ee37737deb85) など，上記動画の日本語による紹介もいくつかある。

なお，界隈ではデファクトになりつつある，Anthropicが主導する[Model Context Protocol(MCP)](https://modelcontextprotocol.io/introduction)は，Tool Useパターンを促進するプロトコル／ツールと整理すると理解しやすい（と思うが，自分の解釈なので誤解している可能性はある）。おそらくAI系アプリケーションでもっとも普及している，シンプルなRAGパターンもまた，代表的なTool Useのひとつと位置づけられる。

Ng先生の説明はアカデミックでハイレベルなので，実装にはエンジニアリングによせた理解が必要で，たとえばこちらのWeaviateのブログが図解つきでわかりやすいように思う（同じ言葉を使っていてもNg先生の整理のしかたとは細かいところで違う部分はある）。

- [What Are Agentic Workflows? Patterns, Use Cases, Examples, and More (via Weaviate blog)](https://weaviate.io/blog/what-are-agentic-workflows)

＃ この記事の最後の"Challenges and Limitations of Agentic Workflows"のセクションは肝に命じておこう。

> - Is the task complex enough to require adaptive decision-making, or would a deterministic approach suffice?
> - Would a simpler AI-assisted tool (such as RAG without an agent) achieve the same outcome?
> - Does the workflow involve uncertainty, changing conditions, or multi-step reasoning that an agent could handle more effectively?
> - What are the risks associated with giving the agent autonomy, and can they be mitigated?

なお"AI agent"や"Agentic workflow"は新興の言葉なので，その定義はさまざまな人の解釈やベンダーの思惑によって揺れている。タイトルに「Andrew Ng氏による」とつけたのは，用語の解釈が今のところ定まってはいないため。

たとえば以下のGoogleによる解説ビデオでは，AI agentとAgentic workflowを対比させて，Agentic workflowは，agentの自律性を抑え，より決定的な振る舞いをすると説明されている（私も最初はこういう対比があると解釈していた）。

- (video) [Agentic AI: Workflows vs. agents](https://www.youtube.com/watch?v=Qd6anWv0mv0)

Ng先生のAgentic workflowの定義には，agentの自律性を制限するというような意味づけはなく，PlanningやMulti-agent collaborationといった高度に自律的なパターンも含まれる。個人的にはNg先生の定義のほうがより包括的で良いかなと感じている。

## AI agent/Agentic workflow開発キット

現時点では，人気という点で[LangGraph](https://github.com/langchain-ai/langgraph)一強という印象。OpenAIも最近[Agents SDK](https://github.com/openai/openai-agents-python)をリリースしていて，数日前にはGoogleが[Agent Development Kit](https://github.com/google/adk-python)を発表して話題になった。なお言語の選択肢は限られていて，メジャーなツールはどれもPythonベース。LangGraphだけJavaScript(TypeScript)もサポートしている。
