# Java 工程师转行企业级 Agent 开发：从 0 到 1 完整路线图

> 日期：2026-06-10
> 阅读时长：约 60 分钟（实操 6 个月）
> 适合：3 年以上 Java 后端经验、想转企业级 Agent 应用开发的工程师

---

## 写在前面：先吃一颗定心丸

你是不是看到 AI 这波浪潮，心里有点慌？

"Python 才是 AI 的主流语言"、"模型更新太快我跟不上"、"我写 Spring Boot 的怎么转 AI"、"我数学不行做不了"……

**这些担心，90% 都是多余的。**

我直接给你讲个真相：

> **企业级 Agent 开发这个赛道，70% 是后端架构能力，20% 是 AI 工程能力，10% 是 Prompt 和产品能力。**

这意味着什么？意味着你这几年熬过的夜、踩过的坑、调过的 JVM、看过的 Spring 源码、设计过的分布式系统、扛过的高并发——**全都不是白费的。** 在企业级 Agent 领域，这些是核心资产，不是历史包袱。

而且说个内幕：**企业客户买 Agent 系统的时候，最关心的不是"你的模型多酷"，而是"能不能稳定跑 7×24 小时"、"数据会不会泄漏"、"出问题了能不能追溯"。** 这些事儿，Java 工程师干了 10 年，Python 团队反而要补课。

所以，**你不是转行，你是升级。**

下面我会用最白话的方式，给你讲清楚这条路怎么走，每一步要做什么，要看什么书，写什么代码，做什么项目，遇到什么坑。

---

## 第一部分：先看清全景

### 1.1 到底什么是"企业级 Agent"？

我先帮你把概念掰开了讲。

**"Agent"这个词你肯定听烂了，但很多人其实没真懂。** 我给你打个比方：

- **普通 LLM 调用**：你问 AI "北京今天天气怎么样"，AI 回你一句话。这叫"一问一答"，AI 是个"答题机器"。
- **Agent**：你说"帮我订明天去上海的机票，要靠窗、价格不超过 1500、晚上 6 点前到"。AI 自己去查航班、比价格、选座位、调支付、确认订单。这叫"自主完成任务"。

**Agent 和普通 LLM 的本质区别：Agent 会自己"动手"，不只是"动嘴"。**

"动手"的意思是：它能调用工具。能查数据库、能发请求、能写文件、能执行代码、能调别的系统 API。

**那"企业级"又是什么意思？** 

就是**让这个 Agent 能在企业生产环境里跑**，能服务几百几千个员工，能扛住高并发、能审计、能限流、能多租户隔离、能跟企业内部的 SAP/钉钉/企业微信/飞书/自研系统打通。

### 1.2 个人 Agent vs 企业级 Agent（用表格看清楚）

我知道你听过 Hermes、OpenClaw 这些个人 Agent 框架。它们和企业级 Agent 差别很大，我用一张表说清楚：

| 维度 | 个人 Agent（Hermes/OpenClaw） | 企业级 Agent |
|---|---|---|
| **谁在用** | 一个人，自己电脑跑 | 一家公司，几百几千人用 |
| **怎么部署** | 你 `git clone` 下来自己跑 | 运维团队部署到 K8s 集群 |
| **数据存在哪** | 你本机文件 + 你的云盘 | 公司的数据中心 + 多租户隔离 |
| **权限怎么管** | 你自己信任自己 | 严格的权限系统，谁能用啥功能得审批 |
| **出问题怎么办** | 重启就好 | 7×24 不能挂，有 SLA 合同 |
| **花了多少钱** | 你自己掏钱买 API | 公司按部门核算成本 |
| **要审计吗** | 不需要 | 每一步操作都要留痕，金融行业还要过等保 |

**一句话总结：个人 Agent 是"我自己用着爽"，企业级 Agent 是"让一万人用着稳"。**

### 1.3 你已经会的技能，哪些能直接复用？

这是我最想告诉你的好消息——**你的 Java 后端经验，80% 都能直接平移到企业级 Agent 领域。**

我列一下对应关系，你心里就有数了：

| 你会的东西 | 在企业级 Agent 里干啥 |
|---|---|
| Spring Boot 微服务 | Agent Runtime 本质就是个无状态微服务 |
| MyBatis / JPA + MySQL / PG | 存 Agent 的记忆、对话历史、用户配置 |
| Redis | 存会话上下文、做限流、做缓存 |
| Kafka / RocketMQ | Agent 异步任务、工具调用事件流 |
| 多线程 / 线程池 | Agent 并发工具调用、流式输出 |
| Spring Security + RBAC | **多租户权限隔离**（企业级核心难点） |
| Docker + K8s | Agent 集群部署、工具沙箱 |
| OpenFeign / RestTemplate | 调用 LLM API、调内部系统的 HTTP 接口 |
| JVM 调优 | LLM 推理服务的延迟和吞吐优化 |
| SkyWalking / Zipkin 链路追踪 | **Agent 可观测性**（企业级必备） |
| 单元测试 / 集成测试 | Agent 评测体系 |

**你看，是不是几乎全覆盖了？**

真正要补的，集中在 AI 工程那一块：怎么调 LLM、怎么做 RAG、怎么让 Agent 用工具、怎么管理 Prompt。

---

## 第二部分：你缺什么？要补什么？

我用"优先级 + 周期"的方式给你列清楚。

### 2.1 P0 级别（必学，1-2 个月内搞定）

| 技能 | 学多久 | 怎么学 |
|---|---|---|
| **LLM API 编程**（OpenAI / Claude / 国产模型 SDK） | 1-2 周 | 看官方文档 + 写 5 个小 Demo |
| **Prompt Engineering 系统化** | 2 周 | 读 Anthropic / OpenAI 官方 Prompt 指南 |
| **Spring AI / LangChain4j 框架** | 2 周 | 跟着官方 Quick Start 跑一遍 |
| **RAG 架构**（向量库 + 检索 + 重排） | 3 周 | 用 Spring AI + Milvus 搭一个真实系统 |
| **Function Calling / Tool Use** | 1 周 | 调通一次 LLM 调本地工具的完整流程 |

### 2.2 P1 级别（重要，3-4 个月内搞定）

| 技能 | 学多久 | 怎么学 |
|---|---|---|
| **Agent 编排框架**（LangGraph / Dify / Coze） | 3 周 | 跑官方 Example + 自己改造 |
| **向量库选型**（Milvus / Qdrant / pgvector） | 2 周 | 部署一个集群版，对比 MySQL 用法 |
| **可观测性**（Langfuse / Phoenix） | 2 周 | 接到你现有的 SkyWalking / ELK |
| **Policy Engine**（OPA / Cedar / Spring Security 扩展） | 2 周 | 给你的 Agent 加上"谁能用啥"判断 |

### 2.3 P2 级别（加分项，5-6 个月）

| 技能 | 学多久 | 怎么学 |
|---|---|---|
| **多 Agent 协作 / A2A 协议** | 4 周 | 读 Google A2A 规范 + 跑 Demo |
| **MCP 协议**（Model Context Protocol） | 1 周 | 这是 2025-2026 工具调用的事实标准 |
| **流式输出 / SSE / WebSocket** | 1 周 | 你可能本来就会，再补一下 |
| **国产大模型适配**（DeepSeek / 通义 / 智谱） | 1 周 | 至少要会调一个国产的 |

---

## 第三部分：6 个月详细学习计划（带项目）

下面这 6 个月是**经过验证**的路径。每个月都有明确的产出、明确的项目、明确的技术栈。学完你就有 3 个能写到简历上的实战项目。

---

### 📅 第 1 个月：LLM 基础 + Prompt 工程

**这个月的目标：你能用 Java 调通任何主流 LLM，能写出企业级 Prompt，能流式输出给前端。**

#### 1.1 准备工作（Week 1）

##### 1.1.1 注册账号 + 申请 API Key

你需要至少 2 个模型的 API Key：
- **国产**（国内直连快）：推荐 **DeepSeek**（性价比之王，2026 年企业首选）或者 **通义千问 Qwen3**（阿里云百炼平台）
- **海外**（能力强、文档全）：**OpenAI GPT-4o** 或 **Anthropic Claude Sonnet 4.5**

**国产注册**：
- DeepSeek：https://platform.deepseek.com/ → 注册 → 充值 10 块（够用半年）→ 创建 API Key
- 通义千问：https://dashscope.aliyun.com/ → 阿里云账号 → 开通"百炼"服务 → 创建 API Key
- 智谱 GLM：https://open.bigmodel.cn/ → 注册 → 领免费额度 → 创建 Key

**海外注册**（你懂的，需要点手段）：
- OpenAI：https://platform.openai.com/
- Anthropic：https://console.anthropic.com/

##### 1.1.2 本地环境准备

