# 企业级 Agent 面试题专题 3：Agent 架构 + 工具调用 + 编排（30 题）

> 日期：2026-06-10
> 适用：1-3 年 Agent 开发经验
> 配套：深挖版 1（Spring AI）、深挖版 6（编排）、深挖版 11（Tool Use）、深挖版 8（A2A）、深挖版 9（MCP）

---

## 第一部分：Agent 基础（10 题）

### ⭐ Q1：什么是 AI Agent？跟 LLM 有什么区别？

**参考答案**：

**AI Agent = 能自主感知、规划、决策、行动的智能体**。

**核心特征**（4 大）：
1. **感知（Perception）**——能接收输入（文本 / 图像 / 语音 / 环境）
2. **规划（Planning）**——能拆解目标、制定步骤
3. **决策（Decision）**——能选择工具 / 路径
4. **行动（Action）**——能调用工具 / 操作系统

**LLM vs Agent**：

| 维度 | LLM | Agent |
|---|---|---|
| **本质** | 模型（参数 + 推理）| **系统**（LLM + Tools + Memory + Loop）|
| **能力** | 只能输出文本 | **能"做事"**（调 API、写文件、操作电脑）|
| **循环** | 一次性 | **多步循环**（感知→规划→行动→反思）|
| **自主性** | 被动响应 | **主动规划** |
| **工具** | 无 | **有**（Function Calling / MCP）|
| **记忆** | 短期（上下文）| **短期 + 长期 + 工具** |

**Agent 完整架构**：

```
                用户
                 ↓
            [感知/输入]
                 ↓
       ┌────────┴────────┐
       ↓                 ↓
   [记忆]            [规划]
   短期/长期         拆解目标
       │                 │
       └────────┬────────┘
                ↓
           [决策] ←── [反思]
                ↓           ↑
           [行动]  ─────┘
                ↓
         工具/外部系统
```

**加分项**：
- 提到 **Agent = LLM + 4 大能力**（规划 / 工具 / 记忆 / 反思）
- 提到 **Anthropic 定义**——Agent 是"能长时间自主完成任务的 LLM"
- 提到 **Agentic AI vs AI Agent**——前者是"范式"（强调自主性），后者是"实体"

---

### ⭐⭐ Q2：Agent 的核心循环是什么？ReAct 是什么？

**参考答案**：

**Agent 核心循环 = 感知 → 思考 → 行动 → 观察**（ReAct）。

**ReAct = Reasoning + Acting**（Yao et al. 2022）——经典的 Agent 范式。

```
用户：北京天气怎么样？上海呢？

思考 1：用户问两个城市的天气，需要调 get_weather 工具
行动 1：调用 get_weather("北京")
观察 1：晴，25℃

思考 2：北京查完了，需要查上海
行动 2：调用 get_weather("上海")
观察 2：多云，28℃

思考 3：两个都查完了，可以回答用户了
行动 3：生成最终回复
```

**3 种 Agent 范式**：

| 范式 | 思路 | 适合 |
|---|---|---|
| **ReAct** | 推理 + 行动交替 | 简单任务、QA |
| **Plan-and-Execute** | 先规划所有步骤，再执行 | **多步复杂任务** |
| **Reflexion** | 反思 + 自我改进 | 长任务、需要学习 |

**Plan-and-Execute 详解**：

```java
// 1. 规划阶段：LLM 生成所有步骤
List<Step> plan = planner.plan(goal);
// 步骤 1：查北京天气
// 步骤 2：查上海天气
// 步骤 3：对比并总结

// 2. 执行阶段：按步骤执行
for (Step step : plan) {
    Result result = executor.execute(step);
    state.add(result);
}

// 3. 重规划：发现失败，重新规划
if (state.hasFailure()) {
    plan = replanner.replan(goal, state);
}
```

**加分项**：
- 提到 **ReAct 的问题**——单步推理不够稳定
- 提到 **Reflexion（反思）**——失败时自我总结，下次避免
- 提到 **CoT + ReAct 混合**是 2026 主流

---

### ⭐⭐ Q3：什么是 Function Calling？底层协议是什么？

**参考答案**：

**Function Calling = LLM 输出结构化"工具调用"**（详见专题 1 Q6）。

**底层协议**（2026 年 3 大主流）：

