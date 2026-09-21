+++
title = "Spring Boot 接入大模型 LangChain4j 实战"
date = 2026-05-12
draft = false
tags = ["Java", "AI", "毕业设计", "LangChain4j"]
+++
> 在毕设项目"智能笔记系统"中，我需要给笔记加上 AI 摘要、写作助手、智能标签推荐、自然语言数据分析四个功能。这篇文章记录我从选型到实现的全过程，重点讲怎么在 Spring Boot 项目中接入大模型 API。

# 一、常见接入方式对比

在 Java/Spring Boot 项目中接入大模型（比如 DeepSeek、ChatGPT、通义千问），常见的有四种方式：

| 方式                | 思路                         | 优点                       | 缺点                           |
| ----------------- | -------------------------- | ------------------------ | ---------------------------- |
| **直接 HTTP 调用**    | 用 OkHttp/RestTemplate 拼请求体 | 零依赖，完全可控                 | 手动管理 JSON、token 统计、流式响应，容易出错 |
| **OpenAI 官方 SDK** | `openai-java` 第三方库         | 类型安全，功能全                 | 只支持 OpenAI 官方 API，其他厂商要自己适配  |
| **Spring AI**     | Spring 官方 AI 框架            | 与 Spring 生态深度集成，抽象度高     | 版本迭代快，文档相对少，学习成本高            |
| **LangChain4j**   | Python LangChain 的 Java 移植 | 支持 OpenAI 兼容接口，文档丰富，社区活跃 | 某些高级特性更新慢于 Python 版          |

**我的选择：LangChain4j + DeepSeek**

原因很实际：

1. **DeepSeek 兼容 OpenAI API 格式**——只要改 `baseUrl`，不用改代码逻辑
2. **LangChain4j 的 **`langchain4j-open-ai`** 模块天然支持**——直接传入 DeepSeek 的地址就能用
3. LangChain4j 文档多、示例丰富，上手快

# 二、第一步：添加 Maven 依赖

在 `note` 模块的 `pom.xml` 中添加两个依赖：

```xml
<!-- LangChain4j 核心 -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
    <version>0.35.0</version>
</dependency>

<!-- OpenAI 兼容接口（DeepSeek 也能用） -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <version>0.35.0</version>
</dependency>

```

为什么不需要额外加 DeepSeek 的 SDK？因为 DeepSeek 的 API 完全兼容 OpenAI 的请求/响应格式，`langchain4j-open-ai` 模块只需要换个 `baseUrl` 就能对接。

# 三、配置 API 连接

## 3.1 配置文件

在 `application-dev.yml` 中配置 DeepSeek 的连接信息：

```yaml
ai:
  deepseek:
    api-key: ${DEEPSEEK_API_KEY:your_key_here}
    base-url: https://api.deepseek.com
    model-name: deepseek-chat
    max-tokens: 500

```

- `api-key`：从 DeepSeek 官网申请的 API Key
- `base-url`：DeepSeek 的 API 地址（如果换成其他兼容厂商，只改这一行）
- `model-name`：模型名称，DeepSeek 用 `deepseek-chat`
- `max-tokens`：AI 回复的最大 token 数

## 3.2 创建模型实例

在 Service 中通过 `@PostConstruct` 创建模型实例：

```java
@Value("${ai.deepseek.api-key:sk-xxx}")
private String apiKey;
@Value("${ai.deepseek.base-url:https://api.deepseek.com}")
private String baseUrl;
@Value("${ai.deepseek.model-name:deepseek-chat}")
private String modelName;
@Value("${ai.deepseek.max-tokens:500}")
private Integer maxTokens;

private ChatLanguageModel chatModel;

@PostConstruct
public void init() {
    chatModel = OpenAiChatModel.builder()
            .apiKey(apiKey)
            .baseUrl(baseUrl)
            .modelName(modelName)
            .maxTokens(maxTokens)
            .build();
}

```

核心类是 `ChatLanguageModel`，它是 LangChain4j 的统一接口。实际创建的是 `OpenAiChatModel`——别被名字骗了，它支持所有 OpenAI 格式兼容的 API。

