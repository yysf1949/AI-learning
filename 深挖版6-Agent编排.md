# 深挖版 6：企业级 Agent 编排——从单 Agent 到多 Agent 协作

> 日期：2026-06-10
> 配套基础版 + 深挖版 1/2/3/4/5
> 适合：负责企业级 Agent 系统的"大脑"——工作流、状态机、复杂决策的工程师

---

## 写在前面：为什么"编排"是 Agent 系统的"操作系统"？

**先说个真实场景**。

老板让你做一个**销售助手 Agent**：
1. 收到客户邮件
2. **判断**客户是新客户还是老客户
3. 新客户 → 查 CRM 创建商机 → 走标准报价流程 → 邮件回复
4. 老客户 → 查历史订单 → 智能推荐补货 → 邮件回复
5. **如果是 VIP 客户** → 转人工 + 抄送销售总监
6. 客户**回复确认** → 更新 CRM 状态 → 通知销售

**这种"多分支 + 状态 + 异步 + 异常恢复"的需求，用 Spring AI 那种简单的 ChatClient 调不通**。

你需要的是 **Agent 编排引擎**——Agent 系统的"操作系统"。

**2026 年企业级 Agent 编排的三大流派**：
- **代码编排**（LangGraph / Spring AI Graph / AutoGen）—— 程序员友好，灵活
- **低代码编排**（Dify / Coze / 阿里百炼）—— 业务人员友好，快速
- **协议编排**（A2A / ANP）—— 多 Agent 跨厂商协作

**这一篇我把三大流派全讲透**，给你 Java 工程师的视角，重点是**怎么选、怎么用、怎么不踩坑**。

---

## 第一部分：编排模式——5 种核心范式

**不管用什么框架，Agent 编排本质就 5 种模式**。理解这 5 种，看什么框架都门儿清。

### 1.1 Chain（链式）—— 最简单

```
Input → Agent A → Agent B → Agent C → Output
```

**适用**：线性任务，每步都执行，没分支。

**例子**：翻译任务
```
中文文本 → 翻译成英文 → 校对语法 → 输出
```

**Spring AI 实现**：
```java
String result = chatClient.prompt()
    .user(translatePrompt)   // 第一步：翻译
    .call()
    .content();
// 然后手动做第二步...
```

**缺点**：不灵活，没分支。

### 1.2 Router（路由）—— 简单分支

```
Input → 路由判断 → 走 A 路径 / B 路径 / C 路径
```

**适用**：根据输入选择不同处理路径。

**例子**：客服路由
```
用户问题 → 路由判断
  ├── 售前问题 → 售前 Agent
  ├── 售后问题 → 售后 Agent
  └── 投诉问题 → 投诉 Agent（升级）
```

**实现**：用 Spring AI 的 Advisor 机制或 LLM 分类。

```java
@Service
public class RouterAgent {

    @Autowired private ChatClient classifierClient;
    @Autowired private Map<String, ChatClient> agentMap;

    public String route(String question) {
        // 1. 用 LLM 分类
        String intent = classifierClient.prompt()
            .system("判断用户问题属于：售前 / 售后 / 投诉 / 其他")
            .user(question)
            .call()
            .content()
            .trim();

        // 2. 路由到对应 Agent
        ChatClient agent = agentMap.getOrDefault(intent, agentMap.get("default"));
        return agent.prompt().user(question).call().content();
    }
}
```

### 1.3 State Machine（状态机）—— 复杂业务流程 ⭐

**这是企业用得最多的模式。**

```
        ┌──→ 已创建 ──→ 处理中 ──→ 已完成
开始 ───┤      ↓           ↓
        └──→ 已取消 ←── 失败 ←──┘
```

**核心概念**：
- **State**（状态）：当前在哪一步
- **Transition**（转移）：从状态 A 到状态 B 的条件
- **Event**（事件）：触发转移的输入
- **Guard**（守卫）：转移的前置条件
- **Action**（动作）：转移时执行的操作

**销售助手的完整状态机**：
```
状态：NEW_LEAD → QUALIFIED → PROPOSAL_SENT → NEGOTIATING → CLOSED_WON/CLOSED_LOST

事件：
  - qualify_lead：线索合格
  - send_proposal：发送报价
  - customer_reply：客户回复
  - deal_won / deal_lost：成单 / 丢单

守卫：
  - 只有 score > 60 的线索才能 qualify
  - VIP 客户必须转人工
```