| 协议 | 厂商 | 特点 |
|---|---|---|
| **OpenAI 协议** | OpenAI | 行业事实标准，5 年统治 |
| **Anthropic Tool Use** | Claude | 跟 OpenAI 类似，名字不同 |
| **MCP 协议** | Anthropic | **新标准**，跨厂商 |
| **A2A 协议** | Google | **Agent ↔ Agent**（不是 Agent ↔ Tool）|

**OpenAI Function Calling 协议**：

```json
// Request
{
  "model": "gpt-5.4",
  "messages": [...],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "查询天气",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {"type": "string"}
          }
        }
      }
    }
  ]
}

// Response
{
  "choices": [{
    "message": {
      "tool_calls": [{
        "id": "call_001",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"北京\"}"
        }
      }]
    }
  }]
}
```

**加分项**：
- 提到 **Function Calling ≠ Agent**——是 Agent 的一部分
- 提到 **MCP 协议**是 2026 年的趋势
- 提到 **多厂商 API 兼容**——DeepSeek、阿里百炼都兼容 OpenAI 协议

---

### ⭐⭐ Q4：Spring AI 1.0 的 Tool 定义方式是什么？最新 API 是什么？

**参考答案**：

**Spring AI 1.0 GA**（2025-05-20 发布）用 **`@Tool` 注解 + `ToolCallback`**：

#### 基础用法

```java
@Component
public class WeatherTools {

    @Tool(description = "查询指定城市的天气")
    public String getWeather(
        @ToolParam(description = "城市名") String city
    ) {
        return weatherService.get(city);
    }
}
```

**关键 API**：

| API | 用途 |
|---|---|
| `@Tool` | 声明方法为工具 |
| `@ToolParam` | 声明参数 + 描述 |
| `ChatClient` | 调用 LLM |
| `ChatClient.tools(...)` | 注入工具 |
| `MethodToolCallback` | 把方法转成 ToolCallback |
| `ToolCallbackProvider` | 批量注入工具 |

**完整调用**：

```java
@Service
public class AgentService {

    @Autowired private ChatClient chatClient;
    @Autowired private WeatherTools weatherTools;

    public String chat(String query) {
        return chatClient.prompt()
            .user(query)
            .tools(weatherTools)  // 注入工具
            .call()
            .content();
    }
}
```

**自动生成 JSON Schema**：

Spring AI 1.0 反射读取 `@Tool` + `@ToolParam`，自动生成：

```json
{
  "type": "function",
  "function": {
    "name": "getWeather",
    "description": "查询指定城市的天气",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {"type": "string", "description": "城市名"}
      },
      "required": ["city"]
    }
  }
}
```

**加分项**：
- 提到 **Spring AI 1.0 是事实 Java 栈标准**（前 LangChain4j 主流）
- 提到 **`@Tool` 注解的 description 决定 LLM 何时调用**——**50% 时间写好 description**
- 提到 **Spring AI Graph 1.0**（编排引擎，详见 Q8）
- 提到 **Spring AI MCP 集成**（`spring-ai-starter-mcp-server`）

---

### ⭐⭐⭐ Q5：什么是 Tool Retrieval？1000 个工具怎么给 LLM？

**参考答案**：

**问题**：1000 个工具全塞给 LLM → Token 爆炸 + LLM 选错。

**Tool Retrieval 思路** = **用 RAG 的思想检索"相关工具"**。

**完整架构**：

```
1. 启动时：把 1000 个工具的 description 灌到向量库
   ↓
2. 用户提问时：先用 query 检索 top 10 相关工具
   ↓
3. 只把这 10 个工具给 LLM
```

**Java 实现**：

```java
@Service
public class ToolRetrievalService {

    @Autowired private VectorStore toolVectorStore;
    @Autowired private Map<String, ToolCallback> allTools;

    @PostConstruct
    public void init() {
        for (ToolCallback tool : allTools.values()) {
            Document doc = new Document(tool.getToolDefinition().description());
            doc.getMetadata().put("tool_name", tool.getToolDefinition().name());
            toolVectorStore.add(List.of(doc));
        }
    }

    public ToolCallback[] selectRelevant(String query, int topK) {
        return toolVectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(topK).build()
        ).stream()
            .map(d -> allTools.get(d.getMetadata().get("tool_name").toString()))
            .filter(Objects::nonNull)
            .toArray(ToolCallback[]::new);
    }
}
```

