# 深挖版 11：企业级 Agent Tool Use 完整实战——从 Function Calling 到工具生态

> 日期：2026-06-10
> 配套基础版 + 深挖版 1/2/3/4/5/6/7/8/9/10
> 适合：负责企业级 Agent 工具系统的工程师

---

## 写在前面：Tool Use 是 Agent 的"双手"

**先说个真实场景**。

老板让你做一个 Agent，能"帮你做事"。

**这个 Agent 有什么能力？** —— **取决于它能调多少工具**。

- 能查数据库 → 是"数据助手"
- 能发邮件 → 是"办公助理"
- 能写代码 → 是"研发同事"
- 能订机票 → 是"秘书"
- 能下单付款 → 是"采购员"

**Tool Use 就是 Agent 的"能力扩展机制"**。

**这一篇我把 Tool Use 讲透**——从基础 Function Calling，到复杂参数、异步工具、工具组合、沙箱安全、错误重试。

---

## 第一部分：Tool Use 基础（快速回顾）

### 1.1 Function Calling 工作原理

```
用户：北京天气怎么样？
   ↓
LLM 思考：用户问天气，我需要调"查天气"工具
   ↓
LLM 输出：function_call = {
    "name": "get_weather",
    "arguments": {"city": "北京"}
}
   ↓
你的代码：执行 get_weather("北京") → 返回 "晴，25℃"
   ↓
LLM 收到结果，生成自然语言回复：北京今天晴，25℃
```

### 1.2 Spring AI 1.0 的 Tool 定义

**基础回顾**（深挖版 1 已讲过）：

```java
@Component
public class WeatherTools {

    @Tool(description = "查询指定城市的天气")
    public String getWeather(
        @ToolParam(description = "城市名") String city
    ) {
        return weatherApi.get(city);
    }
}
```

**核心要素**：
- **`@Tool` 注解**：声明方法为工具
- **`description`**：告诉 LLM **何时调用**（最重要）
- **`@ToolParam`**：声明参数 + 描述
- **方法签名**：决定 input JSON Schema

### 1.3 Tool 的 4 大要素

```java
@Tool(
    name = "create_quote",                                       // 1. 工具名（英文，简洁）
    description = """
        为指定客户创建报价单。返回报价单 ID 和 PDF URL。
        适用场景：客户确认需求后需要出具正式报价。
        不要用于：订单创建（用 create_order）、询价（用 query_quote）。
        """                                                          // 2. 详细描述（决定 LLM 何时用）
)
public QuoteDto createQuote(
    @ToolParam(
        description = "客户 ID（CRM 中的 UUID）",                    // 3. 参数描述
        required = true,                                              // 4. 是否必填
        example = "C-001"                                             // 5. 示例
    ) String customerId,
    @ToolParam(description = "产品列表") List<ProductItem> items,
    @ToolParam(description = "是否立即发送邮件给客户", defaultValue = "false") boolean sendEmail
) {
    return quoteService.create(customerId, items, sendEmail);
}
```

**Description 写法铁律**：
- ❌ "查客户" —— 太简单
- ✅ "查询客户档案，包括基础信息、历史订单、未付款金额。返回结构化数据，可用于后续工具调用" —— **详尽 + 边界清晰 + 返回值说明**

---

## 第二部分：复杂参数处理

### 2.1 嵌套对象参数

**实际场景**：创建订单需要"产品列表"，每个产品有 ID + 数量。

```java
@Tool(description = "创建订单")
public OrderDto createOrder(
    @ToolParam(description = "客户 ID") String customerId,
    @ToolParam(description = "产品列表，每个产品有 product_id 和 quantity") List<ProductItem> items
) {
    return orderService.create(customerId, items);
}

public record ProductItem(
    @ToolParam(description = "产品 ID") String productId,
    @ToolParam(description = "数量") int quantity,
    @ToolParam(description = "是否加急", required = false, defaultValue = "false") boolean urgent
) {}
```

**LLM 调用时自动 JSON**：

```json
{
  "name": "create_order",
  "arguments": {
    "customer_id": "C-001",
    "items": [
      {"product_id": "P-100", "quantity": 2, "urgent": false},
      {"product_id": "P-200", "quantity": 1, "urgent": true}
    ]
  }
}
```