### 1.4 Planner-Executor（计划-执行）—— Agent 自主规划 ⭐⭐

**这是 2024-2026 年最火的模式。**

```
用户问题 → Planner（规划步骤）→ Executor（逐步执行）→ 验证 → 总结
```

**例子**：研究型 Agent
```
用户：分析苹果公司 2025 Q3 财报
   ↓
Planner 规划：
  1. 搜索"苹果 2025 Q3 财报"
  2. 抓取财报 PDF
  3. 提取关键财务数据
  4. 对比上季度
  5. 生成分析报告
   ↓
Executor 执行（逐步调工具）
   ↓
Verifier 验证（每步结果是否合理）
```

**优点**：能处理"未知任务"
**缺点**：慢、贵、可能走错

**Spring AI + LangGraph 实现**（这是 2026 年最实用的组合）。

### 1.5 Multi-Agent（多 Agent 协作）—— 终极形态 ⭐⭐⭐

```
                 ┌─→ 研究员 Agent
                 │
Orchestrator ────┼─→ 写手 Agent
                 │
                 └─→ 审核 Agent
```

**适用**：复杂任务，需要多个"专家"协作。

**例子**：内容生产
- 研究员：搜集资料
- 写手：写初稿
- 审核：检查质量
- 润色：提升文笔

**这部分和深挖版 8 的 A2A 协议深度联动**。

---

## 第二部分：LangGraph——代码编排的王者

### 2.1 为什么选 LangGraph？

**2026 年代码编排框架对比**：

| 框架 | 语言 | 状态机 | 学习曲线 | Java 栈 | 适合 |
|---|---|---|---|---|---|
| **LangGraph** | Python | ⭐⭐⭐⭐⭐ | 中 | 需 Python | 复杂 Agent 编排**首选** |
| **Spring AI Graph** | Java | ⭐⭐⭐⭐ | 中 | ✅ 原生 | **Java 栈首选** |
| **AutoGen** | Python | ⭐⭐⭐ | 中 | 需 Python | 多 Agent 对话 |
| **CrewAI** | Python | ⭐⭐ | 低 | 需 Python | 快速多 Agent 演示 |
| **AgentScope** | Python | ⭐⭐⭐⭐ | 中 | 需 Python | 阿里开源，多 Agent |

**Java 工程师建议**：**主力用 Spring AI Graph 1.0**（Spring AI 1.0 GA 自带），**研究 LangGraph 思路**（它是事实标准）。

**本节重点讲 Spring AI Graph + LangGraph 思路的 Java 落地**。

### 2.2 Spring AI Graph 完整实战

#### Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-deepseek</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-graph-core</artifactId>  <!-- 1.0 GA 新增 -->
</dependency>
```

#### 第一个状态机：客户投诉处理

```java
package com.example.orchestration;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.graph.StateGraph;
import org.springframework.ai.graph.action.NodeAction;
import org.springframework.ai.graph.node.QuestionClassifierNode;
import org.springframework.ai.graph.node.LlmNode;
import org.springframework.ai.graph.state.AgentState;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.Optional;

import static org.springframework.ai.graph.StateGraph.END;
import static org.springframework.ai.graph.StateGraph.START;

/**
 * 客户投诉处理状态机
 * 
 * 流程：
 *   START → 分类（严重程度）→ 解决 / 转人工 / 升级
 *                    ↓
 *                  回复客户
 */
@Component
public class ComplaintWorkflow {

    private final ChatClient chatClient;

    public ComplaintWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public StateGraph build() {
        // 1. 状态定义
        AgentStateFactory stateFactory = AgentStateFactory.builder()
            .add("complaint", String.class)          // 客户投诉内容
            .add("category", String.class)           // 分类：low / mid / high
            .add("solution", String.class)           // 解决方案
            .add("handled_by", String.class)         // 处理人：ai / human / manager
            .build();

        StateGraph graph = new StateGraph("complaint-workflow", stateFactory);

        // 2. 节点定义
        graph.addNode("classify", classifyNode());
        graph.addNode("auto_resolve", autoResolveNode());
        graph.addNode("transfer_human", transferHumanNode());
        graph.addNode("escalate_manager", escalateManagerNode());
        graph.addNode("reply_customer", replyCustomerNode());

        // 3. 边定义
        graph.addEdge(START, "classify");

        // 4. 条件路由
        graph.addConditionalEdges("classify",
            state -> state.get("category", String.class),
            Map.of(
                "low", "auto_resolve",       // 简单问题：自动解决
                "mid", "transfer_human",     // 中等问题：转人工
                "high", "escalate_manager"   // 严重问题：升级经理
            )
        );

        graph.addEdge("auto_resolve", "reply_customer");
        graph.addEdge("transfer_human", "reply_customer");
        graph.addEdge("escalate_manager", "reply_customer");
        graph.addEdge("reply_customer", END);

        return graph;
    }