> **踩坑提醒**：我项目中有三个 Service（摘要、助手、分析）各自创建了一份 `OpenAiChatModel` 实例，代码重复了三遍。更好的做法是抽成一个 `@Configuration` 类统一管理：
> 
> ```java
> @Configuration
> public class AiConfig {
>     @Bean
>     public ChatLanguageModel chatLanguageModel(
>             @Value("${ai.deepseek.api-key}") String apiKey,
>             @Value("${ai.deepseek.base-url}") String baseUrl,
>             @Value("${ai.deepseek.model-name}") String modelName,
>             @Value("${ai.deepseek.max-tokens}") Integer maxTokens) {
>         return OpenAiChatModel.builder()
>                 .apiKey(apiKey)
>                 .baseUrl(baseUrl)
>                 .modelName(modelName)
>                 .maxTokens(maxTokens)
>                 .build();
>     }
> }
> 
> ```
> 
> 然后各 Service 通过构造器注入 `ChatLanguageModel chatModel` 即可（和项目中 `@RequiredArgsConstructor` 的风格一致）。毕设赶进度没来得及重构，但这是生产项目应该做的。

# 四、基础调用

有了 `ChatLanguageModel` 实例，调用大模型的核心代码就三行：

```java
// 1. 构建请求（包含你的 Prompt）
ChatRequest request = ChatRequest.builder()
        .messages(List.of(UserMessage.from("你好，请用一句话介绍 Java")))
        .build();

// 2. 发送请求，拿到响应
ChatResponse response = chatModel.chat(request);

// 3. 提取 AI 的回复
String result = response.aiMessage().text();

// 4. 提取 token 用量
TokenUsage usage = response.tokenUsage();
int inputTokens = usage.inputTokenCount();    // 输入 token 数
int outputTokens = usage.outputTokenCount();  // 输出 token 数

```

就这么简单。不需要手动拼 JSON、不需要处理 HTTP 响应解析，LangChain4j 全部帮你封装好了。

# 五、项目中的四个 AI 功能

接下来讲在毕设项目中落地的四个 AI 功能，每个都是对基础调用的业务化封装。

## 5.1 AI 摘要生成

用户写完一篇笔记，点一下"生成摘要"，AI 返回 100 字以内的摘要和 3-5 个关键词。

```java
// AiSummaryServiceImpl.java
public Map<String, String> generateSummary(Long noteId, Long userId) {
    // 1. 检查配额
    aiQuotaService.checkQuota(userId);

    // 2. 查笔记内容
    Note note = noteMapper.selectById(noteId);

    // 3. 构建 Prompt（内容截断到 2000 字防止超长）
    String prompt = "请为以下笔记生成一段简洁的摘要（100字以内），"
            + "并提取3-5个关键词（逗号分隔）。\n\n"
            + "标题：" + note.getTitle() + "\n"
            + "内容：" + note.getContent().substring(0, Math.min(note.getContent().length(), 2000));

    // 4. 调用 AI
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(UserMessage.from(prompt)))
            .build();
    ChatResponse response = chatModel.chat(request);
    String result = response.aiMessage().text();

    // 5. 解析结果（按换行符分割：第一行是摘要，第二行是关键词）
    String[] parts = result.split("\n", 2);
    String summary = parts[0].trim();
    String keywords = parts.length > 1 ? parts[1].trim() : "";

    // 6. 记录 token 用量，存入数据库
    aiQuotaService.recordUsage(userId, response.tokenUsage(), modelName, noteId);
    noteAiSummaryMapper.insertOrUpdate(/* 摘要实体 */);

    return Map.of("summary", summary, "keywords", keywords);
}

```

几个设计要点：

- **内容截断**：`substring(0, 2000)`——笔记可能很长，但 prompt 越长花费的 token 越多，2000 字足够 AI 理解核心内容
- **结果解析用换行符分割**：prompt 里约定了"摘要和关键词分行返回"，回来直接 `split("\n")`
- **配额检查在前**：先查用户本月 token 用量有没有超限，超了直接拒绝

## 5.2 AI 写作助手

支持三种操作：扩写、润色、总结。本质是同一个接口，区别只在 prompt 不同：