**性能对比**：

| 方案 | 准确率 | Token 成本 |
|---|---|---|
| **1000 个工具全给** | 60% | 100K tokens/次 |
| **Tool Retrieval 选 10 个** | 75% | 1K tokens/次 |
| **分组 + 检索（2 级）** | 80% | 1.5K tokens/次 |

**加分项**：
- 提到 **1000 工具问题**——必须用 Tool Retrieval 或分组
- 提到 **2 级检索**——先选"工具组"再选"工具"
- 提到 **Spring AI 1.1+ 内置 Tool Retrieval**（未来）

---

### ⭐⭐⭐ Q6：什么是 Self-Consistency？什么是 Tree of Thoughts？

**参考答案**：

#### Self-Consistency

**核心思想**：**让 LLM 多次采样（不同 Temperature），然后投票**。

```java
List<String> answers = new ArrayList<>();
for (int i = 0; i < 5; i++) {
    String ans = chatClient.prompt()
        .user(question)
        .options(ChatOptions.builder().temperature(0.7).build())  // 略高
        .call()
        .content();
    answers.add(ans);
}
// 投票 / 取最常见的
String finalAnswer = mostCommon(answers);
```

**效果**（GSM8K 数学题）：

| 方法 | 准确率 |
|---|---|
| Greedy | 60% |
| Self-Consistency (5) | 75% |
| Self-Consistency (10) | 78% |
| Self-Consistency (40) | 80% |

#### Tree of Thoughts（ToT）

**核心思想**：**多分支探索 + 评估 + 剪枝**。

```
                起点
               /  |  \
              A   B   C
            / |   |   | \
           A1 A2  B1  B2  C1
           ↓  ↓   ↓
          评估 + 剪枝 → 选最优路径
```

**适合**：复杂规划问题（数学证明、谜题、规划）

**代价**：5-20x LLM 调用

**加分项**：
- 提到 **Self-Consistency 简单有效**——先试这个
- 提到 **ToT 慢**——只在需要时用
- 提到 **Anthropic 报告**——**生产中 Self-Consistency 不一定有效**（成本高）

---

### ⭐⭐⭐ Q7：什么是 ReAct 范式？什么是 Plan-and-Execute？怎么选？

**参考答案**：

**ReAct（Reasoning + Acting）**：

```
单步循环：思考 → 行动 → 观察 → 思考 → ...
```

**Plan-and-Execute**：

```
两阶段：1) 规划（一次生成所有步骤）  2) 执行（按步骤跑）
```

**对比**：

| 维度 | ReAct | Plan-and-Execute |
|---|---|---|
| **思考** | 每步都思考 | 先规划好 |
| **灵活** | 高（每步调整）| 低（计划死板）|
| **稳定** | 中（LLM 单步不一定准）| 高（计划明确）|
| **速度** | 慢（多次 LLM 调用）| 快（少次 LLM 调用）|
| **适合** | **短任务**（3-5 步）| **长任务**（10+ 步）|
| **失败恢复** | 自动调整 | 需重规划 |

**怎么选？**

```
任务步数？
    │
    ├── < 3 步 → 直接 ReAct
    ├── 3-10 步 → ReAct + Reflection
    └── > 10 步 → Plan-and-Execute + Re-planning
```

**加分项**：
- 提到 **ReAct 的"思考"让 prompt 变长**——成本高
- 提到 **Re-planning**（发现失败，重新规划）
- 提到 **Reflexion**（失败时反思，下次避免）

---

### ⭐⭐⭐ Q8：Spring AI Graph 1.0 是什么？怎么用？

**参考答案**：

**Spring AI Graph = 状态机 + 节点 + 边**（LangGraph 的 Java 版）。

**核心概念**：

| 概念 | 含义 |
|---|---|
| **State** | 全局状态（Map）|
| **Node** | 状态转换的"操作" |
| **Edge** | 节点之间的连接 |
| **START / END** | 起点 / 终点 |

**完整示例**（投诉处理）：