    /**
     * 节点 1：分类（判断严重程度）
     */
    private NodeAction classifyNode() {
        return state -> {
            String complaint = state.get("complaint", String.class);
            String category = chatClient.prompt()
                .system("""
                    你是客服质检专家。判断客户投诉的严重程度：
                    - low: 一般咨询、商品细节问题
                    - mid: 商品质量问题、要求退换货
                    - high: 涉及安全、法律、严重服务事故
                    
                    只输出 low/mid/high 一个词。
                    """)
                .user(complaint)
                .call()
                .content()
                .trim();
            return Map.of("category", category);
        };
    }

    /**
     * 节点 2：自动解决（FAQ 库）
     */
    private NodeAction autoResolveNode() {
        return state -> {
            // 查知识库 / FAQ
            String solution = "根据您的问题，建议：...";  // 简化
            return Map.of(
                "solution", solution,
                "handled_by", "ai"
            );
        };
    }

    /**
     * 节点 3：转人工
     */
    private NodeAction transferHumanNode() {
        return state -> {
            // 调客服系统 API 转单
            ticketService.createTicket(state.get("complaint", String.class));
            return Map.of(
                "solution", "已为您转接专属客服，工单号 #" + ticketId,
                "handled_by", "human"
            );
        };
    }

    /**
     * 节点 4：升级经理
     */
    private NodeAction escalateManagerNode() {
        return state -> {
            // 通知客户经理 + 发短信
            notificationService.escalate(state.get("complaint", String.class));
            return Map.of(
                "solution", "已升级到客户经理处理，将 24 小时内联系您",
                "handled_by", "manager"
            );
        };
    }

    /**
     * 节点 5：回复客户
     */
    private NodeAction replyCustomerNode() {
        return state -> {
            String reply = state.get("solution", String.class);
            // 实际生产：调短信 / 邮件 API
            notificationService.sendToCustomer(reply);
            return Map.of("final_reply", reply);
        };
    }
}
```

**调用**：

```java
@Service
public class ComplaintService {

    @Autowired private ComplaintWorkflow workflow;

    public String handleComplaint(String customerId, String complaint) {
        // 1. 编译图
        CompiledGraph app = workflow.build().compile();

        // 2. 执行
        AgentState result = app.invoke(Map.of(
            "complaint", complaint
        ));

        // 3. 返回最终回复
        return result.get("final_reply", String.class);
    }
}
```

### 2.3 循环 + 错误重试（实际项目必备）

**问题**：LLM 调用可能失败（网络、限流、幻觉）

**解决**：加 retry 节点 + 最大重试次数

```java
graph.addNode("call_llm", callLlmNode());
graph.addNode("verify", verifyNode());
graph.addNode("retry_decision", retryDecisionNode());

graph.addEdge(START, "call_llm");
graph.addEdge("call_llm", "verify");
graph.addEdge("verify", "retry_decision");

// 条件：验证通过就 END，不通过就回到 call_llm
graph.addConditionalEdges("retry_decision",
    state -> {
        boolean isValid = state.get("is_valid", Boolean.class);
        int retryCount = state.get("retry_count", Integer.class);
        if (isValid) return "end";
        if (retryCount >= 3) return "fail";
        return "retry";
    },
    Map.of(
        "end", END,
        "fail", "fallback",
        "retry", "call_llm"
    )
);

graph.addNode("fallback", fallbackNode());  // 兜底
graph.addEdge("fallback", END);

// verify 节点：调第二个 LLM 验证第一个 LLM 的输出
private NodeAction verifyNode() {
    return state -> {
        String llmOutput = state.get("llm_output", String.class);
        boolean isValid = chatClient.prompt()
            .system("检查以下输出是否合理：是 → true，否 → false")
            .user(llmOutput)
            .call()
            .content()
            .contains("true");
        return Map.of("is_valid", isValid);
    };
}