```java
// AiAssistServiceImpl.java
public String assist(Long userId, String content, String action) {
    aiQuotaService.checkQuota(userId);

    String prompt;
    switch (action) {
        case "expand":
            prompt = "请将以下文字扩写到200-300字，保持原意，丰富细节，使用流畅的中文：\n\n" + content;
            break;
        case "polish":
            prompt = "请润色以下文字，使其更加通顺、专业，保持原意不变：\n\n" + content;
            break;
        case "summarize":
            prompt = "请用50字以内总结以下文字的核心内容：\n\n" + content;
            break;
        default:
            throw new ServiceException("不支持的操作类型: " + action);
    }

    ChatRequest request = ChatRequest.builder()
            .messages(List.of(UserMessage.from(prompt)))
            .build();
    ChatResponse response = chatModel.chat(request);
    String result = response.aiMessage().text().trim();

    TokenUsage usage = response.tokenUsage();
    aiQuotaService.recordUsage(userId, usage.inputTokenCount(),
            usage.outputTokenCount(), modelName, null, 1, null);
    return result;
}

```

这就是 **策略模式** 的应用——`switch (action)` 选择不同的 prompt 模板，底层调用完全一样。以后想加新操作（比如"翻译成英文"），只需要加一个 `case` 和对应的 prompt。

## 5.3 AI 标签推荐

这个功能最值得讲——**如何防止 AI 幻觉**。

用户编辑笔记时，AI 从用户已有的标签中推荐 1-3 个。难点在于：AI 可能推荐一个你根本不存在的标签名。

```java
// AiAssistServiceImpl.java
public List<String> recommendTags(Long userId, String content) {
    // 1. 先查出用户所有已有标签
    List<NoteTag> myTags = noteTagMapper.selectList(/* 按 userId 查询 */);
    String tagNames = myTags.stream().map(NoteTag::getName).collect(Collectors.joining("、"));

    // 2. Prompt 中限定选择范围
    String prompt = "以下是我的标签列表：[" + tagNames + "]\n\n"
            + "请从上面的标签中，选出与下面这篇笔记最相关的1-3个标签。\n"
            + "只返回标签名，多个用逗号分隔，不要返回其他内容。"
            + "如果没有匹配的，返回空。\n\n"
            + "笔记内容：\n" + content.substring(0, Math.min(content.length(), 1000));

    // 3. 调用 AI
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(UserMessage.from(prompt)))
            .build();
    ChatResponse response = chatModel.chat(request);
    String text = response.aiMessage().text().trim();

    // 4. 关键：二次校验，过滤掉 AI 凭空捏造的标签
    Set<String> myTagSet = myTags.stream().map(NoteTag::getName).collect(Collectors.toSet());
    List<String> result = new ArrayList<>();
    for (String tag : text.split("[,，、\\s]+")) {
        String t = tag.trim();
        if (!t.isEmpty() && myTagSet.contains(t)) {  // 必须在已有标签中
            result.add(t);
        }
    }
    return result;
}

```

三道防线防止幻觉：

1. **Prompt 限定范围**——"从上面的标签中选"，明确告诉 AI 只能从给定列表里选
2. **要求只返回标签名**——"不要返回其他内容"，减少 AI 发挥空间
3. **后端二次校验**——第 4 步用 `myTagSet.contains(t)` 过滤，AI 返回的标签如果不在用户已有标签列表中，直接丢弃

## 5.4 自然语言数据分析

这是最复杂的功能——管理员输入自然语言问题（如"上周新增了多少篇笔记"），AI 自动生成 SQL 并返回分析结果。

整个流程是**两阶段 LLM 调用**：

```
用户问题 → [第一次 AI 调用] → SELECT SQL → 执行 SQL → 查询结果 → [第二次 AI 调用] → 自然语言回答

```

```java
// AiAnalyzeServiceImpl.java
public Map<String, Object> analyze(Long userId, String question) {
    aiQuotaService.checkQuota(userId);

    // 第一阶段：NL → SQL
    // 把数据库表结构告诉 AI，让它生成查询语句
    ChatRequest sqlReq = ChatRequest.builder()
            .messages(List.of(UserMessage.from(
                    SCHEMA + "\n只返回 SELECT SQL，不加代码块。\n问题：" + question)))
            .build();
    ChatResponse sqlResp = chatModel.chat(sqlReq);
    String sql = sqlResp.aiMessage().text().trim();

    // 安全过滤（防止 AI 生成恶意 SQL）
    sql = sanitizeSql(sql);

    // 执行 SQL
    List<Map<String, Object>> data = jdbcTemplate.execute(/* 执行 SQL，超时 3 秒 */);

    // 第二阶段：结果 → 自然语言总结
    ChatRequest sumReq = ChatRequest.builder()
            .messages(List.of(UserMessage.from(
                    "问题：" + question + "\nSQL：" + sql + "\n结果：" + data + "\n用中文简洁总结。")))
            .build();
    ChatResponse sumResp = chatModel.chat(sumReq);
    String answer = sumResp.aiMessage().text().trim();

    return Map.of("answer", answer, "sql", sql, "data", data);
}

```