```java
@Service
public class ComplaintWorkflow {

    public StateGraph build() {
        StateGraph graph = new StateGraph("complaint-handling");

        // 1. 节点 1：识别投诉类型
        graph.addNode("classify", node(state -> {
            String text = (String) state.get("input");
            String type = classifier.classify(text);  // 退款/质量/服务
            return Map.of("type", type);
        }));

        // 2. 节点 2：查订单（条件路由）
        graph.addNode("query_order", node(state ->
            Map.of("order", orderService.findById((String) state.get("order_id")))
        ));

        // 3. 节点 3：退款处理
        graph.addNode("refund", node(state -> {
            RefundResult result = refundService.process((Order) state.get("order"));
            return Map.of("refund_result", result);
        }));

        // 4. 节点 4：质量投诉处理
        graph.addNode("quality", node(state -> {
            return Map.of("quality_result", qualityService.handle((String) state.get("input")));
        }));

        // 5. 边（流程）
        graph.addEdge(START, "classify");
        graph.addEdge("classify", "query_order");
        graph.addConditionalEdge("classify",
            state -> "refund".equals(state.get("type")) ? "refund" : "quality",
            Map.of("refund", "refund", "quality", "quality")
        );
        graph.addEdge("refund", END);
        graph.addEdge("quality", END);

        return graph;
    }
}
```

**执行**：

```java
CompiledGraph compiled = workflow.build().compile();
Map<String, Object> result = compiled.invoke(Map.of("input", "我要退款", "order_id", "12345"));
```

**加分项**：
- 提到 **LangGraph**（Python）vs **Spring AI Graph**（Java）类似
- 提到 **Checkpointer**（持久化）
- 提到 **Streaming**（流式输出）
- 提到 **Human-in-the-Loop**（人工介入）

---

### ⭐⭐⭐ Q9：什么是 Multi-Agent 协作？5 大模式是什么？

**参考答案**：

**Multi-Agent = 多个 Agent 协作完成复杂任务**。

**5 大模式**：

#### 模式 1：Supervisor（监督者）

```
        Supervisor
        /  |  \
       A   B   C
   （监督者调度）
```

#### 模式 2：Hierarchical（分层）

```
       Top
      /   \
   Mid1  Mid2
   / \    / \
  A   B  C   D
```

#### 模式 3：Collaborative（协作 / 辩论）

```
     A ←→ B
      ↕   ↕
       C
   （互相讨论）
```

#### 模式 4：Pipeline（流水线）

```
A → B → C → D
（按顺序）
```

#### 模式 5：Blackboard（黑板）

```
A ──┐
B ──┼──→ Blackboard（共享状态） ──→ 任意 Agent 读
C ──┘
```

**对比**：

| 模式 | 优势 | 适合 |
|---|---|---|
| **Supervisor** | 简单清晰 | 90% 场景 |
| **Hierarchical** | 复杂任务可拆 | 大型企业 Agent |
| **Collaborative** | 决策质量高 | 决策型任务 |
| **Pipeline** | 稳定可预测 | 标准化流程 |
| **Blackboard** | 灵活 | 复杂未知任务 |

**加分项**：
- 提到 **CrewAI / AutoGen** 是 Python 主流 Multi-Agent 框架
- 提到 **Spring AI Graph 支持所有 5 种模式**
- 提到 **A2A 协议**是 Multi-Agent 跨厂商协作的标准

---

### ⭐⭐⭐ Q10：什么时候用 Multi-Agent？什么时候用单 Agent？

**参考答案**：

**经验法则**：

```
一个 Agent 能搞定吗？
    │
    ├── 是 → 用单 Agent
    │
    └── 否 → 拆 Multi-Agent
```

**应该用 Multi-Agent 的信号**：

1. **任务明显可拆**（销售 + 客服 + 财务）
2. **需要不同专业**（不同 prompt / 工具 / 模型）
3. **需要并发执行**（并行查 5 个数据源）
4. **决策复杂**（需要辩论 / 反思）
5. **多租户 / 多角色**（每个 Agent 服务一个租户）

**应该用单 Agent 的信号**：

1. **任务简单**（FAQ 客服）
2. **响应延迟敏感**（< 1s）
3. **预算有限**（多 Agent 成本翻倍）
4. **问题可串行**（不用并行）

**反模式**：

- ❌ **为拆而拆**——3 个 Agent 干 1 个 Agent 的活（成本 ×3）
- ❌ **过度抽象**——把所有问题都设计成 Multi-Agent
- ✅ **从单 Agent 起步**——发现不够用再拆