### 2.2 枚举参数

**约束取值范围**：

```java
public enum Priority { LOW, MEDIUM, HIGH, URGENT }

@Tool(description = "创建工单")
public TicketDto createTicket(
    @ToolParam(description = "客户 ID") String customerId,
    @ToolParam(description = "问题描述") String description,
    @ToolParam(description = "优先级：LOW / MEDIUM / HIGH / URGENT") Priority priority
) {
    return ticketService.create(customerId, description, priority);
}
```

**Spring AI 自动生成 JSON Schema**：

```json
{
  "type": "object",
  "properties": {
    "priority": {
      "type": "string",
      "enum": ["LOW", "MEDIUM", "HIGH", "URGENT"]
    }
  }
}
```

### 2.3 可选参数 + 默认值

```java
@Tool(description = "查询订单")
public List<OrderDto> queryOrders(
    @ToolParam(description = "客户 ID（与时间范围二选一）", required = false) String customerId,
    @ToolParam(description = "开始时间 yyyy-MM-dd", required = false) String startDate,
    @ToolParam(description = "结束时间 yyyy-MM-dd", required = false) String endDate,
    @ToolParam(description = "返回数量限制", required = false, defaultValue = "10") int limit
) {
    if (customerId == null && startDate == null) {
        throw new IllegalArgumentException("必须提供 customer_id 或 start_date");
    }
    return orderService.query(customerId, startDate, endDate, limit);
}
```

### 2.4 Map 参数（动态参数）

**适合参数不固定的场景**：

```java
@Tool(description = "通用查询接口，根据 filter 过滤")
public List<Map<String, Object>> query(
    @ToolParam(description = "表名") String table,
    @ToolParam(description = "过滤条件，key 是字段名，value 是值") Map<String, Object> filter
) {
    return dbService.query(table, filter);
}
```

**LLM 调用**：

```json
{
  "table": "customers",
  "filter": {
    "tenant_id": "acme",
    "department": "hr",
    "active": true
  }
}
```

**⚠️ Map 参数要给清晰描述，否则 LLM 不知道传什么**。

---

## 第三部分：异步工具

### 3.1 同步 vs 异步工具

| 场景 | 同步工具 | 异步工具 |
|---|---|---|
| **耗时** | < 5s | > 5s |
| **例子** | 查数据库、查天气 | 跑批处理、视频生成、模型训练 |
| **实现** | `@Tool` 直接返回 | 返回 `CompletableFuture` 或 polling 模式 |
| **用户体验** | 等结果 | 先返回 taskId，让 LLM 轮询 |

### 3.2 CompletableFuture 异步工具

```java
@Tool(description = """
    批量处理图片（缩略图 + 水印）。
    这是一个长时间任务，调用后会返回 task_id，
    用 query_batch_task 查询进度。
    """)
public BatchTaskResult batchProcessImages(
    @ToolParam(description = "图片 URL 列表") List<String> imageUrls
) {
    String taskId = UUID.randomUUID().toString();

    // 异步执行
    CompletableFuture.runAsync(() -> {
        for (int i = 0; i < imageUrls.size(); i++) {
            processOne(imageUrls.get(i));
            // 更新进度
            taskProgressService.update(taskId, i + 1, imageUrls.size());
        }
    }, batchExecutor);

    return new BatchTaskResult(taskId, "submitted",
        imageUrls.size() + " 张图片已加入处理队列");
}

@Tool(description = "查询批量任务的进度")
public TaskProgress queryBatchTask(
    @ToolParam(description = "任务 ID") String taskId
) {
    return taskProgressService.get(taskId);
}
```

**LLM 会自动**：
1. 调 `batchProcessImages` → 拿到 taskId
2. 调 `queryBatchTask` → 查进度
3. 完成后总结给用户

### 3.3 Streaming 流式工具

**适合大文件下载、长生成**：

```java
@Tool(description = "流式生成报告（适合长报告）")
public Flux<String> streamReport(
    @ToolParam(description = "报告主题") String topic
) {
    return chatClient.prompt()
        .user("生成关于 " + topic + " 的报告")
        .stream()
        .content();
}
```