**为什么这里用 JdbcTemplate 而不是 MyBatis-Plus？**

项目里其他所有数据库操作都走 MyBatis-Plus，只有这个场景用了 JdbcTemplate。原因很简单：

- **MyBatis-Plus 要求提前定义**：每个查询对应一个实体类 + Mapper 方法，SQL 是固定的
- **AI 生成的 SQL 是动态的**：运行时才知道具体是什么 SQL，没法提前写 Mapper

```java
// MyBatis-Plus：SQL 提前写死，类型安全
noteMapper.selectById(1);
noteMapper.selectList(wrapper);

// JdbcTemplate：直接执行任意 SQL 字符串
jdbcTemplate.execute(stmt -> stmt.executeQuery(sql));  // sql 是 AI 刚生成的

```

简单说：**常规 CRUD 用 MyBatis-Plus，AI 动态 SQL 用 JdbcTemplate**。

## 5.5 SQL 安全防护

让 AI 生成 SQL 然后直接执行，安全风险极高。`sanitizeSql` 方法是整个功能的安全底线：

```java
String sanitizeSql(String sql) {
    // 1. 去掉 AI 喜欢加的 markdown 代码块标记
    sql = sql.replaceAll("```sql\\s*", "").replaceAll("```\\s*", "").trim();
    if (sql.endsWith(";")) {
        sql = sql.substring(0, sql.length() - 1).trim();
    }

    String lower = sql.toLowerCase().replaceAll("/\\*.*?\\*/", " ");

    // 2. 只允许 SELECT
    if (!lower.startsWith("select")) {
        throw new ServiceException("只允许 SELECT 查询");
    }

    // 3. 黑名单：禁止一切写操作
    String[] forbidden = {"insert", "update", "delete", "drop", "alter",
                          "truncate", "create", "exec", "execute", "grant", "revoke"};
    for (String kw : forbidden) {
        if (lower.contains(" " + kw + " ") || lower.contains(";" + kw)) {
            throw new ServiceException("禁止执行 " + kw + " 操作");
        }
    }

    // 4. 禁止查询敏感字段
    String[] blocked = {"credential", "password", "phone", "email", "birthday"};
    for (String col : blocked) {
        if (lower.contains(col)) {
            throw new ServiceException("禁止查询敏感字段: " + col);
        }
    }

    // 5. 自动加 LIMIT 防止大结果集
    if (!lower.contains("limit")) {
        sql = sql + " LIMIT 100";
    }

    return sql;
}

```

五层防护：

- 去掉 markdown 代码块（AI 特别喜欢加 ````sql` 标签）
- 只允许 SELECT 开头
- 关键字黑名单拦截写操作
- 敏感字段黑名单（密码、手机号等）
- 自动加 `LIMIT 100` 防止一次查太多

# 六、Token 用量与配额管理

大模型 API 是按 token 计费的，不做用量控制可能被刷爆。项目中实现了一套完整的配额体系。

## 6.1 数据模型

两张表：

- `ai_usage_log`：每次调用的 token 用量记录（谁、什么时候、用了多少 token、成功还是失败）
- `ai_user_quota`：每个用户的月度配额（token 上限、请求次数上限、已用量）

## 6.2 配额检查逻辑

```java
// AiQuotaServiceImpl.java
public void checkQuota(Long userId) {
    // 1. 首次使用自动创建（默认：每月 10 万 token，50 次请求）
    AiUserQuota quota = quotaMapper.selectById(userId);
    if (quota == null) {
        quota = new AiUserQuota();
        quota.setUserId(userId);
        quota.setMonthlyTokenLimit(100000);
        quota.setMonthlyRequestLimit(50);
        quotaMapper.insert(quota);
    }

    // 2. 跨月自动重置
    LocalDate today = LocalDate.now();
    if (quota.getQuotaResetDate().getMonth() != today.getMonth()) {
        quota.setUsedTokens(0);
        quota.setUsedRequests(0);
        quotaMapper.updateById(quota);
    }

    // 3. 检查请求次数
    if (quota.getUsedRequests() >= quota.getMonthlyRequestLimit()) {
        throw new ServiceException(403, "本月 AI 请求次数已用完");
    }

    // 4. 检查 token 用量（从日志表实时统计）
    int usedTokens = logMapper.sumTokensByUserThisMonth(userId);
    if (usedTokens >= quota.getMonthlyTokenLimit()) {
        throw new ServiceException(403, "本月 AI token 用量已用完");
    }
}

