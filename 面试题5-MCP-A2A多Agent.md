# 企业级 Agent 面试题专题 5：MCP / A2A 协议 + 多 Agent 协作（20 题）

> 日期：2026-06-10
> 适用：3-5 年 Agent 经验 / 架构师岗
> 配套：深挖版 8（A2A）、深挖版 9（MCP）、深挖版 6（编排）

---

## 第一部分：MCP 协议（8 题）

### ⭐⭐ Q1：什么是 MCP 协议？为什么需要 MCP？

**参考答案**：

**MCP（Model Context Protocol，模型上下文协议）= Agent 工具生态的 USB 接口**。

**发布**：2024-11（Anthropic 发起，2025 已被广泛采纳）。

**为什么需要 MCP**——传统 Function Calling 的痛点：

| 痛点 | 说明 |
|---|---|
| **重复造轮子** | 每个 Agent 框架要重写工具集成 |
| **不通用** | 工具不能跨框架共享 |
| **维护难** | 工具定义散落各处 |
| **升级痛** | 改协议要全栈改 |

**MCP 解决**：

```
传统：
LangChain ─┐
Dify ──────┼── 各自实现 query_weather
Spring AI ─┘

MCP 后：
              ┌── 任何 MCP 客户端
weather- ────┤   Claude / Cursor
mcp-server   ├── Spring AI
              └── LangChain4j
```

**MCP 三大原语**：

| 原语 | 含义 | 类比 |
|---|---|---|
| **Tools** | Agent 能调的函数 | USB 设备的"功能" |
| **Resources** | Agent 能读的数据 | U 盘里的"文件" |
| **Prompts** | 预定义 Prompt 模板 | IDE 的"代码片段" |

**加分项**：
- 提到 **MCP 已成 Agent 工具生态事实标准**（2025-2026）
- 提到 **50+ 官方/社区 MCP Server**（filesystem / git / postgres / slack / gdrive）
- 提到 **Spring AI 1.0 原生支持 MCP**

**Java 视角**：
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server</artifactId>
</dependency>
```

---

### ⭐⭐ Q2：MCP 跟 Function Calling 有什么区别？

**参考答案**：

**Function Calling**：OpenAI 协议，针对 LLM API。

**MCP**：标准协议，针对 Agent 工具生态。

**完整对比**：

| 维度 | Function Calling | MCP |
|---|---|---|
| **发起方** | OpenAI | Anthropic |
| **范围** | 工具调用 | 工具 + 资源 + Prompt |
| **协议** | 各厂商略有不同 | 统一协议（JSON-RPC）|
| **可移植性** | ❌ 换框架重写 | ✅ 一次实现到处用 |
| **发现机制** | 启动时告知 | 运行时动态发现 |
| **生态** | 各家孤岛 | 共享 Server 库 |
| **状态** | 无状态 | 支持通知、订阅 |

**举个例子**：

**Function Calling 写法**（OpenAI）：
```json
{
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "parameters": {...}
    }
  }]
}
```

**MCP 写法**：
```json
// Server 注册
{
  "name": "get_weather",
  "description": "查询天气",
  "inputSchema": {...}
}

// Client 调用
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {"city": "北京"}
  }
}
```

**加分项**：
- 提到 **MCP 跨语言**（Python / Java / Node / Go）
- 提到 **stdio 模式 + SSE/HTTP 模式**
- 提到 **MCP 协议还在演进**（v0.3+ 加了 Sampling、Resources Subscribe）

---

### ⭐⭐ Q3：MCP 的通信方式有几种？

**参考答案**：

**2 种通信方式**：

#### 方式 1：stdio（标准输入输出）

**适合**：本地工具（同一台机器）

```
MCP Host  ←→  MCP Server
   stdin/stdout
   （启动子进程）
```

**特点**：
- 启动：MCP Host 启动 MCP Server 子进程
- 通信：通过 stdin/stdout 发 JSON-RPC
- 优势：简单、零网络
- 劣势：不能跨机器

**Claude Desktop 配置示例**：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/docs"]
    }
  }
}
```

#### 方式 2：HTTP + SSE（Server-Sent Events）

**适合**：远程服务（跨机器）

```
MCP Host  ←→  MCP Server
   HTTP POST + SSE
   （远程 URL）
```

**特点**：
- 请求：HTTP POST `/message`
- 响应：SSE 流式
- 优势：跨机器、负载均衡
- 劣势：需要 HTTPS

**Spring AI SSE 配置**：

```yaml
spring:
  ai:
    mcp:
      server:
        sse:
          enabled: true
          port: 3000
          endpoint: /sse
```

