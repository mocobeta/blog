---
title: 常時稼働AIエージェントの構築：5つの基本プリミティブとアーキテクチャの解説
date: '2026-11-22'
taxonomies:
  categories:
  - Short Posts
  tags:
  - til
  - ai agent
---

出典：[Building always-on agents（Anthropic）](<https://claude.com/resources/articles/building-always-on-agents>)

Museのような常時起動AIエージェントの設計ガイド。スケジュールタスク、イベンド駆動タスク、リアルタイム（ユーザーリクエスト）タスクの３種類を統合されたInboxとOutboxで管理する。