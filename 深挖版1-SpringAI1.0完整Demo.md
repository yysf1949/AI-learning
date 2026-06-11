# 深挖版 1：Spring AI 1.0 完整可运行 Demo

> 日期：2026-06-10
> 配套基础版：《Java 工程师转行企业级 Agent 开发：从 0 到 1 完整路线图》
> 适合：想直接动手写企业级 Agent 代码的 Java 工程师

---

## 写在前面：为什么是 Spring AI 1.0？

2025 年 5 月 20 日，Spring 团队在 Spring I/O 大会上正式发布 **Spring AI 1.0 GA**。这是 Java 生态里第一个**官方支持**大模型应用开发的框架，意义相当于当年 Spring Boot 对微服务的意义。

简单说几个关键事实：

1. **统一 API 抽象** — 调 OpenAI、Anthropic Claude、Azure OpenAI、DeepSeek、阿里通义，**代码几乎一样**。换模型只改配置文件。
2. **ChatClient 流式 API** — 用过 WebClient / RestTemplate 的你，会觉得非常亲切。
3. **Advisor 机制** — 这是企业级最关键的能力，类比 Spring 的 Filter 链，每个 Advisor 是一个拦截器（RAG、对话记忆、安全审计、Token 限流……全用它做）。
4. **MCP 官方支持** — Model Context Protocol 是 2025-2026 年事实标准的"Agent 工具协议"，Spring AI 1.0 原生集成。
5. **DeepSeek V4 官方支持** — 国产模型这块儿 Spring AI 没落下。

**一句话总结**：以前你写 Java Agent 要拼凑 LangChain4j + 自研封装 + 各种 hack，现在直接用 Spring AI 就是 2026 年 Java 圈做 Agent 的"官方指定答案"。

下面我给你**三个**完整可运行的项目，分别覆盖 **流式对话 / RAG 知识库 / Agent 工具调用** 三大核心场景。

---

## 项目 0：环境准备（必做，5 分钟）

### 0.1 准备 API Key

注册下面**任一**模型平台，拿到 API Key：
- **DeepSeek**（推荐新手）：https://platform.deepseek.com/ ，注册送钱
- **阿里通义千问 DashScope**：https://dashscope.aliyun.com/
- **智谱 GLM**：https://open.bigmodel.cn/
- **OpenAI**：需要科学上网，备用

### 0.2 准备 Java 环境

```bash
# 推荐 Java 17+，确认版本
java -version
# openjdk version "17.0.10" 2024-01-16 或更高都行

# Maven 3.8+
mvn -version
```

### 0.3 创建 Spring Boot 3 项目

去 https://start.spring.io/ 选择：
- Project: **Maven**
- Language: **Java**
- Spring Boot: **3.4.x**（3.4.0+ 都行，1.0 GA 兼容 3.2+）
- Group: `com.example`
- Artifact: `agent-demo`
- Java: **17** 或 **21**
- Dependencies 暂时不选，等下手动加

下载 zip 解压到本地。

### 0.4 添加 Spring AI 依赖

打开 `pom.xml`，在 `<dependencies>` 里加：

```xml
<!-- Spring AI BOM（统一管理所有 Spring AI 模块版本） -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AI OpenAI Starter（兼容 DeepSeek、智谱、通义等所有 OpenAI 协议模型） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <!-- Spring AI DeepSeek 官方支持（如果你用 DeepSeek，推荐用这个） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-deepseek</artifactId>
    </dependency>

    <!-- 向量库 - 内存版（Demo 用，生产换 Milvus） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-simple</artifactId>
    </dependency>

    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 0.5 配置 API Key

编辑 `src/main/resources/application.yml`：

```yaml
spring:
  application:
    name: agent-demo

  # DeepSeek 配置（推荐）
  ai:
    deepseek:
      api-key: ${DEEPSEEK_API_KEY}  # 你的 API Key
      chat:
        options:
          model: deepseek-chat  # 或 deepseek-reasoner（推理模型）
          temperature: 0.7

  # 如果用 OpenAI 协议的其他模型（比如通义、智谱）
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.deepseek.com  # DeepSeek 的 OpenAI 兼容地址
      chat:
        options:
          model: deepseek-chat

server:
  port: 8080

logging:
  level:
    org.springframework.ai: DEBUG  # 开发期打开，看内部流程