**加分项**：
- 提到 **stdio 适合开发**，SSE 适合生产
- 提到 **stdio 也可以用 stdio 之外的 IPC**（named pipe 等）
- 提到 **未来 WebSocket 模式**（MCP v0.4+）

---

### ⭐⭐⭐ Q4：MCP 的能力协商是怎么做的？

**参考答案**：

**能力协商 = Client 和 Server 启动时告诉对方"我支持什么"**。

**流程**：

```
1. Client: initialize 请求（带我的能力）
   ↓
2. Server: initialize 响应（带 Server 的能力）
   ↓
3. 协商结果 = 双方能力交集
   ↓
4. 用协商后的能力集通信
```

**示例**：

```json
// Client → Server
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": {"listChanged": false},
      "sampling": {}
    },
    "clientInfo": {
      "name": "spring-ai-client",
      "version": "1.0.0"
    }
  }
}

// Server → Client
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {"listChanged": true},
      "resources": {"subscribe": true}
    },
    "serverInfo": {
      "name": "weather-mcp-server",
      "version": "1.0.0"
    }
  }
}
```

**协商结果的"交集"**：

| Client 能力 | Server 能力 | 协商后 |
|---|---|---|
| tools.list | tools.list | ✅ 都支持 |
| resources.subscribe | - | ❌ Server 不支持 |
| sampling | - | ❌ Server 不支持 |

**加分项**：
- 提到 **`listChanged` 通知**——Server 可以动态增删工具
- 提到 **协议版本**——`2024-11-05` 是当前版本
- 提到 **降级协商**——能力不够也能用，只是少一些特性

---

### ⭐⭐⭐ Q5：MCP Server 怎么实现？给个 Spring AI 完整例子

**参考答案**：

**完整 Spring AI MCP Server 实现**：

```java
@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}
```

**Tool 实现**：

```java
@Component
public class CustomerMcpServer {

    @Autowired private CustomerService customerService;

    @McpTool(description = """
        查询客户信息。
        必传 customer_id 或 customer_name 二选一。
        include_orders 决定是否包含历史订单。
        """)
    public CustomerDto queryCustomer(
        @McpParam(description = "客户 ID", required = false) String customerId,
        @McpParam(description = "客户姓名", required = false) String customerName,
        @McpParam(description = "是否包含订单", defaultValue = "true") boolean includeOrders
    ) {
        if (customerId == null && customerName == null) {
            throw new IllegalArgumentException("必须提供 customer_id 或 customer_name");
        }
        return customerService.query(customerId, customerName, includeOrders);
    }

    @McpTool(description = "为指定客户创建订单，返回订单 ID")
    public OrderDto createOrder(
        @McpParam(description = "客户 ID", required = true) String customerId,
        @McpParam(description = "产品列表") List<ProductItem> items
    ) {
        return orderService.create(customerId, items);
    }
}
```

**Resource 实现**：

```java
@Component
public class DocumentMcpServer {

    @McpResource(
        uri = "file://docs/employee-handbook.md",
        name = "员工手册",
        description = "公司员工手册",
        mimeType = "text/markdown"
    )
    public String getEmployeeHandbook() {
        return """
            # 员工手册
            ## 考勤：9:00-18:00，弹性 1 小时
            ## 年假：5-15 天（按工龄）
            """;
    }
}
```

**Prompt 实现**：

```java
@Component
public class PromptMcpServer {

    @McpPrompt(name = "code_review", description = "代码审查助手")
    public List<Map<String, Object>> codeReview(
        @McpArg(name = "language") String language,
        @McpArg(name = "code") String code
    ) {
        return List.of(Map.of("role", "user", "content", String.format("""
            请审查以下 %s 代码：
            ```%s
            %s
            ```
            关注：风格、性能、安全。
            """, language, language, code)));
    }
}
```

**加分项**：
- 提到 **MCP Server 是 stateless**——多个 Client 共享
- 提到 **错误返回格式**（JSON-RPC 标准错误码）
- 提到 **OAuth2 鉴权**（生产环境必须）

---

### ⭐⭐⭐ Q6：MCP 生态的 Server 列表有哪些？

**参考答案**：

**Anthropic 官方 + 社区维护的 50+ MCP Server**：

#### 文件 / 数据类

| Server | 用途 |
|---|---|
| `@modelcontextprotocol/server-filesystem` | 读本地文件 |
| `@modelcontextprotocol/server-postgres` | 查 PostgreSQL |
| `@modelcontextprotocol/server-sqlite` | 查 SQLite |
| `@modelcontextprotocol/server-redis` | 查 Redis |
| `@modelcontextprotocol/server-gdrive` | Google Drive |

#### 开发类