你已经有 Java 开发环境了，需要补几个：
```bash
# Maven（你应该装了）
mvn -version

# 安装 jq（处理 JSON 好看）
sudo apt install jq

# 安装 curl（你应该有了）
curl --version

# 推荐装一个 API 测试工具：Insomnia 或 Postman
# 我个人更推荐 HTTPie，更轻
pip install httpie
```

##### 1.1.3 第一个 Hello World

先别急着上框架，手写一个最原始的 HTTP 调用 LLM。

**创建一个 Maven 项目**：
```bash
mvn archetype:generate -DgroupId=com.learn.agent \
    -DartifactId=llm-hello-world \
    -DarchetypeArtifactId=maven-archetype-quickstart
cd llm-hello-world
```

**pom.xml 加依赖**：
```xml
<dependencies>
    <!-- HTTP 客户端 -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>
    <!-- JSON 处理 -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>
    <!-- 日志 -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>2.0.12</version>
    </dependency>
</dependencies>
```

**写一个最简的 LLM 客户端**：
```java
package com.learn.agent;

import okhttp3.*;
import com.fasterxml.jackson.databind.*;
import java.util.*;

public class HelloLLM {
    public static void main(String[] args) throws Exception {
        // 1. 准备 API Key（生产环境从环境变量读）
        String apiKey = System.getenv("DEEPSEEK_API_KEY");
        String url = "https://api.deepseek.com/chat/completions";
        
        // 2. 构造请求
        OkHttpClient client = new OkHttpClient();
        Map<String, Object> message = new HashMap<>();
        message.put("role", "user");
        message.put("content", "用一句话解释什么是 Agent");
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", "deepseek-chat");
        requestBody.put("messages", List.of(message));
        requestBody.put("temperature", 0.7);
        
        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(requestBody);
        
        // 3. 发送请求
        Request request = new Request.Builder()
            .url(url)
            .addHeader("Authorization", "Bearer " + apiKey)
            .addHeader("Content-Type", "application/json")
            .post(RequestBody.create(json, MediaType.get("application/json")))
            .build();
        
        // 4. 打印结果
        try (Response response = client.newCall(request).execute()) {
            String responseBody = response.body().string();
            JsonNode root = mapper.readTree(responseBody);
            String content = root.path("choices").get(0).path("message").path("content").asText();
            System.out.println("AI 说: " + content);
            System.out.println("用了 Token: " + root.path("usage").path("total_tokens").asInt());
        }
    }
}
```

**运行**：
```bash
export DEEPSEEK_API_KEY="sk-你的key"
mvn compile exec:java -Dexec.mainClass="com.learn.agent.HelloLLM"
```

**你应该看到输出**：
```
AI 说: Agent 是一个能自主理解任务、规划步骤、调用工具来完成目标的 AI 程序，...
用了 Token: 87
```

**这一步的意义**：你跑通了 LLM 调用，后续所有项目都基于这个基本盘。

#### 1.2 升级版：流式输出（Week 1-2）

上面的代码是等 AI 全部生成完才一次性返回。**用户体验差。** 企业级必须支持流式输出（SSE，Server-Sent Events），AI 一边生成一边推给前端。

```java
// 流式输出版本
public class HelloStreaming {
    public static void main(String[] args) throws Exception {
        String apiKey = System.getenv("DEEPSEEK_API_KEY");
        String url = "https://api.deepseek.com/chat/completions";
        
        OkHttpClient client = new OkHttpClient.Builder()
            .readTimeout(60, java.util.concurrent.TimeUnit.SECONDS)  // 流式要长超时
            .build();
        
        Map<String, Object> message = Map.of(
            "role", "user",
            "content", "写一首关于 Java 程序员学 AI 的诗"
        );
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", "deepseek-chat");
        requestBody.put("messages", List.of(message));
        requestBody.put("stream", true);  // ← 关键：开流式
        
        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(requestBody);
        
        Request request = new Request.Builder()
            .url(url)
            .addHeader("Authorization", "Bearer " + apiKey)
            .addHeader("Content-Type", "application/json")
            .post(RequestBody.create(json, MediaType.get("application/json")))
            .build();
        
        try (Response response = client.newCall(request).execute()) {
            // SSE 格式：每行 "data: {...}\n\n"
            java.io.BufferedReader reader = new java.io.BufferedReader(
                response.body().charStream()
            );
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.startsWith("data: ")) {
                    String data = line.substring(6);
                    if (data.equals("[DONE]")) break;
                    JsonNode chunk = mapper.readTree(data);
                    String delta = chunk.path("choices").get(0)
                        .path("delta").path("content").asText("");
                    if (!delta.isEmpty()) {
                        System.out.print(delta);  // 一个字一个字打
                        System.out.flush();
                    }
                }
            }
        }
    }
}
```

**你应该看到**：AI 一边生成，屏幕上一边打字出来。**这就是企业级 ChatGPT 的"打字机效果"。**

#### 1.3 Prompt 工程系统化（Week 2-3）

很多人觉得 Prompt 就是"会提问就行"。**错。** 企业级 Prompt 是一套**工程方法论**。

##### 1.3.1 Prompt 的基本结构

一个好的 Prompt 通常包含这 5 部分：

```
1. 角色（Role）       → 你现在是资深 Java 架构师
2. 任务（Task）       → 帮我审查这段代码
3. 上下文（Context）  → 这是 Spring Boot 项目，用 MyBatis，PG 数据库
4. 约束（Constraint） → 不要超过 200 字，重点关注线程安全和 SQL 注入
5. 输出格式（Format） → 用 Markdown 输出，每行代码前加行号
```

**实战示例**（一个能直接用的"代码审查 Prompt"）：

```markdown
# 角色
你是一位有 15 年 Java 经验的资深架构师，专精高并发、分布式系统、性能优化。

# 任务
审查用户提交的 Java 代码，给出专业的 review 意见。

# 审查重点
1. 线程安全（特别是多线程下的共享变量）
2. 资源泄漏（数据库连接、文件句柄）
3. 异常处理（不要吞异常，不要用 Exception 抓所有）
4. 性能（是否有 N+1、循环查 DB、不必要的对象创建）
5. 命名和可读性

# 输出格式
```markdown
## 总评
[1 句话总结]

## 🔴 严重问题（必须改）
- [文件名:行号] 问题描述 + 修复代码

## 🟡 建议优化（最好改）
- [文件名:行号] 建议 + 示例

## 🟢 做得好的地方
- 列出 3 个亮点
```

# 不要做的事
- 不要啰嗦，控制在 300 字以内
- 不要说"代码整体不错"这种废话
- 严重问题必须给出可运行的修复代码
```

**这个 Prompt 有什么讲究？**
- "15 年架构师"——**激活模型的专业知识**（Prompt 圈叫"角色激活"）
- 5 个审查重点——**给模型明确的思考框架**
- 明确的输出格式——**保证输出稳定可解析**
- "不要做的事"——**约束模型不要跑偏**
- "严重问题必须给修复代码"——**提高输出可用性**

##### 1.3.2 Prompt 的版本管理

**重要**：Prompt 跟代码一样要进 Git。**Prompt 改了，AI 行为就变了**，必须能追溯。

推荐项目结构：
```
src/main/resources/prompts/
├── code-review/
│   ├── v1.0.0.md    # 最初版
│   ├── v1.1.0.md    # 加了异常处理要求
│   └── v1.2.0.md    # 优化了输出格式
├── customer-service/
│   └── v1.0.0.md
└── sql-generator/
    └── v2.0.0.md
```

加载 Prompt 的 Java 代码：
```java
@Component
public class PromptLoader {
    @Value("classpath:prompts/code-review/v1.2.0.md")
    private Resource promptResource;
    
    private String promptContent;
    
    @PostConstruct
    public void load() throws IOException {
        promptContent = StreamUtils.copyToString(
            promptResource.getInputStream(), 
            StandardCharsets.UTF_8
        );
    }
    
    public String get(String name, String version) {
        // 从 Redis 缓存里取，支持热更新
        return redisTemplate.opsForValue().get("prompt:" + name + ":" + version);
    }
}
```

##### 1.3.3 Prompt 评测

**怎么知道我的 Prompt 改版后变好了还是变差了？**

准备一个**测试集**：

```json
[
  {
    "id": 1,
    "input": "public class UserService { public User getUser(Long id) { return userDao.findById(id).orElse(null); } }",
    "expected": "应该提到'返回 null 是反模式'和'应该用 Optional 配合业务异常'"
  },
  {
    "id": 2,
    "input": "List<User> users = new ArrayList(); for (Long id : ids) { users.add(userDao.findById(id).get()); }",
    "expected": "应该提到'N+1 查询'和'批量查询优化'"
  }
]
```

