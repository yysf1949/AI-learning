# 深挖版 9：MCP 协议——Agent 工具生态的"USB 接口"

> 日期：2026-06-10
> 配套基础版 + 深挖版 1/2/3/4/5/6/7/8
> 适合：负责 Agent 工具系统、第三方集成的工程师

---

## 写在前面：为什么 MCP 是"Agent 时代的 USB"？

**先说个真实场景**。

你想让 Agent 能"查公司数据库"。你写了 100 行 Function Calling 代码。

**下个月**，你想让 Agent 也能"查 GitLab"。你又写了 100 行。

**又过一个月**，你想让 Agent 也能"发飞书消息"。**你又又又写了 100 行**。

**3 个月后，你有 30 个工具、3000 行胶水代码、1 个 500 行的"工具注册表"、3 个工程师维护。**

**然后你想换个 Agent 框架**（比如从 Spring AI 换到 LangGraph）——

**所有 3000 行胶水代码全部要重写**。💀

**MCP（Model Context Protocol）就是为解决这个问题而生的**。

**类比**：
- **USB** 让不同厂商的设备能接同一台电脑
- **MCP** 让不同厂商的工具能接同一个 Agent

**MCP 是 Anthropic 2024 年 11 月发布、2026 年已经成为 Agent 工具生态事实标准的协议**。

**Spring AI 1.0 GA 原生集成 MCP**（详见深挖版 1）。

**这一篇我把 MCP 讲透**——从协议设计、Server/Client 实现、跟 Function Calling 对比、Java 完整实战。

---

## 第一部分：MCP 是什么

### 1.1 官方定义

**MCP（Model Context Protocol）**：
> **An open protocol that standardizes how applications provide context to LLMs.**

翻译：**一个开放协议，标准化应用如何给 LLM 提供上下文。**

**关键 3 个词**：
- **Open（开放）**——开源协议
- **Standardizes（标准化）**——一次实现，到处运行
- **Context（上下文）**——不只是工具调用，还包括资源、Prompt 模板

### 1.2 MCP vs Function Calling（最容易混）

| 维度 | Function Calling | MCP |
|---|---|---|
| **范围** | 工具调用 | 工具 + 资源 + Prompt |
| **协议** | 各厂商自定义 | 统一协议 |
| **可移植性** | ❌ 换框架重写 | ✅ 一次实现到处用 |
| **发现** | 手动注册 | 自动发现（list）|
| **跨语言** | 各语言 SDK 各一套 | 统一协议，跨语言 |
| **生态** | 各家孤岛 | 共享生态 |

**举个例子**：
- **Function Calling**：你写了一个查询天气的 function，绑给 OpenAI 的 function calling。
- **MCP**：你写了一个 weather MCP server，**任何支持 MCP 的 Agent（Claude / Cursor / Spring AI / Cline）都能自动发现并调用**。

### 1.3 MCP 三大核心能力

```
┌──────────────────────────────────────────┐
│ 1. Tools（工具）                            │
│    → Agent 能调用的函数                     │
│    → 类比 USB 设备的"功能"                 │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ 2. Resources（资源）                        │
│    → Agent 能读取的数据                    │
│    → 文件、数据库记录、API 响应             │
│    → 类比 U 盘里的"文件"                  │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ 3. Prompts（提示词模板）                    │
│    → 预定义的 Prompt 模板                   │
│    → 用户可以触发的快捷指令                 │
│    → 类比 IDE 的"代码片段"                │
└──────────────────────────────────────────┘
```

---

## 第二部分：MCP 架构

### 2.1 三大角色

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  MCP Host    │ ←────→  │  MCP Client  │ ←────→  │  MCP Server  │
│  (Claude /   │         │  (Agent 内部) │         │  (工具提供方)  │
│   Cursor)    │         │              │         │              │
└──────────────┘         └──────────────┘         └──────────────┘
   IDE / 应用              协议客户端               工具/数据源