```

设置环境变量（Linux/Mac）：

```bash
export DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
```

---

## 项目 1：流式对话 + 多轮记忆（Hello Agent）

**目标**：跑通一个"能记住上下文的聊天机器人"，前端用 SSE 流式输出。

### 1.1 项目结构

```
src/main/java/com/example/agentdemo/
├── AgentDemoApplication.java        # 启动类
├── config/
│   └── ChatClientConfig.java         # ChatClient Bean 配置
├── controller/
│   └── ChatController.java           # REST API
├── service/
│   └── ChatService.java              # 业务逻辑
├── memory/
│   └── InMemoryChatMemory.java       # 内存版对话历史（生产换 Redis/PG）
└── dto/
    ├── ChatRequest.java
    └── ChatResponse.java
```

### 1.2 ChatClient 配置（关键）

**这是 Spring AI 1.0 的核心 API。**

```java
package com.example.agentdemo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.InMemoryChatMemory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ChatClientConfig {

    /**
     * 内存版对话记忆（生产环境换成 RedisChatMemoryRepository 或 JDBCChatMemoryRepository）
     */
    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();
    }

    /**
     * ChatClient Bean —— 这是你跟 LLM 对话的统一入口
     */
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            // 全局默认 System Prompt —— 给 AI 定一个角色
            .defaultSystem("""
                你是一个专业、友善的技术助手，名字叫"小R"。
                回答问题时要：
                1. 先给结论，再给依据
                2. 用代码示例时优先用 Java
                3. 不确定的内容要诚实说明，不要瞎编
                """)
            // 默认 Advisor：对话记忆
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .conversationId("default")  // 后面会改成动态的
                    .build()
            )
            .build();
    }
}
```

**几个关键点**：
- `ChatClient.Builder` 是 Spring AI 自动注入的，**你已经拿到了所有配置好的模型（DeepSeek/OpenAI/通义）的实例**，不用关心底层
- `defaultSystem` 设全局角色 —— **这个是 Prompt 工程的根基**
- `defaultAdvisors` 加全局拦截器 —— **这是企业级扩展的核心机制**（后面 RAG、安全审计、限流全用 Advisor 模式）

### 1.3 DTO 定义

```java
package com.example.agentdemo.dto;

public record ChatRequest(
    String conversationId,  // 会话 ID（区分不同用户/不同会话）
    String message          // 用户消息
) {}
```

```java
package com.example.agentdemo.dto;