| Server | 用途 |
|---|---|
| `@modelcontextprotocol/server-git` | Git 操作 |
| `@modelcontextprotocol/server-github` | GitHub API |
| `@modelcontextprotocol/server-gitlab` | GitLab API |
| `@modelcontextprotocol/server-puppeteer` | 浏览器自动化 |

#### 通信 / 办公类

| Server | 用途 |
|---|---|
| `@modelcontextprotocol/server-slack` | Slack |
| `@modelcontextprotocol/server-gmail` | Gmail |
| `@modelcontextprotocol/server-notion` | Notion |
| `@modelcontextprotocol/server-linear` | Linear |

#### AI / 搜索类

| Server | 用途 |
|---|---|
| `@modelcontextprotocol/server-brave-search` | Brave 搜索 |
| `@modelcontextprotocol/server-fetch` | 抓网页 |
| `@modelcontextprotocol/server-memory` | 知识图谱记忆 |
| `@modelcontextprotocol/server-time` | 时间查询 |

#### Java 生态

| Server | 用途 |
|---|---|
| `spring-ai-starter-mcp-server` | **Java 实现**（任意工具）|

**加分项**：
- 提到 **"awesome-mcp-servers"** GitHub 仓库——MCP Server 索引
- 提到 **企业内部自研 Server**（HR / CRM / ERP）
- 提到 **MCP Server 复用度**——一个 Server 多个 Agent 共享

---

### ⭐⭐⭐ Q7：MCP 怎么保证安全？怎么防注入？

**参考答案**：

**4 层防御**：

#### L1：认证（OAuth2 / JWT）

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

```java
@McpAuth
public void checkAuth(String userId, String tenantId, String toolName) {
    if (!jwtValidator.validate(userId)) {
        throw new UnauthorizedException("未授权");
    }
    if (!permissionService.canCall(userId, toolName)) {
        throw new ForbiddenException("无权限调 " + toolName);
    }
}
```

#### L2：参数校验（防注入）

```java
@McpTool(description = "执行 SQL（只读）")
public List<Map<String, Object>> query(String sql) {
    // 1. 语法校验
    if (!isValidSelect(sql)) {
        throw new IllegalArgumentException("只支持 SELECT");
    }

    // 2. 表名白名单
    String table = extractTable(sql);
    if (!ALLOWED_TABLES.contains(table)) {
        throw new IllegalArgumentException("表 " + table + " 不允许");
    }

    // 3. LIMIT 限制
    sql += " LIMIT 1000";
    return jdbc.queryForList(sql);
}
```

#### L3：限流

```java
@Around("@annotation(McpTool)")
public Object rateLimit(ProceedingJoinPoint pjp) throws Throwable {
    String key = "ratelimit:mcp:" + UserContext.get().userId() + ":" + getToolName(pjp);
    Long count = redis.increment(key);
    if (count > 100) {  // 每分钟 100 次
        throw new RateLimitExceededException(...);
    }
    return pjp.proceed();
}
```

#### L4：审计 + 沙箱

```java
@Aspect
public class McpAuditAspect {
    @AfterReturning("@annotation(McpTool)")
    public void audit(JoinPoint jp) {
        auditLog.record(new AuditEvent(
            "mcp_call", UserContext.get().userId(),
            getToolName(jp), jp.getArgs(), Instant.now()
        ));
    }
}
```

**沙箱**（高危工具）：

```java
@McpTool(description = "执行 Python 代码（沙箱环境）")
public String executePython(String code) {
    // Docker 隔离 + 无网络 + 256MB 内存 + 30s 超时
    return sandboxedExecutor.run(code);
}
```

**加分项**：
- 提到 **OPA 鉴权**（详见深挖版 2）
- 提到 **审计合规**（GDPR / SOC2）
- 提到 **mTLS** 加密传输

---

### ⭐⭐⭐⭐ Q8：MCP v0.3+ 引入的 Sampling 是什么？

**参考答案**：

**Sampling = Server 让 Client 调 LLM**。

**反向调用**——传统是 Client 调 Server 的工具，**v0.3+ Server 可以反过来让 Client 调 LLM**。

**场景**：Server 想做"智能分类"但不想集成 LLM SDK。

```java
@McpTool(description = "智能分类邮件")
public String classifyEmail(String content) {
    // Server 让 Client 调 LLM 帮我们分类
    String category = mcpClient.sample(
        "请将邮件分类为：spam / important / normal",
        content
    );
    return category;
}
```

**MCP v0.3+ 的 3 个新能力**：

| 能力 | 含义 | 用途 |
|---|---|---|
| **Sampling** | Server 调 LLM | Server 内部用 LLM |
| **Resources Subscribe** | 资源变化推送 | 实时数据 |
| **Prompts List** | 列出可用 Prompt | 动态发现 |