```

- **MCP Host**：用户用的应用（Claude Desktop、Cursor、Spring AI 应用）
- **MCP Client**：Host 内部的协议客户端（一个 Host 可以连多个 Server）
- **MCP Server**：提供工具/资源的服务（你自己写的、或者社区的）

### 2.2 通信方式

**MCP 用 JSON-RPC 2.0 over**：
- **stdio**（本地进程通信，最常用）
- **HTTP + SSE**（远程服务）
- **WebSocket**（未来）

**stdio 模式**（本地工具，最常见）：
```bash
# Claude Desktop 配置
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/docs"]
    }
  }
}
```

**HTTP 模式**（远程服务）：
```
http://mcp-server.company.com:3000/sse
```

### 2.3 完整能力协商

MCP 启动时，Client 和 Server 会**协商能力**：

```
Client: "我支持 resources / tools / prompts，能用 stdio 和 SSE"
Server: "我提供 resources + tools，能用 stdio"
   ↓
协商结果：双方能力交集
   ↓
开始用：tools/call, resources/read, prompts/get
```

---

## 第三部分：MCP 协议规范（核心）

### 3.1 三大原语

#### 1. Tools（工具）

**Server 声明工具**：

```json
{
  "name": "query_customer",
  "description": "查询客户信息，根据客户 ID 或姓名",
  "inputSchema": {
    "type": "object",
    "properties": {
      "customer_id": {
        "type": "string",
        "description": "客户 ID"
      },
      "include_orders": {
        "type": "boolean",
        "default": true,
        "description": "是否包含历史订单"
      }
    }
  }
}
```

**Client 调用**：

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "query_customer",
    "arguments": {
      "customer_id": "C-001",
      "include_orders": true
    }
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"name\":\"Acme Corp\",\"orders\":[...]}"
      }
    ]
  }
}
```

#### 2. Resources（资源）

**Server 声明资源**：

```json
{
  "uri": "file:///docs/employee-handbook.pdf",
  "name": "员工手册",
  "description": "公司员工手册 PDF",
  "mimeType": "application/pdf"
}
```

**Client 读取**：

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///docs/employee-handbook.pdf"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///docs/employee-handbook.pdf",
        "mimeType": "application/pdf",
        "blob": "base64-encoded-pdf-data..."
      }
    ]
  }
}
```

#### 3. Prompts（Prompt 模板）

**Server 声明 Prompt**：

```json
{
  "name": "code_review",
  "description": "代码审查助手",
  "arguments": [
    {
      "name": "language",
      "description": "编程语言",
      "required": true
    },
    {
      "name": "code",
      "description": "代码内容",
      "required": true
    }
  ]
}
```

**Client 使用**：

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "language": "Java",
      "code": "public class ..."
    }
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "description": "代码审查 Prompt",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "请审查以下 Java 代码：\n\n```java\n...\n```\n\n请关注：\n1. 代码风格\n2. 性能\n3. 安全性"
        }
      }
    ]
  }
}
```

### 3.2 关键方法清单

| 方法 | 方向 | 用途 |
|---|---|---|
| `initialize` | C → S | 能力协商 |
| `tools/list` | C → S | 列出所有工具 |
| `tools/call` | C → S | 调用工具 |
| `resources/list` | C → S | 列出资源 |
| `resources/read` | C → S | 读取资源 |
| `resources/subscribe` | C → S | 订阅资源变化 |
| `prompts/list` | C → S | 列出 Prompt 模板 |
| `prompts/get` | C → S | 获取 Prompt |
| `notifications/tools/list_changed` | S → C | 工具列表变了 |
| `notifications/resources/updated` | S → C | 资源更新了 |

### 3.3 错误处理

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32602,
    "message": "Invalid params: customer_id is required",
    "data": {
      "missing_field": "customer_id"
    }
  }
}
```

**标准 JSON-RPC 错误码**：

| Code | 含义 |
|---|---|
| -32700 | Parse error |
| -32600 | Invalid request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 ~ -32099 | Server-defined errors |

---

## 第四部分：Java 实现 MCP Server

### 4.1 Spring AI MCP 集成（推荐）

**Spring AI 1.0 GA 内置 MCP 支持**——比从零写快 10 倍。

#### Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server</artifactId>
</dependency>
```

#### 完整 MCP Server 实现

```java
package com.example.mcp;

import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpResource;
import org.springframework.ai.mcp.annotation.McpPrompt;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Component;

@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}
```

#### Tool 实现