private NodeAction retryDecisionNode() {
    return state -> {
        int count = state.getOrDefault("retry_count", 0, Integer.class);
        return Map.of("retry_count", count + 1);
    };
}
```

### 2.4 人工介入（Human-in-the-Loop）

**企业级 Agent 必须有人工审批节点**（转账、删数据、合同确认）。

```java
graph.addNode("human_approval", humanApprovalNode());

// 审批通过 → 执行
// 审批拒绝 → 取消
graph.addConditionalEdges("human_approval",
    state -> state.get("approval_status", String.class),
    Map.of(
        "approved", "execute_action",
        "rejected", "notify_rejection",
        "timeout", "auto_reject"  // 30 分钟没审批 → 自动拒绝
    )
);
```

**人工审批节点实现**：

```java
private NodeAction humanApprovalNode() {
    return state -> {
        // 1. 创建审批单
        String approvalId = approvalService.create(
            userId, "transfer_money", state.get("transfer_info", Map.class)
        );

        // 2. 阻塞等待审批结果（用 Redis 轮询 或 WebSocket 推送）
        ApprovalResult result = approvalService.waitForResult(approvalId, Duration.ofMinutes(30));

        return Map.of("approval_status", result.status());  // approved / rejected / timeout
    };
}
```

**前端配套**：
- 飞书机器人推送审批卡片
- 钉钉工作通知
- 企微审批流

### 2.5 子图（嵌套编排）

**复杂业务往往需要"图里套图"**。

```java
// 父图：销售流程
StateGraph parentGraph = new StateGraph("sales-process", ...);
parentGraph.addNode("qualify_lead", ...);
parentGraph.addNode("send_proposal", ...);
parentGraph.addNode("proposal_subgraph", proposalSubgraph());  // 嵌入子图
parentGraph.addNode("close_deal", ...);

// 子图：报价流程
public StateGraph proposalSubgraph() {
    StateGraph sub = new StateGraph("proposal-subgraph", ...);
    sub.addNode("calculate_price", ...);
    sub.addNode("apply_discount", ...);
    sub.addNode("generate_pdf", ...);
    // ... 子图内部的状态机
    return sub;
}
```

---

## 第三部分：LangGraph 思路（Java 工程师的理解方式）

**LangGraph 是 Python 界的 LangChain 团队出的编排框架，事实标准**。你做 Java 也建议理解它的设计思想。

### 3.1 LangGraph 核心概念（对照 Java）

| LangGraph 概念 | 含义 | Java 对应 |
|---|---|---|
| **State** | 图的全局状态（dict） | AgentState（Map）|
| **Node** | 一个执行单元 | NodeAction（函数）|
| **Edge** | 节点间的转移 | addEdge() |
| **Conditional Edge** | 条件分支 | addConditionalEdges() |
| **Checkpoint** | 状态快照（持久化）| 状态持久化到 Redis/PG |
| **Thread** | 一次执行的"会话" | conversationId |
| **Human-in-the-Loop** | 暂停等人工 | interrupt() |

### 3.2 一个 LangGraph 示例（参考用，不用跑）

```python
# Python LangGraph 示例
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    next_step: str

def classify(state):
    """分类节点"""
    msg = state["messages"][-1]
    # 调 LLM 分类
    return {"next_step": "tech_support" if "技术" in msg else "sales"}

def tech_support(state):
    """技术客服节点"""
    return {"messages": ["技术问题回答：..."]}

def sales(state):
    """销售节点"""
    return {"messages": ["销售问题回答：..."]}

# 构建图
workflow = StateGraph(AgentState)
workflow.add_node("classify", classify)
workflow.add_node("tech_support", tech_support)
workflow.add_node("sales", sales)

workflow.set_entry_point("classify")
workflow.add_conditional_edges("classify",
    lambda state: state["next_step"],
    {"tech_support": "tech_support", "sales": "sales"}
)
workflow.add_edge("tech_support", END)
workflow.add_edge("sales", END)

# 编译 + 持久化
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