**完整流程**：

```
Server 想用 LLM
    ↓
发 sampling/createMessage 请求给 Client
    ↓
Client 调 LLM（DeepSeek / GPT）
    ↓
Client 返回结果给 Server
    ↓
Server 继续执行
```

**加分项**：
- 提到 **Sampling 让 MCP Server 更"智能"**——不用集成 LLM
- 提到 **v0.3 2025-04 发布**——最新规范
- 提到 **Anthropic 主导**+ **OpenAI 兼容**

---

## 第二部分：A2A 协议（8 题）

### ⭐⭐ Q9：什么是 A2A 协议？跟 MCP 有什么区别？

**参考答案**：

**A2A（Agent-to-Agent）= Agent 跨厂商互操作协议**。

**发布**：2025-04（Google 牵头，Linux 基金会支持）。

**核心问题**：

```
5 个不同 Agent 想协作：
- 客服 Agent（Dify 搭的）
- 销售 Agent（Spring AI 写的）
- 财务 Agent（阿里百炼跑的）
- 研发 Agent（自研 Python）
- HR Agent（Coze 搭的）

怎么让它们互通？
```

**A2A 解决方案**：

- 暴露 **Agent Card**（"我能做什么"）
- 走 **JSON-RPC over HTTP/SSE**
- 互相 **发现 + 调用**

**完整对比 MCP vs A2A**：

| 维度 | MCP | A2A |
|---|---|---|
| **定位** | Agent ↔ **Tool** | **Agent ↔ Agent** |
| **发起方** | Anthropic | Google + Linux 基金会 |
| **发布** | 2024-11 | 2025-04 |
| **范围** | 工具 + 资源 + Prompt | 任务 + 消息 + 产物 |
| **通信** | stdio / SSE | HTTP / SSE / Webhook |
| **状态** | 无状态 | **有状态**（Task）|
| **生态** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **2026 趋势** | 事实标准 | 快速成熟 |

**加分项**：
- 提到 **A2A vs MCP 不冲突**——可以并存
- 提到 **A2A 跨厂商**——Java 写的 Agent 可以调 Python 写的

---

### ⭐⭐ Q10：A2A 的 4 大核心对象是什么？

**参考答案**：

#### 1. Agent Card（Agent 名片）

**Agent 的"自我介绍"**——告诉别人"我能做什么"。

```json
{
  "name": "销售助手 Agent",
  "description": "专业的销售支持 Agent",
  "url": "https://sales-agent.company.com",
  "skills": [
    {
      "id": "query_customer",
      "name": "查询客户",
      "description": "...",
      "inputSchema": {...},
      "examples": [...]
    }
  ],
  "authentication": {
    "type": "oauth2",
    "scopes": ["read:customer"]
  }
}
```

#### 2. Task（任务）

**一次协作的会话**。

| 状态 | 含义 |
|---|---|
| `created` | 已创建 |
| `submitted` | 已提交 |
| `working` | 处理中 |
| `input-required` | 需要更多信息 |
| `completed` | 完成 |
| `failed` | 失败 |
| `canceled` | 取消 |

#### 3. Message（消息）

**任务中的对话**——包含多个 parts。

```json
{
  "role": "agent",
  "parts": [
    {"type": "text", "text": "我查到了客户信息..."},
    {"type": "data", "data": {...}},
    {"type": "file", "file": {...}}
  ]
}
```

**3 种 part**：
- **text**——纯文本
- **data**——结构化数据（JSON）
- **file**——文件（URL 引用）

#### 4. Artifact（产物）

**任务完成后的产出**——文件 / 报告 / 数据。

```json
{
  "id": "artifact-001",
  "name": "报价单-2026-06-10.pdf",
  "parts": [
    {"type": "file", "file": {"uri": "https://...", "mimeType": "application/pdf"}}
  ]
}
```

**加分项**：
- 提到 **Agent Card 用 `.well-known/agent-card.json` 暴露**
- 提到 **Task 的 7 种状态是状态机**
- 提到 **Message parts 是多模态的**

---

### ⭐⭐⭐ Q11：A2A 的 4 种通信模式是什么？

**参考答案**：

#### 模式 1：Request/Response（同步）

```
Client → POST /a2a/tasks/send → Server
       ← 200 OK (Task)        ←
```

**适合**：短任务（< 5s）

#### 模式 2：Streaming（流式）

```
Client → POST /a2a/tasks/sendSubscribe → Server
       ← SSE: status + message + artifact events
```

**适合**：长任务（> 5s）

#### 模式 3：Push Notifications（推送）