```java
package com.example.mcp.tool;

import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CustomerTools {

    @Autowired private CustomerService customerService;

    /**
     * MCP Tool 1：查询客户
     */
    @McpTool(description = "查询客户信息，根据客户 ID 或姓名。返回客户的档案和历史订单。")
    public CustomerDto queryCustomer(
        @McpParam(description = "客户 ID（与姓名二选一）", required = false) String customerId,
        @McpParam(description = "客户姓名（与 ID 二选一）", required = false) String customerName,
        @McpParam(description = "是否包含历史订单", required = false, defaultValue = "true") boolean includeOrders
    ) {
        if (customerId == null && customerName == null) {
            throw new IllegalArgumentException("必须提供 customer_id 或 customer_name");
        }
        return customerService.query(customerId, customerName, includeOrders);
    }

    /**
     * MCP Tool 2：创建订单
     */
    @McpTool(description = "为指定客户创建新订单。返回订单 ID 和预计金额。")
    public OrderDto createOrder(
        @McpParam(description = "客户 ID", required = true) String customerId,
        @McpParam(description = "产品列表，每个产品包含 product_id 和 quantity", required = true) List<OrderItem> items,
        @McpParam(description = "收货地址", required = true) String address
    ) {
        return orderService.create(customerId, items, address);
    }
}
```

#### Resource 实现

```java
package com.example.mcp.resource;

import org.springframework.ai.mcp.annotation.McpResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

@Component
public class DocumentResources {

    /**
     * MCP Resource：员工手册
     */
    @McpResource(
        uri = "file://docs/employee-handbook.md",
        name = "员工手册",
        description = "公司员工手册完整版",
        mimeType = "text/markdown"
    )
    public String getEmployeeHandbook() {
        return """
            # 公司员工手册

            ## 1. 考勤制度
            每天 9:00-18:00，弹性 1 小时

            ## 2. 年假
            工龄 1-10 年：5 天
            工龄 10-20 年：10 天
            工龄 20 年以上：15 天

            ## 3. 报销
            需在 30 天内提交
            """;
    }

    /**
     * MCP Resource：产品目录
     */
    @McpResource(
        uri = "db://products",
        name = "产品目录",
        description = "所有在售产品的实时数据",
        mimeType = "application/json"
    )
    public String getProductCatalog() {
        return productService.exportJson();
    }
}
```

#### Prompt 实现

```java
package com.example.mcp.prompt;

import org.springframework.ai.mcp.annotation.McpPrompt;
import org.springframework.ai.mcp.annotation.McpArg;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

@Component
public class PromptTemplates {

    /**
     * MCP Prompt：客户跟进
     */
    @McpPrompt(
        name = "customer_followup",
        description = "生成客户跟进邮件"
    )
    public List<Map<String, Object>> customerFollowup(
        @McpArg(name = "customer_name", description = "客户姓名", required = true) String customerName,
        @McpArg(name = "last_interaction", description = "上次互动内容", required = true) String lastInteraction,
        @McpArg(name = "tone", description = "语气：formal / casual", required = false) String tone
    ) {
        String t = tone != null ? tone : "formal";
        return List.of(
            Map.of("role", "user", "content", String.format("""
                请为客户 %s 生成一封跟进邮件。

                上次互动：
                %s

                要求：
                - 语气：%s
                - 长度：150 字以内
                - 包含明确的 next step

                直接输出邮件正文（不要解释）。
                """, customerName, lastInteraction, t))
        );
    }
}
```

#### 启动配置

```yaml
# application.yml
spring:
  ai:
    mcp:
      server:
        # 启用 stdio 模式（默认）
        stdio: true
        # 或启用 HTTP/SSE 模式
        sse:
          enabled: true
          port: 3000
          endpoint: /sse
```

**启动**：
```bash
mvn spring-boot:run
# Server 在 stdio 模式运行，等待 JSON-RPC 请求
```

### 4.2 配置文件

**Claude Desktop 配置**（`~/Library/Application Support/Claude/claude_desktop_config.json`）：

```json
{
  "mcpServers": {
    "company-customer": {
      "command": "java",
      "args": ["-jar", "/path/to/mcp-server.jar"],
      "env": {
        "DATABASE_URL": "jdbc:postgresql://..."
      }
    }
  }
}
```

**启动 Claude Desktop 后**，它会自动：
1. 启动你的 MCP Server
2. 调 `initialize` 协商能力
3. 调 `tools/list` 拿到所有工具
4. 把工具声明告诉 Claude
5. 用户对话时，Claude 自动决定调不调

