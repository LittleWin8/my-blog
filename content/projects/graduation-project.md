+++
title = "Link Mind"
subtitle = "基于 Spring Boot 的个人知识管理与智能笔记系统（本科毕业设计）"
date = 2026-01-03
weight = 2
online = false
status = "archived"
featured = false
source = "https://github.com/LittleWin8/GraduationProject"
image = "images/smart-note.png"
icon = "🧠"
tags = ["Spring Boot", "Vue 3", "uni-app", "AI", "MySQL", "毕业设计", "LangChain4j"]
summary = "已完成：前后端分离的个人知识管理与智能笔记系统——微信小程序（社区+笔记）+ Web 管理端 + Spring Boot 多模块后端，内置 LangChain4j + DeepSeek 的摘要/润色/标签推荐与配额管控。"
+++

个人知识管理与智能笔记系统，覆盖「记录 → 沉淀 → 轻量互动」的完整场景。含三个子项目：

- **smart-note-system**：Spring Boot 多模块后端（认证/权限/AI/审核/消息/日志）
- **smart-note-ui**：Web 管理端（Vue 3 + Element Plus，基于 Geeker-Admin 二次开发）
- **smart-note-mp**：微信小程序端（uni-app）

## 技术要点

- 认证与权限：Spring Security + JWT，Redis Token 黑名单，RBAC 按钮级权限
- AI 能力：LangChain4j + DeepSeek，笔记摘要/扩写/润色/总结、标签推荐、**自然语言 → SQL 数据分析**；用户级 Token 配额与月度重置
- 消息与审核：站内消息（8 类）、笔记审核自动通知
- 缓存策略：Redis 浏览量计数 + 定时同步 DB，Lua 保证幂等
- 移动端：uni-app 多端，Markdown 笔记渲染、公开社区流

## 更多

项目已开发完成（本科毕业设计），仅作学习成果归档，未对外部署。构建方式、截图、E-R 图与数据库脚本见 [仓库 README](https://github.com/LittleWin8/GraduationProject#项目预览)。