```
1. Client 发任务，指定 webhookUrl
2. Server 处理完，主动 POST 到 webhookUrl
```

**适合**：异步任务（跨小时 / 跨天）

#### 模式 4：Multi-turn（多轮）

```
1. Agent: "需要更多信息"（input-required 状态）
2. User: 提供补充信息
3. Agent: 继续处理
```

**适合**：需要追问的任务

**加分项**：
- 提到 **同步适合 RESTful 风格**
- 提到 **SSE 跟 WebSocket 区别**——SSE 单向
- 提到 **Webhook 适合跨天任务**

---

### ⭐⭐⭐ Q12：怎么用 Spring Boot 实现 A2A Server？

**参考答案**：

**核心 Controller**：

```java
@RestController
@RequestMapping("/a2a")
public class A2aController {

    @Autowired private A2aTaskService taskService;
    @Autowired private AgentSkillRegistry skillRegistry;

    @GetMapping("/.well-known/agent-card.json")
    public AgentCard getAgentCard() {
        return skillRegistry.getAgentCard();
    }

    @PostMapping("/rpc")
    public JsonRpcResponse handleRpc(@RequestBody JsonRpcRequest request) {
        Object result = dispatch(request);
        return JsonRpcResponse.success(request.getId(), result);
    }

    @PostMapping("/rpc/stream")
    public SseEmitter handleStreamRpc(@RequestBody JsonRpcRequest request) {
        SseEmitter emitter = new SseEmitter(60_000L);
        // 异步处理 + SSE 推送
        taskService.executeWithStreaming(...);
        return emitter;
    }

    private Object dispatch(JsonRpcRequest request) {
        return switch (request.getMethod()) {
            case "tasks/send" -> taskService.createAndExecute(request);
            case "tasks/get" -> taskService.get(request.getParams().get("id").toString());
            case "tasks/cancel" -> taskService.cancel(request.getParams().get("id").toString());
            default -> throw new IllegalArgumentException("Unknown method");
        };
    }
}
```

**Skill 注册中心**：

```java
@Component
public class AgentSkillRegistry {
    private final Map<String, AgentSkill> skills = new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        register(QueryCustomerSkill.class);
        register(GenerateQuoteSkill.class);

        // 构建 Agent Card
        AgentCard card = AgentCard.builder()
            .name("销售 Agent")
            .skills(skills.values().stream().map(AgentSkill::toSkillDescriptor).toList())
            .build();
    }
}
```

**Task 执行引擎**：

```java
@Service
public class A2aTaskService {
    private final Map<String, A2aTask> taskStore = new ConcurrentHashMap<>();

    public A2aTask createAndExecute(JsonRpcRequest request) {
        A2aTask task = createOrResume(request);
        // 同步执行
        return executeSync(task);
    }

    public void executeWithStreaming(A2aTask task, BiConsumer<String, Object> emitter) {
        emitter.accept("status", Map.of("state", "working"));
        CompletableFuture.runAsync(() -> {
            A2aTask result = executeSync(task);
            emitter.accept("artifact", result.getArtifacts());
            emitter.accept("status", Map.of("state", result.getStatus().getState()));
        });
    }
}
```

**加分项**：
- 提到 **TaskStore 用 Redis 而非内存**——生产环境
- 提到 **Agent Card 用 CDN 缓存**
- 提到 **Health Check 端点**——`/health`

---

### ⭐⭐⭐⭐ Q13：A2A 怎么发现远端 Agent？怎么自动决策调哪个 Agent？

**参考答案**：

**场景**：主 Agent 接到任务，自动选合适的远端 Agent。

**实现步骤**：

#### 步骤 1：发现远端 Agent

```java
@Service
public class SmartOrchestrator {

    @Autowired private A2aClient a2aClient;

    private List<AgentCard> discoverAllAgents() {
        List<AgentCard> agents = new ArrayList<>();
        for (String url : configService.getAgentUrls()) {
            try {
                agents.add(a2aClient.discover(url));
            } catch (Exception e) {
                log.warn("Failed to discover: {}", url);
            }
        }
        return agents;
    }
}
```

#### 步骤 2：让 LLM 决策

```java
public String handleTask(String userQuery) {
    // 1. 列出所有可用 Agent
    List<AgentCard> agents = discoverAllAgents();

    // 2. 让 LLM 选
    String decision = llm.prompt()
        .system("""
            你是任务调度 Agent。
            根据用户问题，从可用 Agent 列表中选择最合适的。
            没有合适的就回复 "no_match"。
            """)
        .user("用户问题：" + userQuery +
              "\n\n可用 Agent：\n" + formatAgentList(agents))
        .call()
        .content();

    // 3. 解析 LLM 决策
    AgentChoice choice = parse(decision);
    if (choice == null) return "没找到合适的 Agent";

    // 4. 调用
    A2aTask result = a2aClient.callSkill(choice.agentUrl, choice.taskId,
        choice.skillId, choice.params, userId, tenantId);

    return formatResult(result);
}
```