**⚠️ Spring AI 1.0 的 Tool 是否支持 Flux 是版本相关**——具体看官方文档。

### 3.4 任务编排（Agent 内部异步）

**场景**：调 3 个异步工具，等全部完成再汇总。

```java
@Service
public class ParallelToolService {

    @Autowired private ToolService tools;

    /**
     * Agent 自动决定并行调 3 个工具
     */
    public String handleComplexQuery(String query) {
        // LLM 会自动：
        // 1. 决定调哪些工具
        // 2. 并行调（不冲突的）
        // 3. 串行调（有依赖的）
        // 4. 整合结果
        return chatClient.prompt()
            .user(query)
            .call()
            .content();
    }
}
```

**关键**：**工具的执行顺序由 LLM 决定**，不是程序硬编码。

---

## 第四部分：工具组合（Tool Composition）

### 4.1 链式调用

**场景**：查客户 → 查订单 → 算总金额

```
Tool 1: query_customer("C-001")
   ↓
Tool 2: query_orders("C-001")
   ↓
Tool 3: calculate_total(orders)
   ↓
总结给用户
```

**LLM 自主决策**（不需要你写代码）：

```java
// 你只管定义工具
@Tool(description = "查询客户") public CustomerDto queryCustomer(String id) {...}
@Tool(description = "查询客户订单") public List<OrderDto> queryOrders(String customerId) {...}
@Tool(description = "计算订单总金额") public BigDecimal calculateTotal(List<OrderDto> orders) {...}

// LLM 看到这些工具后，会自动决定调用顺序
```

**但 LLM 不是 100% 准确**——复杂链路**显式定义**更稳。

### 4.2 显式工具编排（用 Spring AI Graph）

```java
@Service
public class CustomerAnalysisWorkflow {

    public StateGraph build() {
        StateGraph graph = new StateGraph("customer-analysis");

        graph.addNode("query_customer", node(state ->
            Map.of("customer", customerService.findById(state.get("customer_id")))
        ));
        graph.addNode("query_orders", node(state ->
            Map.of("orders", orderService.findByCustomer(state.get("customer_id")))
        ));
        graph.addNode("calculate_ltv", node(state -> {
            Customer c = state.get("customer");
            List<Order> orders = state.get("orders");
            BigDecimal ltv = orders.stream()
                .map(Order::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            return Map.of("ltv", ltv);
        }));

        graph.addEdge(START, "query_customer");
        graph.addEdge("query_customer", "query_orders");
        graph.addEdge("query_orders", "calculate_ltv");
        graph.addEdge("calculate_ltv", END);

        return graph;
    }
}
```

**适合**：**调用顺序明确、业务逻辑固定**的场景。

### 4.3 工具检索（Tool Retrieval）

**问题**：1000 个工具，全部塞给 LLM → Token 爆炸，LLM 选错。

**解法**：**只给 LLM 相关的工具**（用 RAG 思想检索工具）。

```java
@Service
public class ToolRetrievalService {

    @Autowired private VectorStore toolVectorStore;  // 工具的"工具库"
    @Autowired private Map<String, ToolCallback> allTools;

    @PostConstruct
    public void init() {
        // 把所有工具的 description 灌到向量库
        for (ToolCallback tool : allTools.values()) {
            Document doc = new Document(tool.getToolDefinition().description());
            doc.getMetadata().put("tool_name", tool.getToolDefinition().name());
            toolVectorStore.add(List.of(doc));
        }
    }

    /**
     * 动态选择相关工具
     */
    public ToolCallback[] selectRelevantTools(String userQuery, int topK) {
        // 1. 用 query 检索最相关的 K 个工具
        List<Document> relevant = toolVectorStore.similaritySearch(
            SearchRequest.builder().query(userQuery).topK(topK).build()
        );

        // 2. 取这些工具的实例
        return relevant.stream()
            .map(d -> d.getMetadata().get("tool_name").toString())
            .map(name -> allTools.get(name))
            .filter(Objects::nonNull)
            .toArray(ToolCallback[]::new);
    }
}

// 在 ChatService 中
public String chat(String userQuery) {
    // 1. 动态选择工具
    ToolCallback[] tools = toolRetrieval.selectRelevantTools(userQuery, 10);

    // 2. 只把这些工具给 LLM
    ChatClient client = chatClientBuilder.defaultTools(tools).build();
    return client.prompt().user(userQuery).call().content();
}
```