# 执行
config = {"configurable": {"thread_id": "user-001"}}
result = app.invoke({"messages": ["我的订单有问题"]}, config)
```

**思路跟 Java 一样**——Java 用 Spring AI Graph 1.0 实现，Python 用 LangGraph。

### 3.3 持久化（Checkpoint）—— 关键能力

**为什么需要持久化？**
- **崩溃恢复**：执行到一半服务挂了，重启后从断点继续
- **人工介入**：暂停 → 人工 → 继续
- **长时间任务**：跨天执行（报价审批可能要等 24h）

```java
/**
 * Spring AI Graph 用 Redis 做 checkpoint
 */
@Configuration
public class GraphPersistenceConfig {

    @Bean
    public CheckpointSaver checkpointSaver(RedisTemplate redis) {
        return new RedisCheckpointSaver(redis, Duration.ofDays(7));
    }
}

// 编译时指定
CompiledGraph app = graph.compile(
    CompileConfig.builder()
        .checkpointSaver(checkpointSaver)
        .build()
);

// 执行时传 thread_id
AgentState result = app.invoke(
    Map.of("input", userInput),
    InvokeConfig.builder()
        .threadId(conversationId)  // 唯一标识
        .build()
);

// 7 天后回到同一 thread_id，可以从断点继续
```

### 3.4 流式输出（用户感知的"打字机效果"）

```java
// Spring AI Graph 1.0+ 的流式
CompiledGraph app = graph.compile(...);

app.stream(Map.of("input", userInput), config)
    .subscribe(event -> {
        switch (event.type()) {
            case "node_start":
                System.out.println("开始节点：" + event.nodeName());
                break;
            case "node_end":
                System.out.println("节点完成：" + event.nodeName());
                System.out.println("输出：" + event.state());
                break;
            case "llm_chunk":
                // 实时输出 LLM token
                System.out.print(event.chunk());
                break;
        }
    });
```

---

## 第四部分：低代码编排——Dify / Coze / 阿里百炼

**对于业务人员**（PM、运营、销售），低代码编排才是他们的"工作流"。

### 4.1 Dify 完整实战

**Dify** 是 2026 年最火的低代码 Agent 平台，**开源 + 自部署友好**。

#### 安装（Docker Compose）

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d
```

访问 `http://localhost/install` 完成初始化。

#### 用 Dify 搭一个"销售助手"（可视化）

1. **创建应用** → 选择"Chatflow"（多轮对话 + 工作流）
2. **拖拽节点**：
   ```
   开始 → 分类器 → [分类 A] RAG 检索
                    → [分类 B] 工具调用
                    → [分类 C] 转人工
   ```
3. **每个节点配置**：
   - **LLM 节点**：选模型、写 Prompt
   - **知识检索节点**：选向量库、写检索参数
   - **工具节点**：选工具（HTTP / 函数 / 数据集）
   - **条件分支**：用 IF 表达式

#### Dify 的"代码扩展"（DSL）

Dify 应用导出是 **YAML/DSL 格式**，可以纳入版本管理：

```yaml
# dify-app.yml
app:
  name: 销售助手
  mode: advanced-chat  # 工作流模式
  model_config:
    provider: langgenius/deepseek/deepseek-chat
    parameters:
      temperature: 0.7
  workflow:
    nodes:
      - id: start
        type: start
        data: {}
      - id: classify
        type: question-classifier
        data:
          model:
            provider: langgenius/deepseek/deepseek-chat
          categories:
            - name: 售前
              description: 客户询问产品功能、价格
            - name: 售后
              description: 客户已购买，遇到问题
            - name: 投诉
              description: 客户表达不满
      - id: handle_presale
        type: knowledge-retrieval
        data:
          dataset_ids: ["ds-product-info"]
          retrieval_mode: semantic
          top_k: 5
```

**优势**：
- 业务人员能独立维护
- 改 Prompt 不需要开发发版
- A/B 测试内置

### 4.2 Coze（字节扣子）—— C 端最火

**Coze 是字节 2024 年出的低代码平台，C 端用户多**。

**核心特性**：
- **插件市场**（几千个现成插件）
- **工作流 + 对话流**
- **一键发布到飞书/微信/抖音**
- **Bot Store**（用户分享 Bot）

**企业版**收费，但个人版免费。

### 4.3 阿里百炼 / 腾讯元宝—— 国产合规

**适合金融、政府、央国企**（要求国产化 + 私有化）。

**阿里百炼**：
- 完整工作流编排
- 内置通义千问 + 国产模型
- **企业级权限 / 审计 / 国产合规**
- 私有化部署支持

### 4.4 低代码 vs 代码——怎么选？