### 4.3 Spring AI Agent 调用 MCP Server

```java
@Service
public class McpAgentService {

    @Autowired private ChatClient.Builder chatClientBuilder;
    @Autowired private McpClient mcpClient;

    public String chat(String userMessage) {
        // 1. 发现 MCP Server 提供的所有工具
        List<ToolDefinition> mcpTools = mcpClient.listTools();

        // 2. 把 MCP 工具注入 ChatClient
        ChatClient chatClient = chatClientBuilder
            .defaultTools(mcpTools.toArray(new ToolCallback[0]))
            .build();

        // 3. 调用
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

---

## 第五部分：Java 实现 MCP Client

### 5.1 简化版 MCP Client（JSON-RPC over stdio）

```java
package com.example.mcp.client;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Component;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * MCP Client：通过 stdio 跟 MCP Server 通信
 */
@Component
public class McpStdioClient {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private final AtomicLong requestId = new AtomicLong(0);
    private final Map<Long, String> pendingRequests = new ConcurrentHashMap<>();

    private Process serverProcess;
    private BufferedWriter writer;
    private BufferedReader reader;
    private Thread readerThread;

    /**
     * 启动 MCP Server 子进程
     */
    public void start(String command, List<String> args, Map<String, String> env) throws IOException {
        ProcessBuilder pb = new ProcessBuilder(command);
        pb.command().addAll(args);
        if (env != null) pb.environment().putAll(env);
        this.serverProcess = pb.start();

        this.writer = new BufferedWriter(
            new OutputStreamWriter(serverProcess.getOutputStream(), StandardCharsets.UTF_8));
        this.reader = new BufferedReader(
            new InputStreamReader(serverProcess.getInputStream(), StandardCharsets.UTF_8));

        // 启动读取线程
        this.readerThread = new Thread(this::readLoop, "mcp-client-reader");
        readerThread.setDaemon(true);
        readerThread.start();

        // 初始化
        initialize();
    }

    /**
     * 初始化（能力协商）
     */
    public void initialize() throws IOException {
        Map<String, Object> request = Map.of(
            "jsonrpc", "2.0",
            "id", requestId.incrementAndGet(),
            "method", "initialize",
            "params", Map.of(
                "protocolVersion", "2024-11-05",
                "capabilities", Map.of(),
                "clientInfo", Map.of(
                    "name", "java-mcp-client",
                    "version", "1.0.0"
                )
            )
        );
        send(request);
    }

    /**
     * 列出所有工具
     */
    public List<Map<String, Object>> listTools() throws IOException {
        Map<String, Object> request = Map.of(
            "jsonrpc", "2.0",
            "id", requestId.incrementAndGet(),
            "method", "tools/list"
        );
        Map<String, Object> response = sendSync(request);
        return (List<Map<String, Object>>) response.get("tools");
    }

    /**
     * 调用工具
     */
    public Map<String, Object> callTool(String name, Map<String, Object> arguments) throws IOException {
        Map<String, Object> request = Map.of(
            "jsonrpc", "2.0",
            "id", requestId.incrementAndGet(),
            "method", "tools/call",
            "params", Map.of(
                "name", name,
                "arguments", arguments
            )
        );
        return sendSync(request);
    }

    /**
     * 读取资源
     */
    public Map<String, Object> readResource(String uri) throws IOException {
        Map<String, Object> request = Map.of(
            "jsonrpc", "2.0",
            "id", requestId.incrementAndGet(),
            "method", "resources/read",
            "params", Map.of("uri", uri)
        );
        return sendSync(request);
    }

    /**
     * 获取 Prompt 模板
     */
    public Map<String, Object> getPrompt(String name, Map<String, Object> arguments) throws IOException {
        Map<String, Object> request = Map.of(
            "jsonrpc", "2.0",
            "id", requestId.incrementAndGet(),
            "method", "prompts/get",
            "params", Map.of(
                "name", name,
                "arguments", arguments
            )
        );
        return sendSync(request);
    }

    private void send(Map<String, Object> request) throws IOException {
        String json = objectMapper.writeValueAsString(request);
        writer.write(json + "\n");
        writer.flush();
    }