每次改 Prompt 跑一遍测试集，看 AI 输出是不是更符合预期。**这就是企业级的 Prompt 迭代流程。**

#### 1.4 第 1 个月产出

✅ 一个能流式输出 LLM 的 Java 客户端（500 行左右）  
✅ 一套企业级 Prompt 模板（至少 3 个场景）  
✅ 一个 Prompt 测试集（50+ 条）  
✅ 第 1 个项目：**GitHub 开源一个 `llm-client-spring-boot-starter`**

---

### 📅 第 2 个月：RAG 架构（企业 Agent 第一战场）

**这个月的目标：你能搭一个生产可用的"企业内部知识库问答系统"，员工能问"年假怎么请"，AI 答得准、还带引用链接。**

#### 2.1 什么是 RAG？（用最白话讲）

RAG 全称叫"检索增强生成"（Retrieval-Augmented Generation）。听起来很玄，我打个比方你就懂了：

**你是一名新员工**，想了解公司年假政策。你有两个选择：
- **选择 A**：直接问 HR 同事（这就是普通 LLM——靠"脑子里的记忆"回答，容易瞎编）
- **选择 B**：先去公司内网搜"年假"，找到《员工手册》第 38 页，然后把相关内容摘出来，再去问 HR（这就是 RAG——**先查资料，再回答**）

**RAG 就是给 AI 装了一个"先查资料"的步骤**，让它回答问题时**有据可查**，不瞎编。

#### 2.2 RAG 系统的完整架构

我画个完整的图，你照着搭：

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│ PDF/Word/   │ →  │ 文档解析      │ →  │ 文本分块      │
│ Confluence  │    │ (Tika/POI)  │    │ (Chunking)   │
└─────────────┘    └──────────────┘    └──────────────┘
                                               ↓
                                        ┌──────────────┐
                                        │ Embedding    │
                                        │ (BGE-M3)     │
                                        └──────────────┘
                                               ↓
                                        ┌──────────────┐
                                        │ 向量库        │
                                        │ (Milvus)     │
                                        └──────────────┘

用户提问: "年假怎么请？"
        ↓
┌──────────────┐
│ Embedding    │ ← 同样的 Embedding 模型
└──────────────┘
        ↓
┌──────────────┐
│ 向量检索      │ ← 从 Milvus 找最相关的 Top 20 段落
│ (Milvus)    │
└──────────────┘
        ↓
┌──────────────┐
│ 重排(Rerank)│ ← 用 bge-reranker 精选 Top 5 最相关的
└──────────────┘
        ↓
┌──────────────┐
│ LLM 生成     │ ← 把 Top 5 喂给 AI，让它基于这些内容回答
└──────────────┘
        ↓
    输出: "根据《员工手册》第 38 页，年假需提前 3 天申请..."
```

#### 2.3 项目实战：企业内部知识库 RAG

##### 2.3.1 项目结构

```
company-knowledge-rag/
├── src/main/java/com/company/rag/
│   ├── RagApplication.java
│   ├── config/
│   │   ├── MilvusConfig.java
│   │   ├── LLMConfig.java
│   │   └── EmbeddingConfig.java
│   ├── document/
│   │   ├── DocumentParser.java       # 解析 PDF/Word
│   │   └── TextChunker.java          # 文本分块
│   ├── embedding/
│   │   └── BgeEmbeddingClient.java   # 调用 BGE Embedding
│   ├── vectorstore/
│   │   └── MilvusClient.java         # Milvus 封装
│   ├── rerank/
│   │   └── BgeRerankClient.java      # 重排
│   ├── service/
│   │   ├── IndexService.java         # 建索引
│   │   └── QueryService.java         # 查询
│   ├── controller/
│   │   └── RagController.java        # REST API
│   └── dto/
│       ├── IndexRequest.java
│       └── QueryRequest.java
├── src/main/resources/
│   ├── application.yml
│   └── prompts/
│       └── rag-answer.md
├── pom.xml
└── README.md
```

##### 2.3.2 pom.xml 关键依赖

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Milvus 客户端 -->
    <dependency>
        <groupId>io.milvus</groupId>
        <artifactId>milvus-sdk-java</artifactId>
        <version>2.4.0</version>
    </dependency>
    
    <!-- OpenAI 兼容客户端（DeepSeek/Qwen 都兼容 OpenAI 协议）-->
    <dependency>
        <groupId>com.openai</groupId>
        <artifactId>openai-java</artifactId>
        <version>0.20.0</version>
    </dependency>
    
    <!-- 文档解析 -->
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-core</artifactId>
        <version>2.9.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-parsers-standard-package</artifactId>
        <version>2.9.1</version>
    </dependency>
    
    <!-- Apache POI 处理 Word/Excel -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.5</version>
    </dependency>
    
    <!-- PDF 处理 -->
    <dependency>
        <groupId>org.apache.pdfbox</groupId>
        <artifactId>pdfbox</artifactId>
        <version>3.0.2</version>
    </dependency>
    
    <!-- 工具库 -->
    <dependency>
        <groupId>cn.hutool</groupId>
        <artifactId>hutool-all</artifactId>
        <version>5.8.27</version>
    </dependency>
</dependencies>
```

##### 2.3.3 文档解析（核心代码）

```java
@Service
public class DocumentParser {
    
    /**
     * 解析任意格式的文档，返回纯文本
     */
    public String parse(File file) throws Exception {
        String filename = file.getName().toLowerCase();
        
        if (filename.endsWith(".pdf")) {
            return parsePdf(file);
        } else if (filename.endsWith(".docx") || filename.endsWith(".doc")) {
            return parseWord(file);
        } else if (filename.endsWith(".xlsx") || filename.endsWith(".xls")) {
            return parseExcel(file);
        } else if (filename.endsWith(".md") || filename.endsWith(".txt")) {
            return Files.readString(file.toPath());
        } else {
            // 其他格式用 Tika
            return parseWithTika(file);
        }
    }
    
    private String parsePdf(File file) throws IOException {
        try (PDDocument document = Loader.loadPDF(file)) {
            PDFTextStripper stripper = new PDFTextStripper();
            return stripper.getText(document);
        }
    }
    
    private String parseWord(File file) throws Exception {
        try (XWPFDocument doc = new XWPFDocument(new FileInputStream(file))) {
            StringBuilder text = new StringBuilder();
            // 遍历段落
            for (XWPFParagraph para : doc.getParagraphs()) {
                text.append(para.getText()).append("\n");
            }
            // 遍历表格
            for (XWPFTable table : doc.getTables()) {
                for (XWPFTableRow row : table.getRows()) {
                    for (XWPFTableCell cell : row.getTableCells()) {
                        text.append(cell.getText()).append("\t");
                    }
                    text.append("\n");
                }
            }
            return text.toString();
        }
    }
    
    private String parseWithTika(File file) throws Exception {
        Tika tika = new Tika();
        return tika.parseToString(file);
    }
}
```

##### 2.3.4 文本分块（最关键的一步）

**分块决定了 RAG 的天花板。** 分错了，再好的模型也白搭。

我给你一个**经过实战检验的策略**：

```java
@Service
public class TextChunker {
    
    /**
     * 智能分块：按段落优先 + 固定窗口兜底 + 重叠
     */
    public List<String> chunk(String text, int chunkSize, int overlap) {
        List<String> chunks = new ArrayList<>();
        
        // 1. 先按段落分（保留语义完整性）
        String[] paragraphs = text.split("\n\n+");
        StringBuilder current = new StringBuilder();
        
        for (String para : paragraphs) {
            para = para.trim();
            if (para.isEmpty()) continue;
            
            // 当前块加上这一段会不会超长？
            if (current.length() + para.length() + 2 > chunkSize) {
                // 超了就保存当前块，开新块
                if (current.length() > 0) {
                    chunks.add(current.toString());
                    // 保留 overlap 部分（最后 overlap 字符）
                    String overlapText = current.substring(
                        Math.max(0, current.length() - overlap)
                    );
                    current = new StringBuilder(overlapText);
                }
            }
            
            current.append(para).append("\n\n");
            
            // 如果单个段落就超长，硬切
            if (current.length() > chunkSize * 2) {
                chunks.add(current.toString());
                current = new StringBuilder();
            }
        }
        
        if (current.length() > 0) {
            chunks.add(current.toString());
        }
        
        return chunks;
    }
}
```

**分块参数怎么选？（企业级默认值）**
- `chunkSize = 512`（字符数，不是 token 数；约 200-300 token）
- `overlap = 50`（相邻块重叠 50 字符）
- 重要：保留**元数据**（来源文件、页码、章节）

**为什么不是越大越好？**
- 太大：检索时容易把不相关内容带进来，AI 看着一坨文字抓不到重点
- 太小：语义不完整，AI 看一句话不知道在讲啥
- 经验值：256-512 字符最佳