**效果**：
- 1000 个工具 → 每次只给 LLM 10 个
- 准确率提升（少即是多）
- 成本降低（Prompt 变短）

---

## 第五部分：错误处理 + 重试

### 5.1 工具执行失败的原因

| 失败类型 | 例子 | 处理 |
|---|---|---|
| **参数错误** | LLM 传错参数 | 抛 IllegalArgumentException |
| **业务错误** | 客户不存在 | 抛 BusinessException |
| **系统错误** | DB 挂了 | 抛 RuntimeException |
| **超时** | API 30s 没回 | 抛 TimeoutException |
| **限流** | API 厂商 429 | 抛 RateLimitException |

### 5.2 重试 + 降级

```java
@Tool(description = "查询客户（带重试）")
public CustomerDto queryCustomer(
    @ToolParam(description = "客户 ID") String customerId
) {
    // 1. 参数校验
    if (customerId == null || customerId.isBlank()) {
        throw new IllegalArgumentException("customer_id 不能为空");
    }

    // 2. 重试（指数退避）
    int maxRetries = 3;
    for (int i = 0; i < maxRetries; i++) {
        try {
            return customerService.findById(customerId);
        } catch (Exception e) {
            if (i == maxRetries - 1) {
                // 3. 最后一次失败，尝试降级（从缓存查）
                CustomerDto cached = cacheService.get("customer:" + customerId);
                if (cached != null) {
                    log.warn("Customer service down, using cache for {}", customerId);
                    return cached;
                }
                throw new RuntimeException("客户查询失败", e);
            }
            try {
                Thread.sleep(1000L * (1 << i));  // 1s, 2s, 4s
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
            }
        }
    }
    throw new RuntimeException("Unreachable");
}
```

### 5.3 把错误信息返回给 LLM（让它重试）

**关键设计**：**工具执行失败时，把错误信息返回给 LLM，让 LLM 决定下一步**。

```java
@Tool(description = "创建订单")
public String createOrder(String customerId, List<ProductItem> items) {
    try {
        OrderDto order = orderService.create(customerId, items);
        return "订单创建成功，订单号：" + order.getId();
    } catch (CustomerNotFoundException e) {
        // 返回明确错误，让 LLM 能理解并修正
        return "错误：客户 " + customerId + " 不存在。请先调用 query_customer 确认客户 ID。";
    } catch (InsufficientStockException e) {
        return "错误：产品 " + e.getProductId() + " 库存不足（剩余 " + e.getRemaining() + "）。请减少数量或换其他产品。";
    } catch (Exception e) {
        return "错误：系统异常，请稍后重试。错误信息：" + e.getMessage();
    }
}
```

**LLM 收到错误后**：
- 客户不存在 → 自动调 query_customer 找正确 ID
- 库存不足 → 自动减少数量或换产品
- 系统异常 → 让用户重试

**这就是 Agent 自我纠错的能力**。

### 5.4 死循环检测

**问题**：LLM 反复调同一个工具失败。

**解法**：限制最大步数 + 循环检测。

```java
@Component
public class LoopDetectionAdvisor implements BaseAdvisor {

    @Override
    public ChatClientRequest before(ChatClientRequest request, AdvisorChain chain) {
        // 1. 统计当前对话的 Tool 调用次数
        int toolCallCount = countToolCalls(request);
        if (toolCallCount > 20) {
            throw new RuntimeException("工具调用超过 20 次，可能死循环");
        }

        // 2. 检测重复调用（同一个工具 + 相同参数 3 次）
        if (isRepeatingSameCall(request, 3)) {
            throw new RuntimeException("检测到重复工具调用");
        }

        return request;
    }
}
```

---

## 第六部分：工具安全 + 沙箱

### 6.1 为什么需要工具沙箱？

**真实事故**：2024 年某 Agent 被 Prompt Injection 攻击，**自动执行了"转账 100 万"**。