    private Map<String, Object> sendSync(Map<String, Object> request) throws IOException {
        long id = ((Number) request.get("id")).longValue();
        pendingRequests.put(id, "");
        send(request);
        // 同步等待（实际生产用 CompletableFuture）
        // 简化实现：假设响应立即到达
        while (pendingRequests.containsKey(id)) {
            try { Thread.sleep(10); } catch (InterruptedException e) { break; }
        }
        // 从结果中取（这里简化）
        return (Map<String, Object>) pendingRequests.get(id);
    }

    private void readLoop() {
        try {
            String line;
            while ((line = reader.readLine()) != null) {
                Map<String, Object> response = objectMapper.readValue(line, Map.class);
                long id = ((Number) response.get("id")).longValue();
                pendingRequests.put(id, response);  // 简化存储
            }
        } catch (Exception e) {
            // log
        }
    }

    public void close() throws IOException {
        if (serverProcess != null) serverProcess.destroy();
        if (writer != null) writer.close();
        if (reader != null) reader.close();
    }
}
```

### 5.2 在 Spring AI 中集成 MCP Client

```java
@Configuration
public class McpClientConfig {

    @Bean
    public McpStdioClient mcpClient() {
        McpStdioClient client = new McpStdioClient();
        try {
            client.start("java", List.of("-jar", "/path/to/mcp-server.jar"), null);
        } catch (IOException e) {
            throw new RuntimeException("Failed to start MCP server", e);
        }
        return client;
    }

    @Bean
    public ChatClient mcpEnabledChatClient(ChatClient.Builder builder, McpStdioClient mcpClient) {
        // 把 MCP 工具转成 Spring AI 工具
        List<Map<String, Object>> mcpTools = mcpClient.listTools();

        return builder
            .defaultTools(mcpTools.stream()
                .map(this::toSpringAiTool)
                .toArray(ToolCallback[]::new))
            .build();
    }

    private ToolCallback toSpringAiTool(Map<String, Object> mcpTool) {
        return new McpToolCallback(mcpTool);
    }
}
```

---

## 第六部分：MCP 生态——2026 年主流 MCP Servers

**MCP 最大的价值是"生态"**——已经有一堆现成的 MCP Server 可以用。

### 6.1 官方/社区 MCP Server 列表

| 类别 | MCP Server | 用途 | 接入方式 |
|---|---|---|---|
| **文件** | `@modelcontextprotocol/server-filesystem` | 读本地文件 | stdio |
| **数据库** | `@modelcontextprotocol/server-postgres` | 查 PostgreSQL | stdio |
| **数据库** | `@modelcontextprotocol/server-sqlite` | 查 SQLite | stdio |
| **Git** | `@modelcontextprotocol/server-git` | Git 操作 | stdio |
| **GitHub** | `@modelcontextprotocol/server-github` | GitHub API | stdio |
| **GitLab** | `@modelcontextprotocol/server-gitlab` | GitLab API | stdio |
| **Google** | `@modelcontextprotocol/server-gdrive` | Google Drive | stdio |
| **Google** | `@modelcontextprotocol/server-gmail` | Gmail | stdio |
| **搜索** | `@modelcontextprotocol/server-brave-search` | Brave 搜索 | stdio |
| **网页** | `@modelcontextprotocol/server-puppeteer` | 浏览器自动化 | stdio |
| **Slack** | `@modelcontextprotocol/server-slack` | Slack | stdio |
| **记忆** | `@modelcontextprotocol/server-memory` | 知识图谱记忆 | stdio |
| **时序** | `@modelcontextprotocol/server-time` | 时间查询 | stdio |
| **Java SDK** | `spring-ai-starter-mcp-server` | Java 实现 | stdio / SSE |

**这些都是 Anthropic 官方 + 社区维护的，可以直接用。**

### 6.2 自己写一个 Java MCP Server（数据库版）

```java
package com.example.mcp.db;

import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;

@Component
public class DatabaseMcpServer {

    @Autowired private JdbcTemplate jdbc;

    /**
     * 查数据库（只读）
     */
    @McpTool(description = """
        执行 SQL SELECT 查询并返回结果。
        ⚠️ 只支持 SELECT，不允许 INSERT/UPDATE/DELETE。
        一次最多返回 1000 行。
        """)
    public List<Map<String, Object>> query(
        @McpParam(description = "SQL SELECT 查询", required = true) String sql
    ) {
        // 安全检查
        if (!isSelectOnly(sql)) {
            throw new IllegalArgumentException("只支持 SELECT 查询");
        }

        // 限制行数
        if (!sql.toLowerCase().contains("limit")) {
            sql += " LIMIT 1000";
        }

        return jdbc.queryForList(sql);
    }