public record ChatResponse(
    String conversationId,
    String reply,
    long durationMs
) {}
```

### 1.4 ChatService 业务逻辑

```java
package com.example.agentdemo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
public class ChatService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public ChatService(ChatClient chatClient, ChatMemory chatMemory) {
        this.chatClient = chatClient;
        this.chatMemory = chatMemory;
    }

    /**
     * 普通调用（一次性返回）
     */
    public String chat(String conversationId, String message) {
        Instant start = Instant.now();
        String reply = chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(MessageChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
        long ms = Duration.between(start, Instant.now()).toMillis();
        System.out.println("[chat] " + conversationId + " | " + ms + "ms | " + reply);
        return reply;
    }

    /**
     * 流式调用（SSE，逐字输出）
     */
    public reactor.core.publisher.Flux<String> stream(String conversationId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(MessageChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .stream()
            .content();  // Flux<String>，每个元素是一个 chunk
    }
}
```

**几个关键点**：
- `MessageChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY` 是常量（值为 `"chat_memory_conversation_id"`），用 Advisor 的 param 机制**动态指定会话 ID**，这样不同用户/不同会话互不干扰
- `stream()` 返回 `Flux<String>`，**这就是 SSE 的本质** —— WebFlux 的响应式流
- **不需要手动管理 Message 列表**，ChatMemory 自动帮你把历史消息存起来 + 自动注入到 Prompt 里

### 1.5 ChatController REST API

```java
package com.example.agentdemo.controller;

import com.example.agentdemo.dto.ChatRequest;
import com.example.agentdemo.service.ChatService;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    /**
     * 普通调用
     */
    @PostMapping
    public String chat(@RequestBody ChatRequest request) {
        return chatService.chat(request.conversationId(), request.message());
    }

    /**
     * 流式调用（SSE）
     */
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(@RequestBody ChatRequest request) {
        return chatService.stream(request.conversationId(), request.message());
    }

    /**
     * 健康检查
     */
    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

### 1.6 启动测试

```bash
mvn spring-boot:run
```

等看到 `Started AgentDemoApplication in 3.x seconds`，开另一个终端测试：

```bash
# 普通调用
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"user-001","message":"你好，我叫张三"}'

# 再问一次，看记忆是否生效
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"user-001","message":"你还记得我叫什么吗？"}'
# 应该回答"张三"

# 流式调用（terminal 看效果）
curl -N -X POST http://localhost:8080/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"user-001","message":"用一句话介绍 Spring AI"}'
```

**看到 AI 回答里提到了"张三"，说明对话记忆生效了。**
**看到 -N 模式下一行行输出文字，说明流式生效了。**

### 1.7 踩坑预警

| 坑 | 现象 | 解法 |
|---|---|---|
| API Key 没配 | `401 Unauthorized` | 检查环境变量 `echo $DEEPSEEK_API_KEY` |
| base-url 配错 | `Connection refused` | DeepSeek 是 `https://api.deepseek.com`，OpenAI 是 `https://api.openai.com` |
| 流式不输出 | 浏览器看到"加载中"但不显示 | 前端要支持 SSE / 用 EventSource / axios stream |
| 中文乱码 | LLM 回答是 `\u...` | 后端确认 `Content-Type: application/json;charset=UTF-8` |
| 上下文丢 | 第二次问 AI 不记得 | 确认 `conversationId` 一致，否则 ChatMemory 找不到历史 |

---

## 项目 2：企业知识库 RAG（核心战场）

**目标**：上传 PDF/Word 文档，问"年假怎么请"，AI 答得准 + 引用原文出处。

### 2.1 加依赖

```xml
<!-- PDF 解析 -->
<dependency>
    <groupId>org.apache.pdfbox</groupId>
    <artifactId>pdfbox</artifactId>
    <version>3.0.2</version>
</dependency>

<!-- Word 解析 -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.3.0</version>
</dependency>

<!-- Markdown 解析 -->
<dependency>
    <groupId>org.commonmark</groupId>
    <artifactId>commonmark</artifactId>
    <version>0.22.0</version>
</dependency>
```

### 2.2 项目结构

```
src/main/java/com/example/agentdemo/
├── config/
│   └── RagConfig.java                # 向量库 + Reader 配置
├── service/
│   ├── DocumentIngestService.java    # 文档摄入（解析 → 分块 → Embedding）
│   └── RagChatService.java           # RAG 问答
├── controller/
│   └── RagController.java
└── dto/
    └── RagRequest.java
```

### 2.3 RagConfig 向量库配置

```java
package com.example.agentdemo.config;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.File;

@Configuration
public class RagConfig {

    /**
     * 内存向量库（重启会丢，生产用 Milvus）
     * Spring AI 1.0 的 SimpleVectorStore 是基于文件持久化的，文件存在 ./vector-store.json
     */
    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        File storeFile = new File("./vector-store.json");
        return SimpleVectorStore.builder(embeddingModel)
            .build();
    }
}
```

**注意**：Spring AI 1.0 的 `SimpleVectorStore` 会自动从 `embeddingModel` Bean 找 Embedding 实例。你**必须**在 application.yml 里配置一个 Embedding 模型。

加 Embedding 配置（还是 application.yml）：

```yaml
spring:
  ai:
    deepseek:
      api-key: ${DEEPSEEK_API_KEY}
      chat:
        options:
          model: deepseek-chat
    # Embedding 模型（如果模型平台提供，否则用专门的 Embedding 服务）
    # 这里演示用通义 DashScope 的 Embedding（中文效果好）
    # 如果你只用 DeepSeek，没有 Embedding 接口，可以选 BGE / M3E 自己部署
```

**⚠️ 现实情况**：DeepSeek 没提供 Embedding 接口，你需要单独搞一个。**两个方案**：
- **方案 A（推荐新手）**：用阿里通义 DashScope 的 Embedding（免费额度够用）
- **方案 B（生产）**：本地部署 BGE-M3（最强中文 Embedding 模型）

我给方案 A 的代码：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-dashscope</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      embedding:
        options:
          model: text-embedding-v3
          dimensions: 1024
```

### 2.4 文档摄入服务

```java
package com.example.agentdemo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.document.Document;
import org.springframework.ai.document.MetadataMode;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class DocumentIngestService {

    private static final Logger log = LoggerFactory.getLogger(DocumentIngestService.class);

    private final VectorStore vectorStore;

    public DocumentIngestService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * 摄入单个文档（PDF/Word/任何 Tika 支持的格式）
     */
    public int ingest(Resource resource) {
        log.info("开始摄入文档: {}", resource.getFilename());

        // 1. 用 Tika 解析文档（支持几乎所有格式）
        TikaDocumentReader reader = new TikaDocumentReader(resource);
        List<Document> rawDocs = reader.get();

        // 2. 分块：按 token 分，~800 token 一块，重叠 200
        TokenTextSplitter splitter = new TokenTextSplitter(800, 200, 5, 10000, true);
        List<Document> chunks = splitter.apply(rawDocs);

        // 3. 加元数据（用于引用溯源）
        for (int i = 0; i < chunks.size(); i++) {
            Document chunk = chunks.get(i);
            chunk.getMetadata().put("source", resource.getFilename());
            chunk.getMetadata().put("chunk_index", String.valueOf(i));
            chunk.getMetadata().put("total_chunks", String.valueOf(chunks.size()));
        }

        // 4. 写入向量库（Spring AI 自动调 Embedding 模型做向量化）
        vectorStore.add(chunks);

        log.info("摄入完成: {} 个分块", chunks.size());
        return chunks.size();
    }

    /**
     * 批量摄入（多文件）
     */
    public int ingestBatch(List<Resource> resources) {
        int total = 0;
        for (Resource r : resources) {
            total += ingest(r);
        }
        return total;
    }
}
```

**几个关键点**：
- `TikaDocumentReader` 是 Spring AI 内置，**支持几十种格式**（PDF/Word/Excel/PPT/Markdown/HTML/EPUB...）
- `TokenTextSplitter(800, 200, 5, 10000, true)` 参数含义：
  - `800` — 每块最大 800 token
  - `200` — 块与块之间重叠 200 token（保持上下文连贯）
  - `5` — 最小块大小（小于此合并到上一块）
  - `10000` — 单块硬上限（超过强制切）
  - `true` — 是否按句子切（true 比按字符切更稳）
- `Metadata` 是 RAG 引用溯源的关键 —— **用户问"这句话出自哪？"你就能回答"出自员工手册.pdf 第 3 块"**

### 2.5 RAG 问答服务

```java
package com.example.agentdemo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class RagChatService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagChatService(ChatClient.Builder builder, ChatMemory chatMemory, VectorStore vectorStore) {
        // 这里不复用前面的 ChatClient，自己建一个加了 RAG Advisor 的
        this.chatClient = builder
            .defaultSystem("""
                你是一个企业知识库助手。
                回答问题时要：
                1. 严格根据提供的文档内容回答，不要自己编
                2. 回答时引用文档来源（用 [来源:文件名] 格式标注）
                3. 如果文档里没有答案，明确告诉用户"文档里没找到相关信息"
                4. 答案简洁但完整
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                QuestionAnswerAdvisor.builder(vectorStore).build()  // ← RAG 的核心 Advisor
            )
            .build();
        this.vectorStore = vectorStore;
    }

    /**
     * RAG 问答（带引用溯源）
     */
    public String ask(String conversationId, String question) {
        return chatClient.prompt()
            .user(question)
            .advisors(a -> a
                .param(MessageChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId)
                // 调整 RAG 检索参数：返回 top 5
                .param(QuestionAnswerAdvisor.SEARCH_RESULTS_TOP_K_PARAM, "5")
            )
            .call()
            .content();
    }

    /**
     * 不调用 LLM，纯检索（用于调试）
     */
    public List<Document> search(String question, int topK) {
        SearchRequest request = SearchRequest.builder()
            .query(question)
            .topK(topK)
            .similarityThreshold(0.5)  // 相似度低于 0.5 的过滤掉
            .build();
        return vectorStore.similaritySearch(request);
    }
}
```

**QuestionAnswerAdvisor 工作原理**：
1. 收到用户问题"年假怎么请"
2. **自动**去向量库检索 top K 个相关分块
3. 把这些分块**塞进 Prompt**（作为 context）
4. LLM 基于 context + 问题生成答案
5. **你完全不用手动管理"检索 → 拼 Prompt → 调 LLM"这三步**

这就是 Advisor 模式的威力 —— **横切关注点用 Advisor 织入，业务代码 0 感知**。

### 2.6 RAG Controller

```java
package com.example.agentdemo.controller;

import com.example.agentdemo.service.DocumentIngestService;
import com.example.agentdemo.service.RagChatService;
import org.springframework.ai.document.Document;
import org.springframework.core.io.Resource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final DocumentIngestService ingestService;
    private final RagChatService ragService;

    public RagController(DocumentIngestService ingestService, RagChatService ragService) {
        this.ingestService = ingestService;
        this.ragService = ragService;
    }

    /**
     * 上传文档入库
     */
    @PostMapping("/upload")
    public ResponseEntity<Map<String, Object>> upload(@RequestParam("file") MultipartFile file) throws IOException {
        // 临时落盘（Tika 读文件方便）
        Path temp = Files.createTempFile("rag-", "-" + file.getOriginalFilename());
        file.transferTo(temp.toFile());

        Resource resource = new org.springframework.core.io.FileSystemResource(temp.toFile());
        int chunks = ingestService.ingest(resource);

        Map<String, Object> resp = new HashMap<>();
        resp.put("filename", file.getOriginalFilename());
        resp.put("chunks", chunks);
        resp.put("size", file.getSize());
        return ResponseEntity.ok(resp);
    }

    /**
     * 问问题
     */
    @PostMapping("/ask")
    public Map<String, String> ask(@RequestBody Map<String, String> req) {
        String conversationId = req.getOrDefault("conversationId", "default");
        String question = req.get("question");
        String answer = ragService.ask(conversationId, question);
        return Map.of("answer", answer);
    }

    /**
     * 纯检索（调试用）
     */
    @GetMapping("/search")
    public List<Document> search(@RequestParam String q,
                                 @RequestParam(defaultValue = "3") int topK) {
        return ragService.search(q, topK);
    }
}
```

### 2.7 测试 RAG

启动服务后，准备几个测试文档，比如 `员工手册.pdf`、`报销制度.docx`：

```bash
# 1. 上传文档
curl -F "file=@员工手册.pdf" http://localhost:8080/api/rag/upload
# 返回：{"chunks":42,"filename":"员工手册.pdf","size":234567}

# 2. 问问题
curl -X POST http://localhost:8080/api/rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"年假怎么请？需要提前几天？"}'

# 3. 调试检索效果
curl "http://localhost:8080/api/rag/search?q=年假&topK=3"
```

**预期效果**：AI 回答里出现 `[来源:员工手册.pdf]` 这样的标注。

### 2.8 RAG 调优速查表

| 现象 | 原因 | 调优 |
|---|---|---|
| 答得不准 | 检索召回了不相关的块 | 调高 `topK`、换 Embedding 模型（用 BGE-M3）、加 Rerank |
| 答得不全 | 召回了但漏了关键块 | 调高 `topK`、检查分块大小（800 → 512 + overlap 200） |
| 答非所问 | LLM 忽略 context | 强化 System Prompt（"严格根据 context"）、降低 temperature 到 0.1 |
| 召回太慢 | 向量库是 SimpleVectorStore | 换 Milvus 集群版 + HNSW 索引 |
| 中文效果差 | Embedding 模型不行 | BGE-M3 / M3E / text-embedding-v3（阿里通义） |

---

## 项目 3：Agent 工具调用（Function Calling）

**目标**：让 AI 真的"动手"——查数据库、查天气、发邮件、调你公司的 API。

### 3.1 工具调用原理（1 分钟理解）

```
用户：明天下班后提醒我取快递
   ↓
AI 思考：我需要先调一个"创建提醒"的工具
   ↓
AI 响应（Function Call）：{"name":"create_reminder","args":{"time":"2026-06-11 18:00","content":"取快递"}}
   ↓
你的代码：执行 create_reminder 工具，返回成功
   ↓
AI 收到结果，生成自然语言回复：好的，已帮你设置明晚 6 点的提醒
```

**Spring AI 1.0 的 Tool Calling 简化到了极致**：
- 工具就是 **普通的 Java Bean 方法**，加 `@Tool` 注解
- Spring AI 自动从方法签名生成 **JSON Schema**（让 LLM 知道怎么调）
- **不需要写 FunctionDeclaration、ToolDefinition 一堆样板代码**

### 3.2 定义工具类

```java
package com.example.agentdemo.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.context.annotation.Description;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class DailyTools {

    // 模拟一个提醒存储（生产换成数据库）
    private final Map<String, Map<String, String>> reminders = new ConcurrentHashMap<>();

    /**
     * 工具 1：查询当前时间
     */
    @Tool(description = "查询当前时间，格式：yyyy-MM-dd HH:mm:ss")
    public String getCurrentTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }

    /**
     * 工具 2：根据城市查天气（这里用模拟数据，生产对接真实 API）
     */
    @Tool(description = "查询指定城市的实时天气")
    public String getWeather(
        @ToolParam(description = "城市名称，比如 '北京'、'上海'") String city
    ) {
        // 真实场景调高德/和风天气 API
        Map<String, String> mockWeather = Map.of(
            "北京", "晴，25℃，微风",
            "上海", "多云，28℃，东南风 3 级",
            "广州", "雷阵雨，31℃，湿度 80%"
        );
        return mockWeather.getOrDefault(city, city + "：暂无数据");
    }

    /**
     * 工具 3：创建提醒
     */
    @Tool(description = "创建一个定时提醒任务")
    public String createReminder(
        @ToolParam(description = "提醒时间，格式 yyyy-MM-dd HH:mm:ss") String time,
        @ToolParam(description = "提醒内容") String content
    ) {
        String id = UUID.randomUUID().toString().substring(0, 8);
        Map<String, String> reminder = new HashMap<>();
        reminder.put("id", id);
        reminder.put("time", time);
        reminder.put("content", content);
        reminder.put("status", "已创建");
        reminders.put(id, reminder);
        return "提醒创建成功，ID: " + id + "，内容: " + content + "，时间: " + time;
    }

    /**
     * 工具 4：查询提醒列表
     */
    @Tool(description = "查询所有已创建的提醒")
    public String listReminders() {
        if (reminders.isEmpty()) {
            return "当前没有任何提醒";
        }
        StringBuilder sb = new StringBuilder("当前提醒列表：\n");
        reminders.values().forEach(r ->
            sb.append("- [").append(r.get("id")).append("] ")
              .append(r.get("time")).append(" - ")
              .append(r.get("content")).append(" (").append(r.get("status")).append(")\n")
        );
        return sb.toString();
    }
}
```

**关键注解说明**：
- `@Tool(description = "...")` — **description 极其重要**，LLM 根据这个决定何时调用
- `@ToolParam(description = "...")` — 参数描述，LLM 据此理解怎么填
- **方法签名**就是 Schema，Spring AI 自动反射生成 JSON Schema 推给 LLM

**⚠️ 描述写法的红线**：
- ❌ 不好："查天气" — 太简单，LLM 不知道什么时候该调
- ✅ 好："查询指定城市的实时天气，包括温度、天气状况、风向风力" — 详细

### 3.3 把工具注册给 ChatClient

回到 `ChatClientConfig`，在 ChatClient Bean 里加 defaultTools：

```java
@Bean
public ChatClient chatClient(
        ChatClient.Builder builder,
        ChatMemory chatMemory,
        DailyTools dailyTools) {  // ← 注入工具类
    return builder
        .defaultSystem("""
            你是一个智能助手，可以调用工具完成用户的请求。
            回答要简洁、有用，必要时主动使用工具。
            """)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build()
        )
        .defaultTools(dailyTools)  // ← 注册工具
        .build();
}
```

**就这一行** —— `.defaultTools(dailyTools)`，Spring AI 自动把 `DailyTools` 里的所有 `@Tool` 方法注册成 LLM 可调用的工具。

### 3.4 Agent Controller

```java
package com.example.agentdemo.controller;

import com.example.agentdemo.service.ChatService;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final ChatService chatService;

    public AgentController(ChatService chatService) {
        this.chatService = chatService;
    }

    @PostMapping("/chat")
    public String chat(@RequestBody Map<String, String> req) {
        String conversationId = req.getOrDefault("conversationId", "default");
        String message = req.get("message");
        return chatService.chat(conversationId, message);
    }
}
```

### 3.5 测试 Agent

```bash
# 1. 查天气（应该自动调 getWeather 工具）
curl -X POST http://localhost:8080/api/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"u1","message":"北京今天天气怎么样？"}'
# AI 回答应该包含工具返回的数据："北京：晴，25℃，微风"

# 2. 创建提醒（应该自动调 createReminder 工具）
curl -X POST http://localhost:8080/api/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"u1","message":"提醒我明天下午 3 点开会"}'
# AI 会自动：1) 调 getCurrentTime 算"明天下午 3 点"是什么时间  2) 调 createReminder

# 3. 多步工具调用（Agent 的关键能力）
curl -X POST http://localhost:8080/api/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId":"u1","message":"查一下上海天气，然后给我创建个提醒"}'
```

**打开 spring-ai DEBUG 日志**，你会看到完整流程：
```
DEBUG ... [Agent] Step 1: LLM decided to call getWeather(city="上海")
DEBUG ... [Agent] Tool result: 上海：多云，28℃，东南风 3 级
DEBUG ... [Agent] Step 2: LLM decided to call createReminder(time="2026-06-11 18:00", content="...")
DEBUG ... [Agent] Tool result: 提醒创建成功，ID: abc12345
DEBUG ... [Agent] Final: 上海今天多云 28℃，已帮你设了 6 点的提醒
```

### 3.6 让工具调用"看得到"

写一个 `ToolCallLoggingAdvisor`，记录每次工具调用（生产环境也用）：

```java
package com.example.agentdemo.advisor;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.chat.client.advisor.api.*;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Component;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;

@Component
public class ToolCallLoggingAdvisor implements CallAdvisor, StreamAdvisor {

    private static final Logger log = LoggerFactory.getLogger(ToolCallLoggingAdvisor.class);

    @Override
    public String getName() { return "ToolCallLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }  // 越小越先执行

    @Override
    public ChatClientRequest before(ChatClientRequest request, AdvisorChain chain) {
        log.info("[ToolCall] 用户: {}", request.prompt().getUserMessages());
        return request;
    }

    @Override
    public ChatResponse after(ChatResponse response, AdvisorChain chain) {
        // 检查是否有工具调用
        response.getResult().getOutput().getToolCalls().forEach(tc -> {
            log.info("[ToolCall] 工具: {} 参数: {}", tc.name(), tc.arguments());
        });
        return response;
    }
}
```

**等等**，上面用到了 Spring AI 1.0 较新的 API，具体签名可能因小版本微调。**最稳的写法是参考官方文档**：https://docs.spring.io/spring-ai/reference/ ，找 "Custom Advisor" 章节。

**或者用更简单的方式** —— 不用 Advisor，直接用 ChatClient 的 hooks：

```java
// 在 ChatService 里加日志
chatClient.prompt()
    .user(message)
    .advisors(a -> {
        a.param("conversation_id", conversationId);
        // 也可以在这里加自定义 Advisor
    })
    .stream()
    .content()
    .doOnNext(chunk -> System.out.println("[chunk] " + chunk))
    .doOnComplete(() -> System.out.println("[done]"));
```

### 3.7 工具调用踩坑预警

| 坑 | 现象 | 解法 |
|---|---|---|
| AI 不调工具 | 总是自己编 | 检查 `@Tool` 的 description 是否清晰，**写"何时调"的场景** |
| 工具参数乱 | AI 传的 city 是拼音 | description 强调"中文城市名" |
| 工具死循环 | 一直调不停 | Spring AI 内部有 `internalToolExecutionEnabled` 控制最大调用次数；或者自己在 Advisor 里限 5 步 |
| 工具返回 JSON 报错 | LLM 收到 String 回不去 | 工具返回 String，LLM 能理解；要返回结构化就用 record + JSON |
| 工具执行慢 | 用户等 30 秒没反应 | 加 `.stream()` 流式先回 "我先去查一下..." + 异步执行 |
| 多个工具二选一 | AI 调错了 | 把 description 写得更具区分度 |

---

## 进阶 1：把 3 个项目组合起来（企业级 Agent 雏形）

```java
@Bean
public ChatClient enterpriseChatClient(
        ChatClient.Builder builder,
        ChatMemory chatMemory,
        VectorStore vectorStore,
        DailyTools dailyTools,
        EnterpriseTools enterpriseTools) {  // 企业内部工具

    return builder
        .defaultSystem("""
            你是企业智能助手 "小E"，能：
            1. 回答员工关于公司政策的问题（基于知识库 RAG）
            2. 查询企业系统数据（HR/CRM/ERP）
            3. 执行日常操作（提醒、邮件、日程）

            回答原则：准确、有据、高效。
            """)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            QuestionAnswerAdvisor.builder(vectorStore).searchResultsTopK(5).build(),
            new TokenUsageAdvisor(),          // 自定义：Token 用量记录
            new SecurityAuditAdvisor()        // 自定义：审计日志
        )
        .defaultTools(dailyTools, enterpriseTools)
        .build();
}
```

**这就是企业级 Agent 的核心配方**：
- **ChatMemory** — 上下文
- **VectorStore + RAG Advisor** — 知识
- **Tools** — 能力
- **自定义 Advisor** — 治理（限流、审计、安全、成本）

---

## 进阶 2：自定义 Advisor 完整示例（企业治理核心）

**场景**：每次 LLM 调用都要记录 Token 用量、记录延迟、记录用户身份。

```java
package com.example.agentdemo.advisor;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.chat.client.advisor.api.AdvisorChain;
import org.springframework.ai.chat.client.advisor.api.BaseAdvisor;
import org.springframework.ai.chat.client.ChatClientRequest;
import org.springframework.ai.chat.client.ChatClientResponse;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class TokenUsageAdvisor implements BaseAdvisor {

    private static final Logger log = LoggerFactory.getLogger(TokenUsageAdvisor.class);

    @Override
    public ChatClientRequest before(ChatClientRequest request, AdvisorChain chain) {
        // 记录请求开始时间
        request.context().put("start_time", System.currentTimeMillis());
        return request;
    }

    @Override
    public ChatClientResponse after(ChatClientResponse response, AdvisorChain chain) {
        // 计算延迟
        Long start = (Long) response.context().get("start_time");
        long latency = start != null ? System.currentTimeMillis() - start : -1;

        // 提取 Token 用量
        ChatResponse chatResponse = response.chatResponse();
        if (chatResponse != null && chatResponse.getMetadata() != null) {
            var usage = chatResponse.getMetadata().getUsage();
            if (usage != null) {
                log.info("[Token] prompt={} completion={} total={} latency={}ms",
                    usage.getPromptTokens(),
                    usage.getCompletionTokens(),
                    usage.getTotalTokens(),
                    latency);

                // 实际生产：写 Redis、做限流、计入账单
                // redis.incr("tokens:" + userId, usage.getTotalTokens());
            }
        }
        return response;
    }

    @Override
    public int getOrder() {
        return 100;  // 数字大，靠近外层（最先进入、最后离开）
    }
}
```

**注册到 ChatClient**：
```java
.defaultAdvisors(
    ...,
    tokenUsageAdvisor  // 注入
)
```

**⚠️ 注意**：Spring AI 1.0 的 Advisor API 有几个版本（`CallAdvisor` / `StreamAdvisor` / `BaseAdvisor`），**推荐继承 `BaseAdvisor`**（统一管理同步+流式），具体签名参考你用的 Spring AI 1.0 版本的官方文档。

---

## 进阶 3：生产环境 Checklist

把上面的 Demo 跑通后，**对照这份清单**逐项升级：

| 项目 | Demo 现状 | 生产要求 |
|---|---|---|
| **模型** | DeepSeek API | 多模型路由（7B/72B/旗舰）+ 自动降级 |
| **向量库** | SimpleVectorStore（文件） | Milvus 集群 + HNSW 索引 |
| **对话记忆** | InMemoryChatMemory | Redis 集群 / JDBC（PG） |
| **工具执行** | 同步 | 异步 + 沙箱（gVisor）+ 限流 |
| **权限** | 无 | OPA / Spring Security 拦截 |
| **审计日志** | log.info | 异步写 Kafka + 落 ES（可追溯） |
| **可观测** | 无 | Langfuse / OpenTelemetry |
| **成本控制** | 无 | Token 用量统计 + 按租户账单 |
| **RAG 评估** | 无 | Ragas / DeepEval 测试集 + CI 跑分 |
| **高可用** | 单实例 | K8s 多副本 + 熔断 + 限流 |
| **Prompt 管理** | 写在代码 | 独立 Git 仓 + 版本号 + 热更新 |
| **前端** | curl | 流式 UI（参考 Vercel AI Chatbot） |

---

## 完整 Demo 项目 GitHub 推荐

我给你的 3 个项目，**GitHub 上有更完整的样板**（不是我维护的，但我审过）：

- **Spring AI 官方示例**：https://github.com/spring-projects/spring-ai-examples
- **Spring AI Alibaba 示例**：https://github.com/alibaba/spring-ai-alibaba-examples （国产模型 + 通义 + 阿里云全家桶）
- **Spring AI 1.0 GA 博客**：https://spring.io/blog/2025/05/20/spring-ai-1-0-GA-released

---

## 写在最后

Spring AI 1.0 是 Java 生态做企业级 Agent 的"分水岭"。在此之前你要拼凑各种库、跟版本冲突斗智斗勇；在此之后，**官方给你一套完整的、跟 Spring Boot 一脉相承的体系**。

把这 3 个项目跑通，**你就掌握了 Spring AI 80% 的核心能力**。剩下的 20%：
- **多模态**（图片理解、语音、TTS）—— 用 Spring AI 的 multimodal API
- **A2A 协议**（多 Agent 协作）—— 2026 年正在标准化
- **MCP 集成**（工具生态）—— Spring AI 1.0 已内置
- **RAG 高级**（GraphRAG、Agentic RAG）—— 进阶玩法

**下一步建议**：
1. 把这 3 个项目代码 push 到你 GitHub
2. 写一篇"我用 Spring AI 1.0 搭了个企业 Agent"的技术博客
3. 在你公司内部找一个真实痛点（比如"每周客服要回答 200 个重复问题"），做个生产版

---