#### 步骤 3：解析 LLM 决策

LLM 输出 JSON：

```json
{
  "agent_url": "https://sales-agent.company.com",
  "skill_id": "query_customer",
  "params": {
    "customer_id": "C-001",
    "include_orders": true
  }
}
```

解析：

```java
private AgentChoice parse(String llmOutput) {
    try {
        return objectMapper.readValue(llmOutput, AgentChoice.class);
    } catch (Exception e) {
        return null;
    }
}
```

**加分项**：
- 提到 **Agent Registry**（Nacos / Consul）——集中管理 Agent 列表
- 提到 **健康检查**——剔除不健康 Agent
- 提到 **能力索引**（Vector Store 加速 Agent 检索）

---

### ⭐⭐⭐⭐ Q14：A2A 在企业中的典型应用场景有哪些？

**参考答案**：

#### 场景 1：跨部门审批流程

```
销售 Agent（报价） → 财务 Agent（预算） → 法务 Agent（合同） → 销售 Agent（发送）
```

**实现**：每个 Agent 暴露 1-2 个 Skill，Task 状态持久化。

#### 场景 2：研究型多 Agent

```
用户问："分析某公司 Q1 财报"
   ↓
Orchestrator Agent
   ├── 数据 Agent（拉财报）
   ├── 分析 Agent（计算财务指标）
   ├── 对比 Agent（跟同行比）
   └── 写作 Agent（生成报告）
```

#### 场景 3：跨厂商协作

```
你的客服 Agent（Spring AI）
   + 客户公司的 CRM Agent（Salesforce）
   + 物流 Agent（FedEx）
   + 支付 Agent（Stripe）
```

**优势**：**不需要 API 对接**——只要都支持 A2A，自动发现 + 调用。

#### 场景 4：企业内部"Agent 市场"

```
HR 部门发布 HR Agent
IT 部门发布 IT Agent
财务部门发布 Finance Agent
   ↓
所有部门都能用（按需调用）
```

**加分项**：
- 提到 **"Agent 互操作"是新基础设施**
- 提到 **类比 npm / pip**——Agent 包管理
- 提到 **Linux 基金会 + Google 双背书**——标准稳

---

### ⭐⭐⭐⭐ Q15：A2A 怎么保证安全？怎么鉴权？

**参考答案**：

**4 层安全**：

#### L1：OAuth2 / JWT 认证

```java
@Component
public class A2aAuthInterceptor implements Filter {
    @Override
    public void doFilter(...) {
        String token = extractToken(request);
        if (!jwtValidator.validate(token)) {
            response.setStatus(401);
            return;
        }
        Claims claims = jwtValidator.parse(token);
        String skillId = extractSkillId(request);
        if (!hasScope(claims, skillId)) {
            response.setStatus(403);
            return;
        }
    }
}
```

#### L2：限流

```java
@Component
public class A2aRateLimitAdvisor {
    @Before
    public void check(ChatClientRequest request) {
        String agentUrl = request.context().get("agent_url");
        String key = "ratelimit:a2a:" + agentUrl + ":" + currentMinute();
        Long count = redis.increment(key);
        if (count > 1000) {
            throw new RateLimitExceededException("Agent 调用过于频繁");
        }
    }
}
```

#### L3：审计

```java
@Async
public void logCall(String callerAgent, String targetAgent, String skillId,
                    Map<String, Object> params, A2aTask result) {
    AuditEvent event = new AuditEvent(
        "a2a_call", callerAgent, targetAgent, skillId, params,
        result.getStatus().getState(), Instant.now()
    );
    kafkaTemplate.send("agent-audit", event);
}
```

#### L4：熔断

```java
@Service
public class A2aClientWithCircuitBreaker {
    @Autowired private CircuitBreakerRegistry registry;

    public A2aTask callSkillSafe(String agentUrl, ...) {
        CircuitBreaker cb = registry.circuitBreaker("a2a:" + agentUrl);
        try {
            return cb.executeSupplier(() -> a2aClient.callSkill(agentUrl, ...));
        } catch (CallNotPermittedException e) {
            return fallback();
        }
    }
}
```

**加分项**：
- 提到 **mTLS**——服务间加密
- 提到 **VPC Peering**——网络隔离
- 提到 **GDPR 合规**——审计 + 删除