##### 2.3.5 Embedding 和 Milvus 存储

**Embedding 是什么？** 把文字变成一串数字（向量），让计算机能算"两个句子像不像"。

```java
@Service
public class BgeEmbeddingClient {
    
    private final RestTemplate restTemplate = new RestTemplate();
    
    @Value("${embedding.api-url}")
    private String apiUrl;  // 例如 https://api.siliconflow.cn/v1/embeddings
    
    @Value("${embedding.api-key}")
    private String apiKey;
    
    /**
     * 把一段文字转成 1024 维向量（BGE-M3 的维度）
     */
    public List<Float> embed(String text) {
        Map<String, Object> request = Map.of(
            "model", "BAAI/bge-m3",
            "input", text
        );
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(apiKey);
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(request, headers);
        
        ResponseEntity<Map> response = restTemplate.exchange(
            apiUrl, HttpMethod.POST, entity, Map.class
        );
        
        List<Double> embedding = (List<Double>) 
            ((Map<String, Object>) ((List<?>) response.getBody().get("data")).get(0))
                .get("embedding");
        
        return embedding.stream().map(Double::floatValue).collect(Collectors.toList());
    }
}
```

**Milvus 集成**：

```java
@Service
public class MilvusVectorStore {
    
    private MilvusServiceClient milvusClient;
    
    @PostConstruct
    public void init() {
        milvusClient = new MilvusServiceClient(
            ConnectParam.newBuilder()
                .withHost("localhost")
                .withPort(19530)
                .build()
        );
        
        // 创建集合（如果不存在）
        createCollectionIfNotExists();
    }
    
    private void createCollectionIfNotExists() {
        String collectionName = "company_knowledge";
        // ... 创建 1024 维向量集合的代码
    }
    
    /**
     * 把文本块 + 向量存入 Milvus
     */
    public void insert(String text, List<Float> embedding, Map<String, String> metadata) {
        // 构造插入数据
        List<InsertParam.Field> fields = Arrays.asList(
            InsertParam.Field.builder()
                .fieldName("text")
                .data(Collections.singletonList(text))
                .build(),
            InsertParam.Field.builder()
                .fieldName("embedding")
                .data(Collections.singletonList(embedding))
                .build(),
            InsertParam.Field.builder()
                .fieldName("source")
                .data(Collections.singletonList(metadata.get("source")))
                .build()
        );
        
        milvusClient.insert(InsertParam.newBuilder()
            .withCollectionName("company_knowledge")
            .withFields(fields)
            .build());
    }
    
    /**
     * 向量检索：找最相似的 Top K
     */
    public List<SearchResult> search(List<Float> queryEmbedding, int topK) {
        SearchParam searchParam = SearchParam.newBuilder()
            .withCollectionName("company_knowledge")
            .withVectorFieldName("embedding")
            .withVectors(Collections.singletonList(queryEmbedding))
            .withTopK(topK)
            .build();
        
        R<SearchResults> response = milvusClient.search(searchParam);
        return response.getData().getResults();
    }
}
```

##### 2.3.6 RAG 主流程（拼装起来）

```java
@Service
public class QueryService {
    
    @Autowired private BgeEmbeddingClient embeddingClient;
    @Autowired private MilvusVectorStore vectorStore;
    @Autowired private BgeRerankClient rerankClient;
    @Autowired private ChatClient chatClient;  // LLM 客户端
    
    /**
     * 完整 RAG 查询流程
     */
    public String query(String question) {
        // Step 1: 把用户问题转成向量
        List<Float> questionEmbedding = embeddingClient.embed(question);
        
        // Step 2: 向量检索 Top 20
        List<SearchResult> candidates = vectorStore.search(questionEmbedding, 20);
        
        // Step 3: 重排选 Top 5
        List<String> docs = candidates.stream()
            .map(r -> r.getEntity().get("text").toString())
            .collect(Collectors.toList());
        List<String> topDocs = rerankClient.rerank(question, docs, 5);
        
        // Step 4: 拼 Prompt，调 LLM
        String prompt = buildPrompt(question, topDocs);
        String answer = chatClient.call(prompt);
        
        return answer;
    }
    
    private String buildPrompt(String question, List<String> docs) {
        StringBuilder context = new StringBuilder();
        for (int i = 0; i < docs.size(); i++) {
            context.append("【参考文档 ").append(i + 1).append("】\n");
            context.append(docs.get(i)).append("\n\n");
        }
        
        return """
            你是一个企业内部知识库助手。请基于【参考文档】回答用户问题。
            
            要求：
            1. 严格基于参考文档回答，不要编造
            2. 如果参考文档里没有答案，明确说"未找到相关信息"
            3. 在引用了具体内容的地方标注【参考文档 N】
            4. 简洁清晰，重点突出
            
            【参考文档】
            %s
            
            【用户问题】
            %s
            """.formatted(context, question);
    }
}
```

##### 2.3.7 提供 REST API

```java
@RestController
@RequestMapping("/api/rag")
public class RagController {
    
    @Autowired private QueryService queryService;
    @Autowired private IndexService indexService;
    
    @PostMapping("/query")
    public String query(@RequestBody QueryRequest req) {
        return queryService.query(req.getQuestion());
    }
    
    @PostMapping("/query-stream")
    public Flux<String> queryStream(@RequestBody QueryRequest req) {
        // 流式输出，用户体验好
        return queryService.queryStream(req.getQuestion());
    }
    
    @PostMapping("/index")
    public String index(@RequestParam("file") MultipartFile file) throws Exception {
        File tempFile = File.createTempFile("upload-", "-" + file.getOriginalFilename());
        file.transferTo(tempFile);
        int chunkCount = indexService.indexFile(tempFile);
        return "已索引 " + chunkCount + " 个文本块";
    }
}
```

#### 2.4 RAG 系统的踩坑指南（血泪经验）

我替你把坑都列出来，能让你少走 3 个月弯路：

##### 坑 1：Embedding 模型选错
- ❌ 不要用 OpenAI 的 text-embedding-3 跑**中文**
- ✅ 中文用 **BGE-M3**（智源研究院开源，免费，中文 SOTA）
- ✅ 或者 **M3E**（哈工大开源）

##### 坑 2：不做重排（Rerank）
- ❌ 直接用向量检索的 Top 5 喂给 LLM
- ✅ 一定要**先检索 Top 20，再用重排模型精选 Top 5**
- 重排模型推荐 **bge-reranker-v2-m3**（准确率能从 60% 拉到 85%）

##### 坑 3：分块粒度不对
- ❌ 整篇文档当一个 chunk（太大，检索不精准）
- ❌ 一句话一个 chunk（太小，语义不完整）
- ✅ **512 字符 + 50 重叠**是经验最优值

##### 坑 4：不保留引用来源
- ❌ AI 答对了，但用户不知道依据
- ✅ 每个 chunk 都要带元数据：来源文件、页码、章节、URL
- ✅ 答案里加"**【来源：员工手册.pdf 第 38 页】**"标记

##### 坑 5：不做评测
- ❌ 上线就完事了，效果全靠用户反馈
- ✅ 准备 50+ 条测试集，每次改 RAG 流程都跑一遍评测
- ✅ 推荐用 **Ragas** 或 **DeepEval** 框架

#### 2.5 第 2 个月产出

✅ 一个能跑的企业内部知识库 RAG 系统  
✅ 完整的 README：架构图、效果评测、API 文档  
✅ 第 2 个项目：**GitHub 开源** `company-knowledge-rag`，配 docker-compose 一键起  
✅ 写了至少 2 篇技术博客（"RAG 实战踩坑"+"评测体系搭建"）

---

### 📅 第 3 个月：Agent 核心 — Function Calling + 工具系统

**这个月的目标：你能搭一个可扩展的工具调用框架，让 Agent 能查数据库、查天气、发邮件、调内部 API。**

#### 3.1 Function Calling 是什么？（白话版）

还是用比方：

**普通 LLM**：你问"上海今天多少度？"，AI 答"我无法获取实时信息"。

**Function Calling Agent**：你问"上海今天多少度？"，AI 想"这问题需要查天气 API"，于是**自动调用 `getWeather` 函数**，传入 `city="上海"`，拿到结果后，**再用自然语言告诉你"上海今天 25 度，晴"**。

**整个过程是 LLM 自己决定的**，你不用在代码里写 if-else 判断要不要查天气。

#### 3.2 一次完整的 Function Calling 流程

```
用户: "上海今天多少度？"
         ↓
┌─────────────────┐
│ LLM             │
│ 思考: 需要天气  │
│ 决定调工具      │
│ getWeather      │
│ (city="上海")   │
└─────────────────┘
         ↓
你的代码执行 getWeather("上海")
         ↓
返回: {"temp":25,"weather":"晴"}
         ↓
┌─────────────────┐
│ LLM             │
│ 基于工具结果    │
│ 组织自然语言    │
└─────────────────┘
         ↓
AI: "上海今天 25 度，天气晴"
```