**根因**：**工具执行没有安全边界**——LLM 想调啥就调啥。

### 6.2 工具沙箱的 4 层防御

```
┌─────────────────────────────────────────┐
│ Layer 1: 工具白名单（OPA 鉴权）           │  ← 谁能调这个工具
├─────────────────────────────────────────┤
│ Layer 2: 参数校验（防注入）              │  ← 参数是否合法
├─────────────────────────────────────────┤
│ Layer 3: 隔离执行（gVisor / 容器）        │  ← 工具在哪跑
├─────────────────────────────────────────┤
│ Layer 4: 操作审批（高危操作）             │  ← 转账需人审批
└─────────────────────────────────────────┘
```

### 6.3 工具白名单 + 参数校验

**详见深挖版 2（权限）**——用 OPA 拦截每个 Tool 调用。

### 6.4 隔离执行（沙箱）

**场景**：让 Agent 跑 Python 代码分析数据，但**不能让它访问公司内网**。

**解法**：**用 Docker / gVisor / Firecracker 隔离执行**。

```java
@Service
public class SandboxedPythonExecutor {

    @Tool(description = "执行 Python 代码分析数据（沙箱环境，无网络）")
    public String executePython(
        @ToolParam(description = "Python 代码") String code
    ) {
        // 1. 用 Docker 启动隔离容器
        String containerId = dockerClient.createContainer("python:3.11")
            .withNetworkMode("none")           // 关键：无网络
            .withMemoryLimit(256 * 1024 * 1024)  // 关键：256MB 内存限制
            .withCpuLimit(0.5)
            .withReadonlyRootFs()
            .exec();

        // 2. 复制代码到容器
        dockerClient.copyToContainer(containerId, code.getBytes(), "/tmp/code.py");

        // 3. 执行（30s 超时）
        ExecResult result = dockerClient.exec(containerId, "python", "/tmp/code.py")
            .withTimeout(30)
            .exec();

        // 4. 清理
        dockerClient.removeContainer(containerId);

        return result.getOutput();
    }
}
```

**gVisor 配置**（更安全的沙箱）：

```yaml
# gVisor runsc 运行时
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

---
# Agent Pod 用 gvisor
spec:
  runtimeClassName: gvisor
  containers:
  - name: agent
    image: agent:latest
```

### 6.5 操作审批（人机协同）

**高危操作必须人审批**（转账、删数据、合同确认）：

```java
@Tool(description = "转账（需要人工审批）")
public String transfer(
    @ToolParam(description = "收款账户") String toAccount,
    @ToolParam(description = "金额（元）") BigDecimal amount
) {
    if (amount.compareTo(new BigDecimal("10000")) > 0) {
        // 1. 创建审批单
        String approvalId = approvalService.create(
            UserContext.get().userId(),
            "transfer",
            Map.of("to", toAccount, "amount", amount)
        );

        // 2. 返回"等待审批"状态（不是真转账）
        return "已创建转账审批 " + approvalId + "（金额 ¥" + amount + "），等待 CFO 审批";
    }

    // 3. 小额直接执行
    return transferService.execute(toAccount, amount);
}
```

---

## 第七部分：工具性能优化

### 7.1 工具缓存

```java
@Service
public class CachedToolService {

    @Cacheable(value = "tool:weather", key = "#city", unless = "#result == null")
    public String getWeather(String city) {
        return weatherApi.get(city);
    }
}
```

### 7.2 工具批处理

**场景**：LLM 连续 5 次调 query_customer(不同 ID) → 5 次 DB 查询。

**优化**：**自动合并成 1 次批量查询**。

```java
@Tool(description = "批量查询客户（最多 100 个）")
public List<CustomerDto> batchQueryCustomers(
    @ToolParam(description = "客户 ID 列表") List<String> customerIds
) {
    if (customerIds.size() > 100) {
        throw new IllegalArgumentException("最多 100 个");
    }
    return customerService.findAllByIds(customerIds);  // 1 次 DB 查询
}
```

**告诉 LLM 优先用批量版本**：