| 维度 | 低代码（Dify/Coze）| 代码（Spring AI Graph）|
|---|---|---|
| **开发速度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **灵活性** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **可维护性** | ⭐⭐（业务方维护）| ⭐⭐⭐⭐（开发维护）|
| **可观测性** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **企业集成** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **成本** | 低 | 中（要开发资源）|
| **适合** | 业务方主导 | 技术方主导 |

**实战建议（来自多家企业落地经验）**：
1. **业务方主导的场景**（客服 / 营销 / 知识问答）→ **低代码**优先
2. **技术方主导的场景**（研发效能 / 数据分析 / 业务流程）→ **代码**优先
3. **混合**（低代码搭骨架 + 代码扩展关键节点）→ **Dify + Spring AI**

---

## 第五部分：多 Agent 协作——5 大编排模式

**这部分是 2026 年最前沿的领域**。

### 5.1 编排模式一：Supervisor（监督者模式）

```
              Supervisor（监督者）
             ↙        ↓        ↘
        Worker A   Worker B   Worker C
        (研究)     (写代码)   (测试)
```

**Supervisor 负责**：
- 拆解任务
- 分发给 Worker
- 汇总结果

**Spring AI 实现**：

```java
@Service
public class SupervisorAgent {

    @Autowired private ChatClient supervisor;
    @Autowired private Map<String, Agent> workers;  // 多个 Worker Agent

    public String execute(String task) {
        // 1. Supervisor 拆解任务
        List<String> subtasks = supervisor.prompt()
            .system("你是任务分拆专家。把复杂任务拆成 3-5 个子任务。")
            .user(task)
            .call()
            .entity(new ParameterizedTypeReference<List<String>>() {});

        // 2. 分发给 Worker（可并行）
        List<String> results = subtasks.parallelStream()
            .map(subtask -> {
                String assignedWorker = chooseWorker(subtask);
                return workers.get(assignedWorker).execute(subtask);
            })
            .collect(Collectors.toList());

        // 3. Supervisor 汇总
        return supervisor.prompt()
            .system("你是总结专家。把多个子任务的结果整合成最终回答。")
            .user(String.join("\n\n", results))
            .call()
            .content();
    }
}
```

### 5.2 编排模式二：Hierarchical（层级模式）

```
CEO Agent
   ↓
部门 Agent × N
   ↓
员工 Agent × M
```

**适用**：组织结构化的 Agent（销售部门 / 客服部门 / 财务部门）

### 5.3 编排模式三：Collaborative（协作对话模式）

```
Agent A ←→ Agent B ←→ Agent C
   ↓
讨论 → 共识 → 行动
```

**经典实现**：AutoGen 的 GroupChat

```python
# Python AutoGen 例子（Java 用 Spring AI Graph 模拟）
from autogen import GroupChat, ConversableAgent

researcher = ConversableAgent(name="researcher", llm_config={...})
writer = ConversableAgent(name="writer", llm_config={...})
reviewer = ConversableAgent(name="reviewer", llm_config={...})

groupchat = GroupChat(
    agents=[researcher, writer, reviewer],
    messages=[],
    max_round=10
)
```

### 5.4 编排模式四：Pipeline（流水线模式）

```
Step 1 Agent → Step 2 Agent → Step 3 Agent → Output
```

**适用**：明确的步骤化任务（如 ETL、文档处理流水线）

### 5.5 编排模式五：Blackboard（黑板模式）

```
       ┌─→ Agent A →─┐
Input ─┼─→ Agent B →─┼─→ Blackboard（共享内存） → Output
       └─→ Agent C →─┘
```

**适用**：专家系统、复杂决策（多个 Agent 都能读写共享状态）

---

## 第六部分：实战案例——多 Agent 销售助手

**业务背景**：销售总监让你搭一个"客户开发全流程 Agent 群"。

### 6.1 角色设计

| Agent | 职责 | 工具 | 模型 |
|---|---|---|---|
| **Orchestrator** | 任务分拆、调度、汇总 | 无（纯 LLM）| Sonnet 4.6 |
| **Researcher** | 客户背景调研 | 搜索引擎、LinkedIn、企查查 | Haiku（便宜）|
| **Qualifier** | 客户质量评估（BANT）| CRM 查历史 | Sonnet 4.6 |
| **Outreach** | 邮件撰写、跟进 | 邮件 API、CRM 更新 | Sonnet 4.6 |
| **Analyst** | 数据分析、ROI 计算 | 数据库查询、BI | Sonnet 4.6 |