---

### ⭐⭐⭐⭐ Q16：MCP 和 A2A 怎么协同工作？

**参考答案**：

**MCP 跟 A2A 不冲突——可并存**：

```
              用户
                ↓
          Orchestrator Agent
                ↓
        ┌───────┴───────┐
        ↓               ↓
    [A2A 调用]      [MCP 调用]
    远端 Agent       远端 Tool
   (跨厂商协作)    (数据/工具访问)
        ↓               ↓
   销售 Agent      Git MCP Server
   客服 Agent      Postgres MCP Server
   财务 Agent      Slack MCP Server
```

**实战架构**：

```
Orchestrator Agent (Java / Spring AI)
    │
    ├── [A2A] 调用销售 Agent
    │   → 查客户信息
    │   → 生成报价
    │
    ├── [MCP] 调用 Git MCP Server
    │   → 查代码仓库
    │   → 提交 PR
    │
    └── [MCP] 调用 Slack MCP Server
        → 发消息给团队
```

**用一句话区分**：

- **MCP**：Agent 用工具（A → T）
- **A2A**：Agent 找 Agent（A → A）

**加分项**：
- 提到 **MCP 用 Tools 资源，A2A 用 Agent Card 资源**
- 提到 **MCP 2024 主流，A2A 2026 主流**
- 提到 **Java 栈**：Spring AI 同时支持 MCP + A2A

---

## 第三部分：多 Agent 协作（4 题）

### ⭐⭐⭐ Q17：Multi-Agent 的 5 大模式是什么？什么时候用哪个？

**参考答案**：

| 模式 | 架构 | 适合 |
|---|---|---|
| **Supervisor** | 一个中心调度多个 Worker | **90% 场景** |
| **Hierarchical** | 分层（Top → Mid → Worker）| 大型企业 Agent |
| **Collaborative** | 互相讨论、辩论 | 决策型任务 |
| **Pipeline** | 流水线（A → B → C）| 标准化流程 |
| **Blackboard** | 共享状态，任意 Agent 读 | 复杂未知任务 |

**Supervisor 模式**（最常用）：

```
        Supervisor Agent
        /  |  \
       A   B   C
   (3 个 Worker)
```

**Hierarchical 模式**：

```
       Top Supervisor
      /             \
   Mid1            Mid2
   / \              / \
  A   B            C   D
```

**Collaborative 模式**：

```
     A ←→ B
      ↕   ↕
       C
   (辩论 / 投票)
```

**Pipeline 模式**：

```
A → B → C → D
（流水线）
```

**Blackboard 模式**：

```
A ──┐
B ──┼──→ Blackboard（共享状态）──→ 任意 Agent
C ──┘
```

**加分项**：
- 提到 **CrewAI** 是 Python 主流 Multi-Agent 框架
- 提到 **Spring AI Graph** 支持所有 5 种模式
- 提到 **LangGraph** 是 LangChain 的多 Agent 实现

---

### ⭐⭐⭐ Q18：Multi-Agent 通信成本太高怎么办？

**参考答案**：

**成本分析**：

```
单 Agent：1 次 LLM 调用
Multi-Agent (3 个)：5-20 次 LLM 调用（5-20x 成本）
```

**降低成本的 6 个方法**：

#### 1. 减少 Agent 数量

**经验法则**：能用单 Agent 解决就不用 Multi-Agent。

#### 2. 用小模型做 Worker

```java
// Supervisor 用旗舰
// Worker 用便宜
Supervisor → GPT-5.4（复杂决策，1M 上下文）
Worker → DeepSeek V4（简单执行，1M 上下文）
```

#### 3. 批量执行

```java
// 3 个 Worker 并行（不是串行）
CompletableFuture<A> fa = CompletableFuture.supplyAsync(() -> workerA.run());
CompletableFuture<B> fb = CompletableFuture.supplyAsync(() -> workerB.run());
CompletableFuture<C> fc = CompletableFuture.supplyAsync(() -> workerC.run());
// 1s 内完成（不是 3s）
```

#### 4. 缓存 Worker 结果

```java
@Cacheable(value = "worker_a", key = "#input")
public Result workerA(String input) {...}
```

#### 5. 早停（Early Stopping）

```java
if (supervisor.isSatisfied(currentResult)) {
    return currentResult;  // 早停
}
```

#### 6. 共享 LLM 调用

```java
// 多个 Agent 用同一个 ChatClient（连接复用）
ChatClient shared = chatClientBuilder.build();
```

**加分项**：
- 提到 **5x 成本是经验值**——具体看架构
- 提到 **真实场景 3-5x 更常见**——不是 20x
- 提到 **Agent 间通信用 A2A / MCP 标准化**