#### 3.3 项目实战：研发效能 Agent

这个月我们做一个**有点挑战但不难**的项目：一个能帮你查工单、查 CI 状态、读代码的"研发助手 Agent"。

##### 3.3.1 项目结构

```
dev-efficiency-agent/
├── src/main/java/com/company/devagent/
│   ├── DevAgentApplication.java
│   ├── agent/
│   │   ├── AgentCore.java           # Agent 主循环
│   │   ├── AgentContext.java        # 上下文管理
│   │   └── AgentStep.java           # 单步推理
│   ├── tools/
│   │   ├── Tool.java                # 工具接口
│   │   ├── ToolRegistry.java        # 工具注册中心
│   │   ├── ToolResult.java          # 工具执行结果
│   │   ├── ToolExecutor.java        # 工具执行器（含鉴权、限流、审计）
│   │   ├── impl/
│   │   │   ├── GitLabTool.java      # 查 GitLab Issue/MR
│   │   │   ├── JenkinsTool.java     # 查构建状态
│   │   │   ├── GrafanaTool.java     # 查监控
│   │   │   ├── CodeSearchTool.java  # 搜代码
│   │   │   └── DatabaseTool.java    # 查 DB
│   │   └── schema/                  # JSON Schema 自动生成
│   ├── llm/
│   │   ├── LLMClient.java           # LLM 客户端封装
│   │   └── MessageBuilder.java
│   ├── memory/
│   │   ├── ShortTermMemory.java     # 短期（Redis）
│   │   └── LongTermMemory.java      # 长期（PG）
│   ├── prompt/
│   │   └── AgentSystemPrompt.md
│   └── api/
│       └── AgentController.java
├── src/main/resources/
│   └── application.yml
├── pom.xml
└── README.md
```

##### 3.3.2 工具接口设计（核心抽象）

```java
public interface Tool {
    /**
     * 工具名（LLM 看）
     */
    String getName();
    
    /**
     * 工具描述（LLM 用来决定是否调用这个工具）
     * 写得越清晰，LLM 用得越准
     */
    String getDescription();
    
    /**
     * 参数的 JSON Schema（LLM 用来生成参数）
     */
    JsonNode getParametersSchema();
    
    /**
     * 实际执行
     */
    ToolResult execute(ToolContext context, JsonNode arguments) throws Exception;
}
```

**一个真实的工具实现 — GitLab Tool**：

```java
@Component
public class GitLabTool implements Tool {
    
    @Autowired private GitLabApi gitlabApi;
    
    @Override
    public String getName() {
        return "gitlab_query";
    }
    
    @Override
    public String getDescription() {
        return """
            查询 GitLab 仓库信息。
            支持的操作：
            - 'get_issue': 根据 issue ID 查 issue 详情
            - 'list_open_mrs': 列出某项目的所有开放 MR
            - 'get_pipeline': 查某次 CI 流水线状态
            
            注意：需要 project_id（项目 ID）才能查询。
            """;
    }
    
    @Override
    public JsonNode getParametersSchema() {
        // 用 Jackson 直接构造 JSON Schema
        ObjectMapper mapper = new ObjectMapper();
        return mapper.createObjectNode()
            .put("type", "object")
            .set("properties", mapper.createObjectNode()
                .set("action", mapper.createObjectNode()
                    .put("type", "string")
                    .put("enum", "get_issue", "list_open_mrs", "get_pipeline")
                    .put("description", "要执行的操作"))
                .set("project_id", mapper.createObjectNode()
                    .put("type", "integer")
                    .put("description", "GitLab 项目 ID"))
                .set("issue_id", mapper.createObjectNode()
                    .put("type", "integer")
                    .put("description", "Issue ID（get_issue 时必填）"))
                .set("pipeline_id", mapper.createObjectNode()
                    .put("type", "integer")
                    .put("description", "Pipeline ID（get_pipeline 时必填）"))
            )
            .set("required", mapper.createArrayNode()
                .add("action").add("project_id"));
    }
    
    @Override
    public ToolResult execute(ToolContext ctx, JsonNode args) throws Exception {
        String action = args.get("action").asText();
        int projectId = args.get("project_id").asInt();
        
        // 1. 鉴权（用 ctx 里的用户 token）
        // 2. 限流（Redis 计数）
        // 3. 审计日志（记录谁、什么时候、查了什么）
        // 4. 实际调用
        switch (action) {
            case "get_issue":
                int issueId = args.get("issue_id").asInt();
                Issue issue = gitlabApi.getIssue(projectId, issueId);
                return ToolResult.success(issue.toString());
            case "list_open_mrs":
                List<MergeRequest> mrs = gitlabApi.getOpenMRs(projectId);
                return ToolResult.success(mrs.toString());
            // ...
        }
    }
}
```

##### 3.3.3 Tool 注册中心

```java
@Component
public class ToolRegistry {
    
    private final Map<String, Tool> tools = new HashMap<>();
    
    public ToolRegistry(List<Tool> toolList) {
        for (Tool tool : toolList) {
            tools.put(tool.getName(), tool);
        }
    }
    
    public Tool get(String name) {
        return tools.get(name);
    }
    
    /**
     * 把所有工具转成 LLM 能理解的格式
     * 喂给 LLM，让它自己选
     */
    public JsonNode toLLMFormat() {
        ObjectMapper mapper = new ObjectMapper();
        ArrayNode toolsArray = mapper.createArrayNode();
        
        for (Tool tool : tools.values()) {
            toolsArray.add(mapper.createObjectNode()
                .put("type", "function")
                .set("function", mapper.createObjectNode()
                    .put("name", tool.getName())
                    .put("description", tool.getDescription())
                    .set("parameters", tool.getParametersSchema())
                )
            );
        }
        return toolsArray;
    }
}
```

##### 3.3.4 Agent 主循环（最核心的代码）

```java
@Service
public class AgentCore {
    
    @Autowired private LLMClient llmClient;
    @Autowired private ToolRegistry toolRegistry;
    @Autowired private ToolExecutor toolExecutor;
    
    private static final int MAX_STEPS = 10;  // 防止死循环
    
    public String run(String userId, String sessionId, String userMessage) {
        // 1. 加载历史
        List<Message> messages = memory.load(sessionId);
        messages.add(new Message("user", userMessage));
        
        // 2. 循环执行 LLM + 工具调用
        for (int step = 0; step < MAX_STEPS; step++) {
            log.info("Agent step {}/{}", step + 1, MAX_STEPS);
            
            // 2.1 调 LLM
            LLMResponse response = llmClient.chat(
                messages, 
                toolRegistry.toLLMFormat()
            );
            
            // 2.2 LLM 决定要不要调工具
            if (response.hasToolCall()) {
                ToolCall call = response.getToolCall();
                log.info("LLM 决定调用工具: {} 参数: {}", 
                    call.getName(), call.getArguments());
                
                // 2.3 执行工具
                ToolResult result = toolExecutor.execute(
                    userId, call.getName(), call.getArguments()
                );
                
                // 2.4 把工具结果加入对话历史
                messages.add(new Message("assistant", null, call));
                messages.add(new Message("tool", result.toJson()));
                
                // 2.5 继续循环，让 LLM 基于工具结果再回答
            } else {
                // LLM 给出最终答案
                String finalAnswer = response.getContent();
                
                // 保存对话历史
                memory.save(sessionId, messages);
                return finalAnswer;
            }
        }
        
        return "抱歉，思考超过最大步数限制仍未完成。";
    }
}
```

##### 3.3.5 关键：System Prompt 决定 Agent 行为

```markdown
# 角色
你是"研发效能助手"，一位资深的研发工程师助手，能帮助开发者查询 GitLab Issue、MR、CI 状态、监控数据等。

# 能力
- 查 GitLab Issue / MR / Pipeline
- 查 Jenkins 构建状态
- 查 Grafana 监控
- 搜索公司内部代码库

# 工作方式
1. 仔细分析用户问题，判断是否需要调用工具
2. 如果需要，选择最合适的工具，传入正确参数
3. 基于工具返回的结果，组织清晰的回答
4. 如果一次工具调用不够，可以多次调用

# 输出规范
- 中文回答
- 结构化输出：可以用 Markdown 表格、列表
- 涉及数据时，标明来源（如"根据 GitLab Issue #123"）
- 简洁，不啰嗦

# 注意事项
- 不要重复调用同一个工具（除非必要）
- 不要假设参数，找不到就问用户
- 涉及删除、修改的操作不要做（这是只读 Agent）
```

#### 3.4 工具系统的踩坑指南