```

几个细节：

- **首次使用自动建配额记录**——不用手动初始化，用户第一次调用 AI 时自动创建
- **按月重置**——比较 `quotaResetDate` 的月份，跨月自动清零。注意这里只比较了月份没比较年份，跨年场景下（12月→1月）恰好也不会误判，但如果是隔年同月可能会有问题，生产环境建议改用 `YearMonth` 比较
- **token 用量从日志表实时聚合**——不依赖配额表的计数器（可能被手动调整），而是 `SELECT SUM(total_tokens) FROM ai_usage_log WHERE user_id = ? AND create_time >= 本月`，更准确

## 6.3 调用后记录

每次 AI 调用完成后，无论成功还是失败，都要记录日志：

```java
// 调用 AI
ChatResponse response = chatModel.chat(request);
TokenUsage usage = response.tokenUsage();

// 记录用量（成功）
aiQuotaService.recordUsage(userId, usage.inputTokenCount(),
        usage.outputTokenCount(), modelName, noteId, 1, null);

```

```java
// 异常时记录（失败也要记，方便排查）
catch (Exception e) {
    aiQuotaService.recordUsage(userId, promptTokens, completionTokens,
            modelName, noteId, 0, e.getMessage());
    throw e;
}

```

管理端可以查看全局用量统计和用户排名：

```
GET /api/admin/ai/stats    → 全局 token 消耗、调用次数
GET /api/admin/ai/ranking  → 用户 AI 使用排行
GET /api/admin/ai/logs     → 调用日志（分页）
PUT /api/admin/ai/quota/{userId}  → 手动调整配额

```

# 七、踩坑总结

回顾整个 AI 集成过程，几个值得记录的坑：

**1. 模型实例重复创建**

三个 Service 各自创建了一份 `OpenAiChatModel`，配置代码重复了三遍。正确做法是抽成 `@Configuration` Bean 统一管理。虽然功能没问题，但维护成本高——改一个配置要改三个地方。

**2. Prompt 越长，token 越贵**

一开始没截断笔记内容，遇到几万字的长文直接把 token 额度烧光。后来加了 `substring(0, 2000)` 的截断，摘要质量和成本之间做了平衡。

**3. AI 返回的格式不稳定**

摘要功能期望"摘要\n关键词"两行格式，但 AI 偶尔会返回一段连贯的文字、或者加多余的说明。目前用 `\n` 分割 + trim 处理，容忍度还行，但生产环境建议用 JSON 格式约定（prompt 中要求返回 `{summary: "...", keywords: "..."}`），然后用 JSON 解析更可靠。

**4. 自然语言 → SQL 的安全问题**

让 AI 直接生成 SQL 执行，等于把数据库的钥匙交给了一个"不可控"的角色。`sanitizeSql` 的五层过滤是必要的——即使 prompt 说了"只返回 SELECT"，AI 也可能生成 `INSERT` 或 `DROP`。**不要过度信任 AI 生成的代码，必须在执行前做安全检查。**

# 八、总结

用 LangChain4j 接入 DeepSeek 大模型，核心就三步：

1. **加依赖**：`langchain4j` + `langchain4j-open-ai`
2. **建模型**：`OpenAiChatModel.builder().apiKey().baseUrl().build()`
3. **调接口**：`chatModel.chat(ChatRequest)` → `response.aiMessage().text()`

在此基础上，围绕业务需求做了四个 AI 功能：摘要、写作助手、标签推荐、数据分析。每个功能的核心差异在于 **Prompt 设计**——同一个底层调用，不同的 prompt 就是不同的 AI 能力。

Token 配额管理和 SQL 安全防护是生产级应用必须考虑的问题。毕设项目中做了基础的处理，但还有不少可以优化的地方（集中化配置、重试机制、流式响应等），留待后续迭代。

---

> **源码参考**：[github.com/LittleWin8/GraduationProject](https://github.com/LittleWin8/GraduationProject/tree/main/smart-note-system/note/src/main/java/com/littlewin/note/service/impl)