---

### ⭐⭐⭐⭐ Q19：Multi-Agent 系统怎么调试？怎么排查问题？

**参考答案**：

**调试难点**：

- **多步推理**——哪一步错了？
- **多 Agent**——哪个 Agent 错了？
- **非确定性**——每次结果可能不同

**调试工具链**：

| 工具 | 用途 |
|---|---|
| **Langfuse / Braintrust** | LLM 调用全记录 |
| **OpenTelemetry** | 分布式 Trace |
| **LangSmith** | LangChain 生态 |
| **Spring AI Debugger** | Spring AI 专用 |

**Trace 示例**：

```
HTTP POST /api/agent/chat (3.2s)
├── Supervisor Agent (200ms, decide to delegate to 3 workers)
├── Worker A "查天气" (1.1s, success)
│   ├── LLM Call #1 (300ms, decide to call get_weather)
│   ├── Tool Call: get_weather (700ms, success)
│   └── Return: 晴 25℃
├── Worker B "查日历" (1.3s, success)
│   └── ...
├── Worker C "查股价" (1.8s, FAILED)  ← 问题在这！
│   ├── LLM Call (300ms)
│   ├── Tool Call: get_stock_price (1500ms, timeout)
│   └── Error: RequestTimeoutException
└── Supervisor Synthesis (300ms, "C 失败了，回退到历史数据")
```

**调试技巧**：

1. **单步重放**——把 Agent 决策过程回放
2. **Prompt 注入**——用 mock LLM 测
3. **快照**——每步状态存下来
4. **A/B 对比**——两次执行对比

**Spring AI Graph 调试**：

```java
// 启用详细日志
logging.level.org.springframework.ai.graph=DEBUG

// Checkpointer（保存每步状态）
graph.addNode("checkpoint", node(state -> {
    checkpointStore.save(state);  // 每步存
    return state;
}));
```

**加分项**：
- 提到 **Langfuse 的 LLM 调用 replay**——可以重放
- 提到 **单元测试**每个 Agent 单独测
- 提到 **Property-based testing**（自动生成测试用例）

---

### ⭐⭐⭐⭐ Q20：多 Agent 系统怎么测试？测试策略是什么？

**参考答案**：

**4 层测试**：

#### L1：单元测试（每个 Agent / Tool）

```java
@Test
void testQueryCustomer() {
    CustomerDto result = customerAgent.query("C-001");
    assertNotNull(result);
    assertEquals("Acme Corp", result.getName());
}
```

#### L2：集成测试（Agent 协作）

```java
@Test
void testOrchestratorWithMockWorkers() {
    // Mock 远端 Agent
    A2aClient mockClient = mock(A2aClient.class);
    when(mockClient.callSkill(...)).thenReturn(mockResponse);

    // 测试 Orchestrator
    String result = orchestrator.handleTask("查客户 C-001");
    assertTrue(result.contains("Acme Corp"));
}
```

#### L3：端到端测试（真实 LLM）

```java
@SpringBootTest
@EnabledIfEnvironmentVariable(named = "DEEPSEEK_API_KEY", matches = ".+")
class E2ETest {
    @Test
    void testFullFlow() {
        String result = agentService.chat("北京天气");
        assertTrue(result.contains("北京"));
    }
}
```

#### L4：性能 / 压力测试

```java
@Test
void testConcurrentLoad() {
    ExecutorService executor = Executors.newFixedThreadPool(100);
    List<Future<String>> futures = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        futures.add(executor.submit(() -> agentService.chat("Hello")));
    }
    // 验证所有都成功
}
```

**评测指标**：

| 维度 | 指标 | 工具 |
|---|---|---|
| **答案质量** | Faithfulness / Relevancy | Ragas / DeepEval |
| **延迟** | P50 / P95 / P99 | Prometheus |
| **成本** | Token / Request | Langfuse |
| **成功率** | 任务完成率 | 自建埋点 |

**加分项**：
- 提到 **Golden Test Set**（100-200 个标准 QA）
- 提到 **回归测试**——Prompt 改完跑全套
- 提到 **红队测试**（安全）

---

## 写在最后

**这 20 题覆盖了 2026 年企业级 Agent 最热的话题**——MCP 协议 + A2A 协议 + 多 Agent 协作。

**关键判断**：

1. **MCP 是 2024-2026 的事实标准**——Java 工程师必须学
2. **A2A 是 2026-2027 的趋势**——多 Agent 协作的未来
3. **多 Agent 是 P1+ 的进阶技能**——先把单 Agent 做稳

**配合前 4 篇**——你已经能应对 90% 的企业级 Agent 面试了。

---