##### 坑 1：LLM 选错工具
- **症状**：明明有 `getWeather` 工具，LLM 偏偏调 `getNews`
- **解决**：工具描述要写得**非常明确**，多举几个调用示例
- **进阶**：在 System Prompt 里给 Few-shot 例子

##### 坑 2：LLM 死循环
- **症状**：Agent 一直调同一个工具，跳不出来
- **解决**：**必须有最大步数限制**（通常 5-10 步）

##### 坑 3：工具参数解析失败
- **症状**：LLM 生成的 JSON 格式不对
- **解决**：
  - 用 `JSON Schema` 强校验
  - 解析失败时把错误信息反馈给 LLM，让它重试

##### 坑 4：工具执行慢导致超时
- **症状**：一个工具卡住，整个 Agent 卡住
- **解决**：
  - 工具执行必须**有超时控制**（如 30 秒）
  - 异步执行，LLM 流式返回状态
  - 关键工具走**降级方案**

##### 坑 5：工具调用没审计
- **症状**：谁、什么时候、调了什么工具、传了什么参数，全都不知道
- **解决**（**企业级强制要求**）：
  - 每次工具调用都落审计日志
  - 日志至少保留 180 天（金融行业）
  - 包含：用户 ID、租户 ID、工具名、参数哈希、执行时间、结果摘要

#### 3.5 第 3 个月产出

✅ 一个完整的"研发效能 Agent"，至少 5 个工具  
✅ Tool 接口标准化、注册中心化  
✅ 完整审计日志  
✅ 第 3 个项目：**GitHub 开源** `dev-efficiency-agent`  
✅ 写了 1 篇深度博客："企业级 Agent 工具系统设计"

---

### 📅 第 4 个月：企业级三大件 — 权限、可观测、成本

**这个月的目标：让你的 Agent 能在企业生产环境跑——能管权限、能监控、能算成本。**

#### 4.1 多租户权限设计（最难的）

##### 4.1.1 为什么难？

企业级 Agent 服务 1000 个员工，**A 的权限和 B 的权限完全不同**：
- A 是 HR 部门的，能查员工档案
- B 是销售部门的，能查客户信息
- C 是新员工，啥都查不了
- D 是管理员，啥都能查

**Agent 调工具时，必须判断"当前用户有没有权限调这个工具"。**

##### 4.1.2 设计 AgentContext

```java
public class AgentContext {
    private String tenantId;          // 租户 ID
    private String userId;            // 用户 ID
    private String userName;          // 用户名
    private Set<String> roles;        // 角色：HR/销售/管理员
    private Set<String> departments;  // 部门
    private String dataScope;         // 数据范围：本人/本部门/全公司
    private Map<String, Object> attrs; // ABAC 自定义属性
    
    // ... getter / setter
}
```

##### 4.1.3 用 AOP 做权限拦截

```java
@Aspect
@Component
public class ToolPermissionAspect {
    
    @Autowired private PolicyEngine policyEngine;
    
    @Before("execution(* com.company.devagent.tools.*.execute(..)) && args(ctx, args)")
    public void check(ToolContext ctx, JsonNode args) {
        // 1. 提取工具调用信息
        String toolName = ctx.getCurrentToolName();
        
        // 2. 构造策略请求
        PolicyRequest request = new PolicyRequest();
        request.setUser(ctx.getUser());
        request.setTool(toolName);
        request.setParams(args);
        
        // 3. 调 Policy Engine
        PolicyResponse response = policyEngine.evaluate(request);
        
        if (!response.isAllow()) {
            throw new PermissionDeniedException(
                "用户 " + ctx.getUser().getUserId() + 
                " 无权限调用工具 " + toolName + 
                " 原因: " + response.getReason()
            );
        }
        
        // 4. 如果有字段级权限，从 args 里删除被禁止的字段
        if (response.getFieldFilter() != null) {
            filterFields(args, response.getFieldFilter());
        }
    }
}
```

##### 4.1.4 OPA 策略示例

```rego
# policy/agent_permission.rego
package agent.permission

import rego.v1

# 默认拒绝
default allow = false

# HR 部门可以查员工档案
allow {
    input.user.department == "HR"
    input.tool == "query_employee"
}

# 销售可以查客户
allow {
    input.user.department == "Sales"
    input.tool == "query_customer"
}

# 管理员可以查所有只读工具
allow {
    "admin" in input.user.roles
    not is_write_tool(input.tool)
}

is_write_tool(tool) {
    startswith(tool, "delete_")
}

is_write_tool(tool) {
    startswith(tool, "update_")
}

# 数据范围限制：销售只能看自己负责的客户
field_filter := {"exclude": ["salary", "id_card"]} {
    not "admin" in input.user.roles
}
```

#### 4.2 可观测性

**为什么要做？** Agent 跑在生产环境，出问题你得知道：
- AI 答得对不对？（质量监控）
- 调用 LLM 花了多少钱？（成本）
- 工具调用慢在哪？（性能）
- 哪个用户用的多？（产品决策）

##### 4.2.1 关键 Trace 信息

每次 Agent 运行至少要记录：

```java
public class AgentTrace {
    private String traceId;          // 唯一 ID
    private String userId;           // 谁
    private String sessionId;        // 哪个会话
    private long startTime;          // 何时开始
    private long endTime;            // 何时结束
    private int totalSteps;          // 多少步
    private int llmCalls;            // LLM 调用次数
    private int toolCalls;           // 工具调用次数
    private long totalTokens;        // 用了多少 token
    private BigDecimal cost;         // 花了多少钱
    private List<StepTrace> steps;   // 每步详情
    private String finalAnswer;      // 最终答案
    private String status;           // 成功/失败
    private String errorMessage;     // 失败原因
}
```

##### 4.2.2 集成 Langfuse（自部署）

Langfuse 是开源的 LLM 可观测平台，能自部署到公司内网。

```java
@Component
public class LangfuseTraceService {
    
    @Autowired private LangfuseClient langfuse;
    
    public TraceSpan startTrace(String userId, String input) {
        return langfuse.span(
            "agent-run",
            Map.of(
                "user_id", userId,
                "input", input
            )
        );
    }
    
    public void logLLMCall(TraceSpan parent, String model, 
                          String prompt, String response, long tokens) {
        parent.event("llm-call", Map.of(
            "model", model,
            "prompt_tokens", tokens,
            "completion_tokens", 0,
            "total_tokens", tokens
        ));
    }
    
    public void logToolCall(TraceSpan parent, String tool, 
                            String args, String result, long durationMs) {
        parent.event("tool-call", Map.of(
            "tool", tool,
            "args", args,
            "duration_ms", durationMs,
            "result_length", result.length()
        ));
    }
}
```

#### 4.3 成本控制

企业用 Agent 最怕**账单爆了**。一个员工一天问 100 次，一次 1 万 token，一个月下来就是天价。

##### 4.3.1 Token 用量统计

```java
@Component
public class TokenUsageRecorder {
    
    @Autowired private RedisTemplate<String, Long> redis;
    
    public void record(String tenantId, String userId, 
                      String model, int promptTokens, int completionTokens) {
        // 1. 按租户累计
        String tenantKey = "token:tenant:" + tenantId + ":" + LocalDate.now();
        redis.opsForValue().increment(tenantKey, promptTokens + completionTokens);
        redis.expire(tenantKey, Duration.ofDays(90));
        
        // 2. 按用户累计
        String userKey = "token:user:" + userId + ":" + LocalDate.now();
        redis.opsForValue().increment(userKey, promptTokens + completionTokens);
        
        // 3. 按模型累计（看哪个模型最贵）
        String modelKey = "token:model:" + model + ":" + LocalDate.now();
        redis.opsForValue().increment(modelKey, promptTokens + completionTokens);
    }
}
```

##### 4.3.2 智能模型路由

**不是所有问题都要 GPT-4o！**

```java
@Service
public class ModelRouter {
    
    /**
     * 根据问题难度选模型，省钱
     */
    public String chooseModel(String question) {
        // 简单问题（打招呼、简单问答）→ 用 7B 小模型
        if (isSimple(question)) {
            return "qwen2.5-7b-instruct";  // 0.0005 元/千 token
        }
        
        // 中等问题（一般咨询）→ 用 72B 中模型
        if (isMedium(question)) {
            return "qwen2.5-72b-instruct";  // 0.004 元/千 token
        }
        
        // 复杂问题（深度推理、代码）→ 用旗舰模型
        return "deepseek-r1";  // 0.014 元/千 token
    }
    
    private boolean isSimple(String q) {
        // 长度 < 20 字 + 不含代码 + 不含专业术语
        return q.length() < 20 
            && !q.contains("```")
            && !q.matches(".*(架构|算法|优化|并发).*");
    }
}
```

##### 4.3.3 Prompt 缓存

**同样问题不问第二遍。**

```java
@Service
public class PromptCache {
    