    /**
     * 列出所有表
     */
    @McpTool(description = "列出数据库中所有表的名称")
    public List<String> listTables() {
        return jdbc.queryForList(
            "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'"
        ).stream()
            .map(row -> (String) row.get("table_name"))
            .collect(Collectors.toList());
    }

    /**
     * 描述表结构
     */
    @McpTool(description = "获取指定表的字段信息")
    public List<Map<String, Object>> describeTable(
        @McpParam(description = "表名", required = true) String tableName
    ) {
        return jdbc.queryForList("""
            SELECT column_name, data_type, is_nullable, column_default
            FROM information_schema.columns
            WHERE table_name = ?
            ORDER BY ordinal_position
            """, tableName);
    }

    private boolean isSelectOnly(String sql) {
        String trimmed = sql.trim().toUpperCase();
        return trimmed.startsWith("SELECT") || trimmed.startsWith("WITH");
    }
}
```

**使用**：Agent 问"上个季度销售额"，MCP Server 自动转成 SQL 查数据库。

### 6.3 自己写一个 Java MCP Server（企业内部）

```java
package com.example.mcp.company;

import org.springframework.ai.mcp.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CompanyMcpServer {

    @Autowired private EmployeeService employeeService;
    @Autowired private LeaveService leaveService;
    @Autowired private ReimbursementService reimbursementService;

    @McpTool(description = "查询员工信息（姓名、工号、部门、职位）")
    public EmployeeDto getEmployee(
        @McpParam(description = "员工姓名（精确匹配）", required = false) String name,
        @McpParam(description = "员工工号", required = false) String empNo,
        @McpParam(description = "部门名称", required = false) String department
    ) {
        return employeeService.query(name, empNo, department);
    }

    @McpTool(description = "查询员工剩余年假天数")
    public LeaveBalance getLeaveBalance(
        @McpParam(description = "员工工号", required = true) String empNo
    ) {
        return leaveService.getBalance(empNo);
    }

    @McpTool(description = "提交报销申请")
    public ReimbursementResult submitReimbursement(
        @McpParam(description = "员工工号", required = true) String empNo,
        @McpParam(description = "报销金额（人民币元）", required = true) double amount,
        @McpParam(description = "费用类型：travel / meal / office / other", required = true) String type,
        @McpParam(description = "费用说明", required = true) String description
    ) {
        return reimbursementService.submit(empNo, amount, type, description);
    }

    @McpTool(description = "查询最近 N 天的报销记录")
    public List<ReimbursementRecord> listReimbursements(
        @McpParam(description = "员工工号", required = true) String empNo,
        @McpParam(description = "查询天数，默认 30 天") int days
    ) {
        return reimbursementService.list(empNo, days > 0 ? days : 30);
    }
}
```

---

## 第七部分：MCP 安全 + 治理

### 7.1 认证授权

```java
@Component
public class McpAuthInterceptor {

    @McpAuth
    public void checkAuth(String userId, String tenantId, String toolName) {
        // 1. 验证用户身份（JWT / OAuth2）
        // 2. 检查用户有没有权限调这个工具
        // 3. 注入多租户上下文
    }
}
```

**MCP Server 端**：

```yaml
spring:
  ai:
    mcp:
      server:
        security:
          enabled: true
          oauth2:
            issuer-uri: https://auth.company.com
            required-audience: mcp-server
```

### 7.2 输入校验（防注入）

```java
@McpTool(description = "执行 SQL 查询")
public List<Map<String, Object>> query(
    @McpParam(description = "SQL SELECT", required = true) String sql
) {
    // 1. 语法校验
    if (!isValidSelect(sql)) {
        throw new IllegalArgumentException("只支持 SELECT");
    }

    // 2. 表名白名单
    String tableName = extractTable(sql);
    if (!ALLOWED_TABLES.contains(tableName)) {
        throw new IllegalArgumentException("表 " + tableName + " 不允许访问");
    }

    // 3. LIMIT 限制
    if (!sql.toLowerCase().contains("limit")) {
        sql += " LIMIT 1000";
    }

    return jdbc.queryForList(sql);
}
```

### 7.3 限流

```java
@Component
public class McpRateLimitFilter {