### 6.2 完整工作流

```java
@Component
public class SalesOrchestrator {

    private final StateGraph workflow;

    public SalesOrchestrator(
            ChatClient orchestratorLlm,
            ResearcherAgent researcher,
            QualifierAgent qualifier,
            OutreachAgent outreach,
            AnalystAgent analyst) {

        workflow = new StateGraph("sales-pipeline");

        // 节点
        workflow.addNode("orchestrator", orchestratorNode(orchestratorLlm));
        workflow.addNode("researcher", researcherNode(researcher));
        workflow.addNode("qualifier", qualifierNode(qualifier));
        workflow.addNode("outreach", outreachNode(outreach));
        workflow.addNode("analyst", analystNode(analyst));
        workflow.addNode("human_approval", humanApprovalNode());

        // 边
        workflow.setEntryPoint("orchestrator");

        // Orchestrator 根据任务类型路由
        workflow.addConditionalEdges("orchestrator",
            state -> state.get("task_type", String.class),
            Map.of(
                "research", "researcher",
                "qualify", "qualifier",
                "outreach", "outreach",
                "analyze", "analyst"
            )
        );

        // 每个 Worker 完成后回到 Orchestrator
        workflow.addEdge("researcher", "orchestrator");
        workflow.addEdge("qualifier", "orchestrator");
        workflow.addEdge("analyst", "orchestrator");

        // Outreach 是"对外动作"，需要人工审批
        workflow.addEdge("outreach", "human_approval");
        workflow.addConditionalEdges("human_approval",
            state -> state.get("approval", String.class),
            Map.of("approved", "orchestrator", "rejected", END)
        );
    }

    /**
     * Orchestrator 节点：判断下一步做什么
     */
    private NodeAction orchestratorNode(ChatClient llm) {
        return state -> {
            // 1. 看当前任务进度
            String currentState = formatState(state);

            // 2. LLM 决定下一步
            OrchestratorDecision decision = llm.prompt()
                .system("""
                    你是销售流程 Orchestrator。根据当前进度，决定下一步：
                    - research: 还需调研客户
                    - qualify: 还没评估客户质量
                    - outreach: 可以发邮件了
                    - analyze: 需要数据分析
                    - done: 流程完成
                    
                    输出 JSON: {"task_type": "research|qualify|outreach|analyze|done", "reason": "..."}
                    """)
                .user(currentState)
                .call()
                .entity(OrchestratorDecision.class);

            return Map.of(
                "task_type", decision.taskType(),
                "decision_reason", decision.reason()
            );
        };
    }
}
```

### 6.3 状态持久化（关键！）

**销售流程可能跨天**（客户 24h 内没回，要第二天再跟）。

```java
@Service
public class SalesService {

    @Autowired private SalesOrchestrator orchestrator;
    @Autowired private CheckpointSaver checkpoint;

    public String startSalesProcess(String customerId, String initialRequest) {
        CompiledGraph app = orchestrator.workflow.compile(
            CompileConfig.builder()
                .checkpointSaver(checkpoint)
                .build()
        );

        // 启动 + 立即返回 threadId
        AgentState state = app.invoke(
            Map.of(
                "customer_id", customerId,
                "initial_request", initialRequest,
                "status", "in_progress"
            ),
            InvokeConfig.builder()
                .threadId("sales-" + customerId)  // 用客户 ID 作 thread
                .build()
        );

        return "sales-" + customerId;
    }

    /**
     * 24h 后定时器触发：检查进度，决定继续还是结束
     */
    @Scheduled(fixedRate = 3600000)  // 每小时
    public void checkStalledProcesses() {
        for (String threadId : getActiveThreads()) {
            AgentState state = checkpoint.load(threadId);
            if (shouldContinue(state)) {
                // 继续执行
                orchestrator.workflow.invoke(state, InvokeConfig.builder().threadId(threadId).build());
            }
        }
    }
}
```

---

## 第七部分：生产环境编排的 6 大挑战

### 7.1 挑战 1：状态一致性

**问题**：Worker A 修改了 state，Worker B 读到的还是旧 state。

**解决**：
- **不可变 state**（每次都返回新 Map）
- **状态锁**（同一时刻只有一个 Worker 写）
- **乐观锁**（带版本号，冲突重试）