    @Autowired private RedisTemplate<String, String> redis;
    
    public Optional<String> get(String promptHash) {
        String cached = redis.opsForValue().get("prompt:cache:" + promptHash);
        return Optional.ofNullable(cached);
    }
    
    public void put(String promptHash, String response, Duration ttl) {
        redis.opsForValue().set(
            "prompt:cache:" + promptHash, 
            response, 
            ttl
        );
    }
}
```

#### 4.4 第 4 个月产出

✅ 完整的多租户权限体系（OPA 策略 + AOP 拦截）  
✅ 集成 Langfuse，能看到每次 Agent 运行的完整 Trace  
✅ Token 用量统计 + 模型路由 + Prompt 缓存  
✅ 完整的"成本账单"功能：HR 能看每个部门花了多少

---

### 📅 第 5 个月：多 Agent 编排

**这个月的目标：从"单个 Agent"到"Agent 团队"，让多个 Agent 协同完成复杂任务。**

#### 5.1 为什么需要多 Agent？

**单个 Agent 的局限**：
- Prompt 太长会"分心"，效果下降
- 工具太多选不对
- 复杂任务拆不开

**多 Agent 协同的优势**：
- 每个 Agent 专精一个领域
- 任务可拆解、可追溯
- 可并行执行，效率高

#### 5.2 实战项目：销售助手 Agent 群

**场景**：销售总监说"帮我分析下 Q3 上海大客户流失原因，给我一个应对方案"。

**多 Agent 协作流程**：

```
                  销售总监
                     │
                     ▼
              ┌──────────────┐
              │ Orchestrator │ (协调者)
              └──────────────┘
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 数据 Agent│  │ 舆情 Agent│  │ 方案 Agent│
│ 查 CRM   │  │ 搜新闻    │  │ 写方案    │
│ 查 BI    │  │ 爬评论    │  │ 做 PPT    │
└──────────┘  └──────────┘  └──────────┘
       │             │             │
       └─────────────┼─────────────┘
                     ▼
              统一报告给销售总监
```

#### 5.3 用 LangGraph 实现（Java 版用 Spring AI Alibaba）

```java
// 定义 Agent 节点
public class SalesAnalysisWorkflow {
    