```java
@Tool(description = """
    批量查询多个客户（一次最多 100 个）。
    ⚠️ 不要用 query_customer 多次循环调用，用这个批量接口。
    """)
public List<CustomerDto> batchQueryCustomers(List<String> ids) { ... }
```

### 7.3 工具并行

**Spring AI 1.0+ 默认并行执行无依赖的工具**（取决于模型能力）。

**手动强制并行**：

```java
@Service
public class ParallelToolService {

    public List<CompletableFuture<Object>> executeParallel(List<ToolCall> calls) {
        return calls.stream()
            .map(call -> CompletableFuture.supplyAsync(() -> executeOne(call), toolExecutor))
            .collect(Collectors.toList());
    }
}
```

### 7.4 工具超时控制

```java
@Tool(description = "调用外部 API")
public ApiResult callExternalApi(
    @ToolParam(description = "URL") String url
) {
    return webClient.get()
        .uri(url)
        .retrieve()
        .bodyToMono(ApiResult.class)
        .timeout(Duration.ofSeconds(10))  // 关键：10s 超时
        .onErrorResume(e -> Mono.just(new ApiResult("timeout", null)))
        .block();
}
```

---

## 第八部分：Tool Use 监控 + 调试

### 8.1 工具调用埋点

```java
@Aspect
@Component
public class ToolCallMonitor {

    @Around("@annotation(tool)")
    public Object around(ProceedingJoinPoint pjp, Tool tool) throws Throwable {
        String toolName = tool.name();
        long start = System.currentTimeMillis();

        try {
            Object result = pjp.proceed();
            // 成功
            monitor.recordSuccess(toolName, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            // 失败
            monitor.recordFailure(toolName, e.getClass().getSimpleName());
            throw e;
        }
    }
}
```

### 8.2 工具调用链路追踪

**OpenTelemetry 完整 Trace**：

```
HTTP POST /api/agent/chat (1.8s)
├── LLM Call #1 (200ms, decide to call get_weather + query_customer)
├── Tool Call: get_weather (300ms, success)
│   ├── HTTP GET api.weather.com (250ms)
│   └── Parse Response (50ms)
├── Tool Call: query_customer (200ms, success)
│   ├── SQL SELECT (150ms)
│   └── Map Result (50ms)
└── LLM Call #2 (1.1s, generate final answer)
```

### 8.3 工具调用分析看板

**关键指标**：

| 指标 | 含义 | 健康值 |
|---|---|---|
| **Tool 调用次数** | 总调用数 | 跟业务量成正比 |
| **Tool 成功率** | 成功 / 总调用 | > 95% |
| **Tool P99 延迟** | 99 分位延迟 | < 5s |
| **Tool 错误率** | 错误 / 总调用 | < 5% |
| **最常用 Tool Top 10** | 调用次数排序 | 观察用户行为 |
| **失败 Tool Top 10** | 失败次数排序 | 优先修复 |
| **循环调用检测** | 同一 Tool 重复调用 | 应该是 0 |

---

## 第九部分：常见反模式（避坑指南）

### 反模式 1：Description 写得太简单

```java
// ❌ 错误
@Tool(description = "查客户")
public CustomerDto query(String id) {...}

// ✅ 正确
@Tool(description = """
    查询客户档案，包括基础信息（姓名、联系方式、地址）、历史订单（最近 90 天）、未付款金额。
    输入：客户 ID（必填，CRM 系统中的 UUID 格式）
    输出：客户完整信息
    失败：客户不存在时返回错误
    适用：需要了解客户背景、查询客户状态时
    不适用：创建订单（用 create_order）、查询订单（用 query_orders）
    """)
public CustomerDto query(String id) {...}
```

### 反模式 2：工具粒度太细

```java
// ❌ 错误：把一个完整操作拆成 5 个工具
@Tool public String step1(...) {...}
@Tool public String step2(...) {...}
@Tool public String step3(...) {...}
@Tool public String step4(...) {...}
@Tool public String step5(...) {...}

// ✅ 正确：1 个工具搞定
@Tool public OrderResult placeOrder(...) {...}
```

### 反模式 3：没有错误处理

