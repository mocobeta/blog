+++
title = "[embcli開発日誌] ドキュメンテーションサイトとllama.cppサポートを追加"
date = "2025-05-25"
description = ""

[taxonomies]
categories = ["Short Posts"]
tags = ["embcli", "python", "oss", "embedding"]
+++

[embeddingのためのOSSコマンドラインツールを作っている](https://blog.mocobeta.dev/posts/20250518-entry-cli-for-embedding/)で紹介した `embcli` というツールのアップデート2件です。

## ドキュメンテーションサイトのリリース

ドキュメンテーションサイトができました!!

[embcli - CLI for Embedding](https://embcli.mocobeta.dev/)

- [Quick Start](https://embcli.mocobeta.dev/##quick-start)
- [Basic Command Usage](https://embcli.mocobeta.dev/#basic_usage/)
- [Vector Search Usage](https://embcli.mocobeta.dev/#vector_search/)
- [Model Plugins](https://embcli.mocobeta.dev/#model_plugins/)

余談ですが，GitHub Copilot(GPT-4o Copilot)で補完しながら書いていました。
数単語程度入力すると，2~3センテンス先までは高い確度で当ててくる印象で，特にnon-native English speakerにとっては助かります。
（補完ではなく，「Clineに全部書かせる」案も試したけれど，違うそうじゃない，となることが多すぎたたため，こちらは不採用。）

## llama.cppサポートのリリース

[embcli-llamacpp](https://pypi.org/project/embcli-llamacpp/) プラグインをリリースしました。

GGUFフォーマットに変換したTransformerモデルがロードできるようになります。

インストール:

```bash
pip install embcli-llamacpp
```

使い方:

```bash
# list all available models.
emb models
LlamaCppModel
    Vendor: llama-cpp
    Models:
    * llama-cpp (aliases: llamacpp)
    Model Options:

# assume you have a GGUF converted all-MiniLM-L6-v2 model in the current directory.
# get an embedding for an input text by running the GGUF converted model.
emb embed -m llamacpp -p ./all-MiniLM-L6-v2.F16.gguf "Embeddings are essential for semantic search and RAG apps."

# calculate similarity score between two texts by GGUF converted all-MiniLM-L6-v2 model. the default metric is cosine similarity.
emb simscore -m llamacpp -p ./all-MiniLM-L6-v2.F16.gguf "The cat drifts toward sleep." "Sleep dances in the cat's eyes."
0.8031107247483075

# index example documents in the current directory.
emb ingest-sample -m llamacpp -p ./all-MiniLM-L6-v2.F16.gguf -c catcafe --corpus cat-names-en

# or, you can give the path to your documents.
# the documents should be in a CSV file with two columns: id and text. the separator should be comma.
emb ingest -m llamacpp -p ./all-MiniLM-L6-v2.F16.gguf -c catcafe -f <path-to-your-documents>

# search for a query in the indexed documents.
emb search -m llamacpp -p ./all-MiniLM-L6-v2.F16.gguf -c catcafe -q "Who's the naughtiest one?"
Found 5 results:
Score: 0.03687787040089618, Document ID: 25, Text: Nala: Nala is a graceful and queenly cat, often a beautiful cream or light tan color. She moves with quiet dignity and observes her surroundings with intelligent eyes. Nala is affectionate but discerning, choosing her moments for cuddles, and her loyalty to her family is unwavering, a truly regal companion.
Score: 0.036599260961425885, Document ID: 73, Text: Cody: Cody is a ruggedly handsome tabby, adventurous and brave, always ready to explore new territories. He is curious about everything and enjoys interactive play that mimics hunting. Cody is also a loyal companion, offering strong head-butts and rumbling purrs to show his affection and contentment with his family.
Score: 0.036555873965458334, Document ID: 5, Text: Cosmo: Cosmo, with his wide, knowing eyes, seems to ponder the universe's mysteries. He’s an endearingly quirky character, often found investigating unusual objects or engaging in peculiar solo games. Highly intelligent and observant, Cosmo loves exploring new spaces, and his quiet, thoughtful nature makes him a fascinating and unique companion.
Score: 0.03654979851488558, Document ID: 54, Text: Jasper (II): Jasper the Second, distinct from his predecessor, is a playful and highly energetic ginger tom. He loves to chase, tumble, and explore every nook and cranny with boundless enthusiasm. Jasper is also incredibly affectionate, always ready for a cuddle after a vigorous play session, a bundle of orange joy.
Score: 0.03652978471877379, Document ID: 12, Text: Leo: Leo, with his magnificent mane-like ruff, carries himself with regal confidence. He is a natural leader, often surveying his domain from the highest point in the room. Affectionate on his own terms, Leo enjoys a good chin scratch and will reward loyalty with his rumbling purr and majestic presence.
```