### 7.2 挑战 2：死循环

**问题**：Orchestrator 反复把任务分给同一个 Worker，状态机死循环。

**解决**：
```java
workflow.addNode("max_iter_check", state -> {
    int iter = state.getOrDefault("iter_count", 0, Integer.class);
    if (iter > 20) {
        return Map.of("status", "abort", "reason", "max iterations exceeded");
    }
    return Map.of("iter_count", iter + 1);
});
```

### 7.3 挑战 3：Token 爆炸

**问题**：多 Agent 协作时，每个 Agent 都看完整 context → Token 累加。

**解决**：
- **Context 压缩**（每跳都 summarize 上一跳）
- **Context 过滤**（只传递相关部分）
- **角色分工**（不同 Agent 看不同信息）

### 7.4 挑战 4：调试困难

**问题**：多 Agent 协作出错，不知道是哪一步、哪个 Agent 错。

**解决**：
- **全链路 Trace**（OpenTelemetry）
- **每步日志**（state dump）
- **Replay 能力**（用 checkpoint 重放）

### 7.5 挑战 5：错误恢复

**问题**：Worker A 失败，整个 workflow 挂。

**解决**：
```java
graph.addNode("error_handler", state -> {
    String error = state.get("error", String.class);
    log.error("Workflow error: {}", error);
    // 1. 记录错误
    errorTrackingService.record(error);
    // 2. 尝试降级（换一个 Worker）
    // 3. 或者直接 fail
    return Map.of("status", "failed");
});

// 节点执行 try-catch
private NodeAction safeNode(Agent agent) {
    return state -> {
        try {
            return agent.execute(state);
        } catch (Exception e) {
            return Map.of("error", e.getMessage());
        }
    };
}
```

### 7.6 挑战 6：可观测性

**问题**：编排层、LLM 层、Tool 层 三层混合，看不清全貌。

**解决**：见深挖版 7（可观测性）。

---

## 第八部分：完整 Checklist

| 类别 | 检查项 |
|---|---|
| **选型** | 业务方主导 → 低代码（Dify/Coze）|
| **选型** | 技术方主导 → 代码（Spring AI Graph）|
| **选型** | 跨厂商协作 → 协议（A2A/ANP）|
| **状态机** | 状态定义清晰（不可变）|
| **状态机** | 转移条件明确（不要"模糊"）|
| **状态机** | 守卫条件显式（"VIP 必须转人工"）|
| **循环控制** | 最大迭代次数限制 |
| **错误处理** | 每个节点 try-catch |
| **错误处理** | fallback 兜底节点 |
| **人工介入** | 高危操作必须审批 |
| **人工介入** | 审批超时自动降级 |
| **持久化** | Checkpoint 机制（Redis/PG）|
| **持久化** | Thread 唯一标识 |
| **持久化** | 跨天任务能继续 |
| **可观测** | 全链路 Trace |
| **可观测** | 每步 State Dump |
| **可观测** | Replay 能力 |
| **成本** | Context 压缩（多 Agent）|
| **成本** | Worker 按需激活（不预加载）|
| **成本** | 简单任务单 Agent 优先 |
| **测试** | 编排逻辑单元测试 |
| **测试** | LLM 行为评估（DeepEval）|
| **测试** | 端到端 E2E 测试 |

---

## 写在最后

**Agent 编排 = Agent 系统的"操作系统"**。

选错编排框架 = 选错操作系统 = 后面全盘被动。

**给你的 3 条选型建议**：

1. **Java 团队 + 复杂业务流程 → Spring AI Graph 1.0**（首选，2026 年 Java 栈事实标准）
2. **业务团队主导 + 快速试错 → Dify/Coze**（不是开发说了算，是业务说了算）
3. **多厂商 Agent 协作 → A2A 协议**（详见深挖版 8）

**最后送你 3 句话**：

1. **简单的事不要搞复杂**——单 ChatClient 能搞定的别上状态机
2. **复杂的事别靠堆 Agent**——一个状态机搞不定的事，5 个 Agent 也搞不定
3. **状态机是单 Agent 升级版，多 Agent 是状态机的进化**——**从单 → 编排 → 协作，是自然演进**

**这一篇 + 后面 4 篇（可观测性 / A2A / MCP / 向量库 / Tool Use），合在一起构成企业级 Agent 系统的"完整技术栈"。**

---