    @Around("@annotation(org.springframework.ai.mcp.annotation.McpTool)")
    public Object rateLimit(ProceedingJoinPoint pjp) throws Throwable {
        String toolName = getToolName(pjp);
        String userId = UserContext.get().userId();

        String key = "ratelimit:mcp:" + userId + ":" + toolName;
        Long count = redis.increment(key);
        redis.expire(key, 60, TimeUnit.SECONDS);

        if (count > 100) {  // 每分钟 100 次
            throw new RateLimitExceededException(toolName + " 调用过于频繁");
        }
        return pjp.proceed();
    }
}
```

### 7.4 审计

```java
@Aspect
@Component
public class McpAuditAspect {

    @AfterReturning("@annotation(org.springframework.ai.mcp.annotation.McpTool)")
    public void audit(JoinPoint jp) {
        String toolName = getToolName(jp);
        Map<String, Object> args = Arrays.stream(jp.getArgs())
            .collect(Collectors.toMap(
                a -> a.getClass().getSimpleName(),
                a -> a
            ));

        AuditEvent event = new AuditEvent(
            "mcp_call",
            UserContext.get().userId(),
            toolName,
            args,
            Instant.now()
        );
        kafkaTemplate.send("mcp-audit", event);
    }
}
```

---

## 第八部分：MCP 高级话题

### 8.1 资源订阅（实时更新）

**MCP 支持"资源变了主动推"**：

```java
// Server 端：声明一个会变化的数据源
@McpResource(
    uri = "db://products",
    name = "产品目录",
    description = "实时产品数据",
    mimeType = "application/json"
)
public Resource getProductCatalog() {
    return new Resource(...);
}

// Server 端：资源变化时通知 Client
@EventListener
public void onProductChanged(ProductChangeEvent event) {
    mcpServer.notifyResourceUpdated("db://products");
}
```

**Client 端：订阅**：

```json
{
  "method": "resources/subscribe",
  "params": {
    "uri": "db://products"
  }
}
```

**应用场景**：实时数据看板、监控指标、行情数据。

### 8.2 采样（Sampling）—— Server 调 LLM

**MCP v0.3+ 引入了"采样"能力**——**Server 可以让 Client 调 LLM**。

**场景**：Server 想做"智能分类"但不想集成 LLM SDK。

```java
@McpTool(description = "智能分类邮件")
public String classifyEmail(
    @McpParam(description = "邮件内容", required = true) String content
) {
    // 让 Client 调 LLM 帮我们分类
    String category = mcpClient.sample(
        "请将邮件分类为：spam / important / normal",
        content
    );
    return category;
}
```

### 8.3 进度通知（Progress）

**长任务的进度推送**：

```java
@McpTool(description = "批量处理文件")
public void batchProcess(List<String> fileUris) {
    for (int i = 0; i < fileUris.size(); i++) {
        processFile(fileUris.get(i));
        // 推送进度
        mcpServer.notifyProgress(
            "batch-process",
            i + 1,
            fileUris.size()
        );
    }
}
```

---

## 第九部分：MCP 部署最佳实践

### 9.1 stdio vs SSE 选择

| 模式 | 适合 | 优势 | 劣势 |
|---|---|---|---|
| **stdio** | 本地工具、单机部署 | 简单、零网络、启动快 | 不能跨机器 |
| **SSE/HTTP** | 远程工具、集群部署 | 跨机器、多 Client 共享 | 需要 HTTPS、配置复杂 |

**生产推荐**：
- **企业内部工具** → **SSE 模式**（多 Agent 共享一套 MCP Server）
- **单机开发** → stdio（最简单）

### 9.2 MCP Server 集群化

```
Agent Pod 1 ─┐
Agent Pod 2 ─┼──→ MCP Server 集群 (3 副本) ──→ 数据库
Agent Pod 3 ─┘     ↑
                Consul/Nacos 注册中心