**加分项**：
- 提到 **Multi-Agent 通信成本**——多 5-20 次 LLM 调用
- 提到 **Multi-Agent 调试困难**——4 个 Agent 出了 bug 难定位
- 提到 **CrewAI** 是 Python 主流 Multi-Agent 框架

---

## 第二部分：Agent 高级（10 题）

### ⭐⭐⭐⭐ Q11：什么是 MCP 协议？为什么需要 MCP？

**参考答案**：

**MCP（Model Context Protocol）= Agent 工具生态的 USB 接口**（详见深挖版 9）。

**为什么需要？**

**传统 Function Calling 的痛点**：
- 每个 Agent 框架要重新实现工具集成
- 工具不能跨 Agent 共享
- 重复造轮子

**MCP 解决**：
- **一次实现，到处运行**
- **工具生态共享**（50+ 官方 Server）
- **标准化**（统一协议）

**MCP vs Function Calling**：

| 维度 | Function Calling | MCP |
|---|---|---|
| **范围** | 工具调用 | 工具 + 资源 + Prompt |
| **可移植性** | ❌ 换框架重写 | ✅ 一次实现 |
| **发现** | 手动注册 | **自动发现** |
| **生态** | 各家孤岛 | **共享生态** |

**加分项**：
- 提到 **MCP 是 Anthropic 2024-11 发布**——已成事实标准
- 提到 **Spring AI 1.0 原生 MCP 支持**
- 提到 **50+ 官方/社区 MCP Server**（filesystem / git / postgres / slack）

---

### ⭐⭐⭐⭐ Q12：什么是 A2A 协议？跟 MCP 有什么区别？

**参考答案**：

**A2A（Agent-to-Agent）= Agent 跨厂商互操作协议**（详见深挖版 8）。

**Google 牵头，2025-04 发布**。

**4 大核心对象**：

| 对象 | 含义 |
|---|---|
| **Agent Card** | Agent 的"名片"（能做什么、怎么调）|
| **Task** | 一次协作的会话 |
| **Message** | 任务中的对话 |
| **Artifact** | 任务产出的文件 / 数据 |

**完整对比**：

| 维度 | MCP | A2A |
|---|---|---|
| **定位** | Agent ↔ Tool | **Agent ↔ Agent** |
| **发起方** | Anthropic | Google + Linux 基金会 |
| **发布** | 2024-11 | 2025-04 |
| **成熟度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

**加分项**：
- 提到 **MCP 用 A**（Agent 用工具）→ **A2A 用 A**（Agent 找 Agent）
- 提到 **A2A 的"状态"概念**——Task 有 7 种状态
- 提到 **A2A 的 4 种通信模式**（同步 / SSE / Webhook / 多轮）

---

### ⭐⭐⭐⭐ Q13：LangChain / LlamaIndex / Spring AI / LangChain4j 怎么选？

**参考答案**：

**2026 年 Java 栈对比**：

| 框架 | 厂商 | 特点 | 适合 |
|---|---|---|---|
| **Spring AI** | VMware / Broadcom | **官方标准**、Spring 生态 | **Java 企业首选** |
| **LangChain4j** | LangChain 社区 | LangChain Java 版 | **不想用 Spring** |
| **Quarkus LangChain4j** | Quarkus | 云原生、Dev UI | K8s / 响应式 |
| **Dify** | Dify 团队 | **低代码** + 生产 | **非工程师 / 快速原型** |
| **Coze** | 字节 | 低代码、Agent 编排 | C 端 / 国内 |

**怎么选**：

```
你用什么技术栈？
    │
    ├── Spring Boot → Spring AI（首选）
    ├── Quarkus → Quarkus LangChain4j
    ├── 纯 Java EE → LangChain4j
    ├── 非工程师 → Dify / Coze
    └── 不确定 → Spring AI
```

**对比**：

| 维度 | Spring AI | LangChain4j | Dify |
|---|---|---|---|
| **学习曲线** | 低（Java 熟）| 中 | 极低 |
| **生产就绪** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **生态** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **代码 vs 配置** | 代码 | 代码 | 配置 |
| **企业支持** | VMware | 社区 | 商业 |

**加分项**：
- 提到 **Spring AI 1.0 GA 是 2025-05-20**——生产稳定
- 提到 **LangChain4j 老牌、Spring AI 新锐**——但 2026 年 Spring AI 主流
- 提到 **Dify 适合 P...**[truncated]