    @Bean
    public StateGraph workflow() {
        // 节点 1：数据收集
        AgentNode dataNode = AgentNode.builder()
            .name("data_collection")
            .description("查 CRM 找流失客户清单，查 BI 找业绩数据")
            .tools(List.of(crmTool, biTool))
            .systemPrompt(loadPrompt("data-agent"))
            .build();
        
        // 节点 2：舆情分析
        AgentNode sentimentNode = AgentNode.builder()
            .name("sentiment_analysis")
            .description("搜公开新闻、社交媒体，看大客户最近有什么抱怨")
            .tools(List.of(searchTool, crawlerTool))
            .systemPrompt(loadPrompt("sentiment-agent"))
            .build();
        
        // 节点 3：方案生成
        AgentNode proposalNode = AgentNode.builder()
            .name("proposal_generation")
            .description("基于数据和舆情分析，生成应对方案")
            .tools(List.of(knowledgeBaseTool))
            .systemPrompt(loadPrompt("proposal-agent"))
            .build();
        
        // 编排
        return new StateGraph()
            .addNode(dataNode)
            .addNode(sentimentNode)
            .addNode(proposalNode)
            .addEdge(START, "data_collection")
            .addEdge("data_collection", "sentiment_analysis")
            .addEdge("sentiment_analysis", "proposal_generation")
            .addEdge("proposal_generation", END);
    }
}
```

#### 5.4 A2A 协议简介

**A2A（Agent-to-Agent）** 是 Google 牵头、2025 年开始推的 Agent 通信标准。简单说就是让不同 Agent 之间能"对话"。

**A2A 的核心概念**：
- **Agent Card**：每个 Agent 的"名片"（能干啥、怎么调）
- **Task**：一次任务
- **Message**：消息（带上下文）
- **Artifact**：产物（最终结果）

**示例 Agent Card**（JSON）：
```json
{
  "name": "数据查询 Agent",
  "description": "能查 CRM、BI、ERP 系统的数据",
  "url": "https://agent.company.com/data-agent",
  "version": "1.0.0",
  "skills": [
    {
      "name": "query_customer",
      "description": "查客户信息"
    },
    {
      "name": "query_sales",
      "description": "查销售业绩"
    }
  ]
}
```

#### 5.5 第 5 个月产出

✅ 完整的多 Agent 销售助手系统  
✅ 理解 A2A 协议，能让 Agent 之间协作  
✅ 写了 1 篇深度博客："多 Agent 编排实战"

---

### 📅 第 6 个月：作品集 + 面试

**这个月目标：3 个项目打磨到能写到简历上，准备面试。**

#### 6.1 3 个作品的最终形态

##### 作品 1：`llm-client-spring-boot-starter`
- **定位**：基础工具库
- **技术亮点**：
  - 统一 LLM API 抽象
  - 支持多模型路由
  - 流式输出
  - 自动重试 + 降级
- **代码量**：~1500 行
- **Star 目标**：50+

##### 作品 2：`company-knowledge-rag`
- **定位**：企业级 RAG 系统
- **技术亮点**：
  - 完整 RAG 流程（解析→分块→Embedding→检索→重排→生成）
  - Milvus 集群版
  - 引用溯源
  - 评测体系
- **代码量**：~3000 行
- **Star 目标**：100+

##### 作品 3：`dev-efficiency-agent`
- **定位**：企业级 Agent 平台
- **技术亮点**：
  - Function Calling + 多工具
  - 多租户权限
  - 可观测性（Langfuse）
  - 成本控制
  - 多 Agent 编排
- **代码量**：~5000 行
- **Star 目标**：200+

#### 6.2 简历怎么写

**错误示范**：
> "熟悉 LLM 技术，了解 RAG 架构，做过 Agent 项目。"

**正确示范**：
> "**内部研发效能 Agent 平台**（团队项目）：基于 Spring AI + Spring Boot 构建，支持 GitLab / Jenkins / Grafana 等多种研发工具的智能调用，集成 OPA 实现多租户 RBAC 权限控制，Langfuse 实现全链路可观测。已在所在团队大规模推广使用，承担日常研发流程自动化工作。"

**数字感、规模感、价值感**——这是 HR 和技术面试官最爱看的。

#### 6.3 面试高频题（提前准备）

##### 技术基础类

**Q1: RAG 和 Fine-tuning 有什么区别？什么场景用哪个？**
- **RAG**：让 AI 查资料，适合"知识频繁更新"（公司政策、产品文档）、"需要可追溯"的场景
- **Fine-tuning**：让 AI 学说话风格/格式，适合"垂直领域话术"、"特定输出格式"场景
- **实际**：先上 RAG，效果不够再考虑 Fine-tuning

**Q2: 你的 RAG 系统怎么评估效果？**
- 准备 50+ 条测试集
- 用 Ragas 框架算指标：Faithfulness（忠实度）、Answer Relevancy（答案相关性）、Context Precision（检索精度）
- 人工抽检 10% 看效果

**Q3: Function Calling 的实现原理？**
- LLM 训练时见过"调工具"的模式
- 你给它工具列表（name + description + JSON Schema）
- 它决定调不调、调用哪个、传什么参数
- 你执行工具，把结果喂回去，它再继续思考

##### 架构设计类

**Q4: 企业级 Agent 的权限怎么设计？**
- 三层：认证（你是谁）→ 授权（你能用啥）→ 审计（你干了啥）
- 用 RBAC + ABAC + 数据范围
- 工具调用前 AOP 拦截
- OPA 集中管理策略
- 所有操作落审计日志

**Q5: Agent 怎么防"死循环"和"高额账单"？**
- 最大步数限制（5-10 步）
- 单次会话 Token 上限
- 单用户/单租户 QPS 限流
- Prompt 缓存
- 智能模型路由
- 实时账单告警

**Q6: Agent 怎么和现有系统集成？**
- 内部系统：写适配器（Feign Client）
- 外部 API：直接调
- 数据库：MyBatis/JPA
- 老系统 SOAP：CXF
- 不开放 API：RPA（UiPath）

##### 业务理解类

**Q7: 企业客户最关心 Agent 的什么？**
- 稳定性（7×24）
- 数据安全（不泄漏）
- 效果可控（评测有数据）
- 成本可控（账单透明）
- 可维护（不依赖某个人）

**Q8: Agent 什么时候不适合上？**
- 决策容错成本极低（如自动转账、自动发药）→ 至少要人审批
- 规则明确的任务 → 传统代码更靠谱
- 数据量极小 → 没必要
- 监管严格但 AI 不可解释 → 走传统

#### 6.4 投递策略

##### 目标公司类型

| 类型 | 代表 | 优势 | 难度 |
|---|---|---|---|
| **垂直 Agent 公司** | 澜码、句子互动、数势 | 直接做 Agent，技术深 | 难 |
| **大厂 AI 部门** | 阿里通义、字节豆包、百度 | 资源多，体系化 | 较难 |
| **传统软件 AI 转型** | 金蝶、用友、销售易 | 行业 Know-how + AI | 中 |
| **AI 创业公司** | 月之暗面、智谱 | 待遇好，成长快 | 中 |
| **企业内转岗** | 你现在公司 | 难度最低，优先 | 易 |

##### JD 看什么

- ✅ **Java + AI 框架**（Spring AI / LangChain4j）
- ✅ **RAG 实战经验**
- ✅ **LLM 调优经验**
- ✅ **企业级架构能力**（你本来就有的）
- ❌ 如果只要求 Python 调包、不要求工程能力，慎入（不是真正的企业级 Agent 岗）

---

## 第四部分：常见问题 Q&A

### Q1：我数学不好，能转吗？
**A**: 99% 的企业级 Agent 工作**不需要数学**。你调 API、写 Prompt、设计系统、调工具，这就是工作。Transformer 推导、注意力机制优化，那是算法工程师的事，不是 Agent 应用工程师的事。

### Q2：我用 Python 还是 Java？
**A**: **看你团队和场景**。
- 团队本来就是 Java 栈、企业内部系统、国产化要求 → 选 Java + Spring AI
- 团队算法背景、快速原型、AI 工具链 Python 多 → 选 Python + LangChain
- **2026 年趋势**：国内大厂的 Agent 平台 80% 是 Java 栈（阿里通义、字节火山、华为盘古），所以 Java 红利在。

### Q3：MCP 协议是什么？要学吗？
**A**: **必须学**。MCP（Model Context Protocol）是 Anthropic 2024 年推的**工具调用标准协议**，2025-2026 年已经成事实标准。
- 简单说：以前每个 Agent 跟每个工具都要单独写适配器；MCP 之后，**只要你的工具支持 MCP 协议，任何 Agent 都能直接用**
- 相当于 Agent 世界的"USB 接口"
- 学习成本：1 周（看官方文档 + 跑 2 个 Demo）

### Q4：国产大模型怎么选？
**A**: 2026 年 6 月的企业级 Agent 推荐组合：
- **主力推理**：DeepSeek-V3 / R1（性价比之王，长文本好）
- **复杂任务**：Qwen3-235B（阿里云百炼，工具调用强）
- **超长上下文**：Kimi（月之暗面，200K 上下文，PDF 解析强）
- **轻量便宜**：GLM-4-Flash（智谱，0.0001 元/千 token，分类、提取类任务）
- **私有化部署**：Qwen2.5 / DeepSeek（开源，可本地跑）

### Q5：学完之后能拿多少薪资？
**A**: 一线城市参考（2026 年 6 月行情）：
- 1-3 年 Java 转 Agent 经验：**25-40K**
- 3-5 年 Java + 2 年 Agent：**35-60K**
- 5+ 年 Java + Agent 架构经验：**50-100K+**
- **关键变量**：能不能拿到一个能上生产环境的项目经验，光有 Demo 不行。

### Q6：要不要考研/读 AI 方向的研究生？
**A**: **绝大多数情况下不要**。
- 研究生 2-3 年，这期间 AI 行业会变 5 轮
- 2-3 年的实战经验 + 持续学习 > 研究生学历
- 除非你想做算法研究员（不是 Agent 应用工程师），否则没必要

### Q7：找不到 Agent 相关的岗位怎么办？
**A**: **先在现有公司内部找机会**。
- 跟领导说"我想用 AI 帮团队提效"
- 拿一个月时间做个内部 Demo（哪怕是 RAG 问答 + 几个工具调用）
- 领导看到价值，自然会让你转岗
- **最差**：你有了实战项目，简历也漂亮了，跳槽也容易

---

## 第五部分：推荐学习资源（持续更新）

### 📚 必读书单

| 书 | 适合阶段 | 理由 |
|---|---|---|
| 《Hands-On Large Language Models》（Jay Alammar） | 入门 | LLM 全景图，图文并茂 |
| 《AI Engineering》（Chip Huyen） | 进阶 | 2025 新书，Agent 为主，O'Reilly 出品 |
| 《Designing Machine Learning Systems》（Chip Huyen） | 进阶 | 系统化思维，ML 系统的工程实践 |
| Spring AI 官方文档 | 实战 | Java 栈必读 |
| Anthropic Engineering Blog | 持续学习 | 企业级 Agent 最佳实践来源 |
| OpenAI Cookbook | 实战 | 大量实操代码 |

### 🌐 必看资源

- **Anthropic Engineering Blog**（https://www.anthropic.com/engineering）— **企业级 Agent 圣经**
- **OpenAI Cookbook**（GitHub）— 大量可运行的示例代码
- **Langfuse Blog**（https://langfuse.com/blog）— 可观测性实战
- **阿里云百炼 / 火山方舟 / 智谱开放平台** — 国产大模型和工具链
- **B 站搜**：搜 "Spring AI 实战"、"RAG 实战"、"Agent 编排" 有一堆教程
- **YouTube 搜**：搜 "Chip Huyen AI Engineering"、"Agent design patterns"

### 🛠️ 推荐工具

| 工具 | 用途 |
|---|---|
| **Dify** | 低代码 Agent 平台，能快速搭 Demo |
| **Coze（扣子）** | 字节跳动的，模板多 |
| **Langfuse** | 开源 LLM 可观测，自部署 |
| **Ragas** | RAG 评测框架 |
| **OPA** | 策略引擎，权限控制 |
| **Cherry Studio** | 国产开源 LLM 客户端 |
| **Cursor** | AI 代码编辑器（你写 Agent 必备） |

### 💬 推荐社区

- **即刻** App：搜"AI Agent"、"LLM" 圈子
- **小红书**：搜"AI 产品"、"Prompt 工程" 有大量实战分享
- **稀土掘金**：搜"Agent"、"RAG" 看中文技术博客
- **GitHub**：关注 `anthropics`、`openai`、`langchain-ai`、`spring-projects` 等官方仓库
- **飞书云文档**：建一个自己的知识库，把学到的都记下来

---

## 第六部分：立即可以开始的 7 天行动

如果你看完这篇心潮澎湃，**别只收藏，立刻做下面这些事**：

### Day 1：环境准备
- [ ] 注册 DeepSeek 账号，拿到 API Key
- [ ] 注册通义千问/智谱，拿到 API Key
- [ ] 本地装好 Maven、Java 17+
- [ ] 跑通本文开头的 Hello LLM 代码

### Day 2-3：流式输出 + Prompt
- [ ] 改写代码支持 SSE 流式输出
- [ ] 写 3 个不同场景的 Prompt 模板
- [ ] 准备 20 条 Prompt 测试集

### Day 4-5：搭 RAG Demo
- [ ] Docker 起一个 Milvus
- [ ] 写文档解析 + 分块 + Embedding 完整流程
- [ ] 用 5 篇 PDF 文档做个小知识库

### Day 6：Function Calling
- [ ] 调通一次 LLM 调本地函数的完整流程
- [ ] 实现一个简单的"查询数据库"工具

### Day 7：整理输出
- [ ] 把这周代码 push 到 GitHub
- [ ] 写一篇学习笔记
- [ ] 给自己打气：**你已经迈出第一步了！**

---

## 写在最后

Java 工程师转企业级 Agent 工程师，**这是一条已经被验证过的路**。我见过很多 3 年、5 年、10 年的 Java 工程师，半年内成功转型。

**你的后端经验不是负担，是你最大的资产。**

企业级 Agent 市场的真相是：
- 模型能力 → 头部 5 家大厂在卷，你做不了
- 工具调用 → 跟你的 Feign Client 经验一脉相承
- RAG → 跟你的搜索引擎/ES 经验相通
- 多租户权限 → 跟你的 Spring Security 经验完全重合
- 可观测性 → 跟你的 SkyWalking 经验相通
- 性能优化 → 跟你的 JVM 调优经验相通

**唯一真正"新"的东西，就是怎么写好 Prompt 和怎么让 LLM 用工具。** 这两个 2 周就能学会。

剩下的，**你早就会了。**

**现在唯一的问题是：你什么时候开始？**

---

## 附录：项目代码模板

我前面提到的 3 个项目，核心代码结构都在文里了。

如果想要**完整可运行**的开源项目，可以参考我 GitHub 上的样板（我会在评论区置顶链接）。

---

**祝你转岗成功。半年后，你就是企业级 Agent 工程师。**