```java
// ❌ 错误
@Tool public CustomerDto query(String id) {
    return customerService.findById(id);  // 抛异常给 LLM
}

// ✅ 正确
@Tool public CustomerDto query(String id) {
    try {
        return customerService.findById(id);
    } catch (CustomerNotFoundException e) {
        return new CustomerDto("NOT_FOUND", "客户不存在，请检查 ID 或用 query_customer_by_name 模糊查询");
    }
}
```

### 反模式 4：Tool 太多（> 50 个）

```java
// ❌ 错误：把 200 个内部 API 全暴露成 Tool
// LLM 选错的概率大增，Token 暴涨

// ✅ 正确：分组 + 工具检索
// 1. 把 200 个 API 分成 10 组（按业务域）
// 2. LLM 先选"组"，再选组内的"具体 API"
// 3. 或者用 Tool Retrieval（向量检索相关工具）
```

### 反模式 5：没有限流

```java
// ❌ 错误：用户问一个问题，LLM 调 100 次 Tool
@Tool public List<OrderDto> query(String customerId) {
    return orderService.findAll(customerId);  // 返回 1 万条
}

// ✅ 正确
@Tool public List<OrderDto> query(
    String customerId,
    @ToolParam(description = "返回数量限制", defaultValue = "10") int limit
) {
    return orderService.findAll(customerId, Math.min(limit, 100));
}
```

---

## 第十部分：完整 Checklist

| 类别 | 检查项 |
|---|---|
| **设计** | Description 详尽（说明何时调、参数、输出、限制）|
| **设计** | 工具粒度合适（一个工具 = 一个完整业务）|
| **设计** | 参数有类型 + 必填 + 默认值 + 示例 |
| **设计** | 嵌套参数用 record |
| **设计** | 枚举用 enum |
| **设计** | 返回值结构化（record / DTO）|
| **错误处理** | try-catch 全覆盖 |
| **错误处理** | 错误信息可读（让 LLM 理解）|
| **错误处理** | 重试 + 降级 |
| **错误处理** | 超时控制 |
| **错误处理** | 死循环检测 |
| **安全** | OPA 鉴权（谁能调）|
| **安全** | 参数校验（防注入）|
| **安全** | 沙箱执行（gVisor / Docker）|
| **安全** | 高危操作审批（人机协同）|
| **安全** | 限流（每用户 / 每租户）|
| **安全** | 审计日志 |
| **性能** | 缓存（相同输入不重复调）|
| **性能** | 批量工具（避免循环）|
| **性能** | 并行执行（无依赖的工具）|
| **可观测** | 调用次数 / 延迟 / 错误率 |
| **可观测** | Trace 链路追踪 |
| **可观测** | 失败告警 |
| **可观测** | 用户反馈 |
| **测试** | 单元测试（每个 Tool）|
| **测试** | 集成测试（Tool + LLM）|
| **测试** | 端到端测试 |
| **文档** | Tool 注册表（哪些 Tool 可用）|
| **文档** | Tool 用户手册（业务方）|

---

## 写在最后

**Tool Use 是 Agent 最容易被低估的能力**。

**很多人觉得"我会 Function Calling 就够了"**——**但企业级 Tool Use 包含**：
- 复杂参数（嵌套 / 枚举 / 动态）
- 异步执行
- 工具组合 + 编排
- 错误处理 + 重试 + 降级
- 沙箱安全 + 审批
- 性能优化 + 缓存 + 并行
- 监控 + 调试 + 测试

**给你 3 条建议**：

1. **Description 是灵魂**——花 50% 的时间写好 description，决定 LLM 用对工具的概率
2. **错误信息要可读**——LLM 不是人，但能读懂结构化错误
3. **高危操作必须人审批**——再聪明的 Agent 也不该自动转账

**最后送你 3 句话**：

1. **Tool 数量 ≠ 能力**——10 个设计良好的工具 > 100 个混乱的工具
2. **错误处理是 LLM 的"教学材料"**——它从错误里学
3. **安全沙箱不是可选项**——生产环境的 Agent 没有沙箱是定时炸弹

**Tool Use + MCP + A2A = Agent 工具生态完整图景**。MCP 让工具标准化，A2A 让 Agent 互操作，Tool Use 是底座。

---