```

**关键设计**：
- MCP Server 无状态（除了数据库连接）
- 工具定义存在 ConfigMap
- 健康检查 `/health`

### 9.3 MCP Server 性能优化

- **工具发现缓存**：`tools/list` 结果缓存 5 分钟
- **连接池**：MCP Client 连接池（默认 10 个）
- **并发限制**：每个 MCP Server 限 100 并发

---

## 第十部分：MCP vs A2A 完整对比（深挖版 8 的补充）

既然这两篇一起看，给一张全景对比表：

| 维度 | MCP | A2A |
|---|---|---|
| **发起方** | Anthropic | Google + Linux 基金会 |
| **发布** | 2024-11 | 2025-04 |
| **范围** | Agent ↔ Tool | Agent ↔ Agent |
| **核心能力** | Tools / Resources / Prompts | Tasks / Messages / Artifacts / Agent Card |
| **通信** | JSON-RPC over stdio/SSE | JSON-RPC over HTTP/SSE |
| **发现** | 启动时协商 | 查 Agent Card |
| **认证** | 自行实现 | OAuth2 标准化 |
| **状态** | 无状态 | 有状态（Task）|
| **Java 栈** | Spring AI 1.0 原生 | 自研 + 第三方 SDK |
| **生态** | 50+ 现成 Server | 起步中 |
| **状态机** | 无 | 标准 7 态 |
| **推送** | 简单通知 | Webhook / SSE |
| **流式** | 不支持 | 原生支持 |
| **多轮** | 自行实现 | 原生 input-required |
| **成熟度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **2026 趋势** | 已成事实标准 | 2026-2027 主流化 |

**怎么选**：
- **Agent 要用工具**（查 DB / 读文件 / 调 API）→ **MCP**
- **Agent 要找别的 Agent 协作**（销售 Agent 调 客服 Agent）→ **A2A**
- **两者并存**：Agent 用 MCP 调工具，用 A2A 找别的 Agent

---

## 第十一部分：完整 Checklist

| 类别 | 检查项 |
|---|---|
| **协议** | 符合 MCP 2024-11-05 规范 |
| **协议** | 实现 3 大原语（Tools / Resources / Prompts）|
| **协议** | 正确处理 initialize 协商 |
| **协议** | 支持 stdio 和 SSE 两种模式 |
| **Server** | 用 Spring AI MCP 注解（`@McpTool` 等）|
| **Server** | Tool description 详细（决定 LLM 何时调用）|
| **Server** | 所有 Tool 有 inputSchema |
| **Server** | Resources 有 uri + mimeType |
| **Server** | Prompts 有 arguments 声明 |
| **Client** | 自动发现 Server 能力 |
| **Client** | 连接池管理 |
| **Client** | 同步 / 异步调用 |
| **安全** | OAuth2 / JWT 认证 |
| **安全** | 工具级权限控制 |
| **安全** | 输入校验（防注入）|
| **安全** | 限流（每用户 / 每工具）|
| **安全** | 审计日志（合规）|
| **部署** | Docker 化 |
| **部署** | K8s 部署 + 健康检查 |
| **部署** | stdio 模式本地用、SSE 模式集群用 |
| **可观测** | 调用次数 / 延迟 / 错误率指标 |
| **可观测** | 跨服务 Trace 串联 |
| **可观测** | 失败率告警 |
| **生态** | 优先用社区现成 Server（filesystem / git / postgres / slack）|
| **生态** | 自己写企业专用 Server（HR / CRM / ERP）|

---

## 写在最后

**MCP 不是"新工具调用"，是"工具生态的标准化"**。

**3 条 2026 年的判断**：

1. **MCP 已经成为 Agent 工具生态事实标准**——所有主流 Agent 框架（Spring AI / LangChain / CrewAI / Cline）都原生支持
2. **企业内部会出现"工具市场"**——像 npm 一样，发布 MCP Server 供其他 Agent 使用
3. **Java 工程师的优势**——MCP Server 大多数是 Node/Python 写的，**Java 实现的企业内部 MCP Server 反而是稀缺资源**

**给你 3 条建议**：

1. **今天就开始用 MCP**——别等"完美支持"，Spring AI 1.0 GA 已经稳定
2. **优先用社区 Server**——filesystem / git / postgres / slack 都有现成的
3. **企业内部工具都封装成 MCP Server**——HR / CRM / ERP / 自研系统

**MCP 让你的 Agent 不再"重复造轮子"**——这就是协议的力量。

---

