+++
title = "CodeSpark"
subtitle = "AI 零代码应用生成平台"
date = 2026-08-31
weight = 1
online = true
status = "online"
featured = true
link = "https://codespark.littlewin.top"   
source = "https://github.com/LittleWin8/CodeSpark"
image = "images/codespark.png"
icon = "⚡"
tags = ["Spring Boot", "AI", "LangChain4j", "Vue 3", "低代码", "PostgreSQL"]
summary = "AI 零代码应用生成平台：对话式生成完整应用，Spring Boot 21 + LangChain4j + Vue 3，对话记忆持久化到 RedisJSON。"
+++

CodeSpark 是一个 **AI 零代码应用生成平台**：用自然语言对话描述需求，平台即生成可运行的完整应用。

## 技术要点

- **后端**：Spring Boot（Java 21）+ LangChain4j（DeepSeek），对话记忆通过 langchain4j-community-redis 持久化到 Redis（需 RedisJSON 模块，如 Redis Stack）
- **前端**：Vue 3 + Vite + pnpm，调用后端 OpenAPI 自动生成 TS 客户端（`pnpm openapi2ts`）
- **智能**：支持 AI 生成完整应用零代码部署应用程序

## 链接

- 在线体验：备案通过后上线为 https://codespark.littlewin.top
- 仓库（前后端）：[codeSpark-backend](https://github.com/LittleWin8/CodeSpark/tree/master/codeSpark-backend) / [codeSpark-frontend](https://github.com/LittleWin8/CodeSpark/tree/master/codeSpark-frontend)
