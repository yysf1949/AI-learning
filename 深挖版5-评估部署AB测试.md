# 深挖版 5：企业级 Agent 评估体系 + 生产部署 + A/B 测试完整实战

> 日期：2026-06-10
> 配套基础版 + 深挖版 1/2/3/4
> 适合：负责企业级 Agent 系统从"能跑"到"可上线、可度量、可灰度"的工程师

---

## 写在前面：为什么"工程化"是转岗的最后一公里？

我先问你一个问题。

**你已经能写 Hello Agent、能写 RAG、能调权限、能调优了。** 接下来你要做什么？

**直接上线？** No。**你会在生产环境里死得很难看。**

为什么？因为 LLM 应用跟传统软件有一个**本质区别**：

> **传统软件：你写什么就输出什么（确定性）**
> **LLM 应用：你写 A，模型可能输出 B/C/D/完全胡说（概率性）**

这意味着：
- **改了一个 Prompt → 100 个回归问题，9 个变好，3 个变差，88 个没变但你不知道**——**没有评估你寸步难行**
- **上线了 V1，没法上线 V2**——**因为不知道 V2 是否比 V1 好**——**没有 A/B 你没法迭代**
- **挂了不知道挂在哪**——**没有监控告警你 7×24 守在屏幕前吗？**

**2026 年企业级 Agent 的标准工程化栈**（这才是完整闭环）：

```
评估体系（决定能不能上）
   ↓
CI/CD（决定能不能快上）
   ↓
A/B 测试（决定能不能验证上了更好）
   ↓
生产部署（Blue-Green / Canary）
   ↓
监控告警（决定挂了能不能知道）
   ↓
用户反馈（决定下次迭代方向）
   ↓
回到评估（闭环）
```

**这一篇我把 5 个环节全讲透**。每环节都给：
- **2026 年主流工具对比**（带数据和价格）
- **完整可运行代码**（Java 栈优先）
- **生产级 Checklist**
- **真实事故复盘**（告诉你没做会死在哪）

---

## 第一部分：LLM 评估体系（决定能不能上）

### 1.1 2026 年 LLM 评估框架全景对比

我先把 8 个主流框架的格局讲清楚：

| 框架 | 定位 | RAG 评估 | Agent 评估 | 红队测试 | CI 集成 | 生产监控 | 商业版 | 价格 | 适合 |
|---|---|---|---|---|---|---|---|---|---|
| **DeepEval** | 开源 + 学术 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Confident AI | $0（开源）/$299/月（团队）| **Java 栈首选，CI 强** |
| **Ragas** | 学术 + RAG 专精 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐ | 无 | $0 | **RAG 评估**（Python）|
| **Promptfoo** | 红队 + 端到端 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | Enterprise | $0/$499/月 | **安全测试 + CI** |
| **Braintrust** | 生产监控 + A/B | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 内置 | $0/$499/月 | **生产 A/B**（自部署）|
| **LangSmith** | LangChain 生态 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 强制付费 | $39/月起 | **Python LangChain 用户** |
| **Arize Phoenix** | 开源可观测 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | 内置 | $0（自部署）| **可观测 + 开源** |
| **W&B Weave** | 团队协作 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | 强制付费 | $50/月起 | **大厂 ML 团队** |
| **TruLens** | 学术传统 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐ | 无 | $0 | **研究 / 论文** |

**Java 栈推荐组合**：
- **离线评估** → DeepEval（CI 集成最强，Java 能用 Python SDK 或自研 Java 版）
- **生产监控** → Arize Phoenix（开源，自部署，OpenTelemetry 协议，Java SDK 完善）
- **A/B 测试** → Braintrust（自部署版）或自研 Redis-based
- **红队安全** → Promptfoo（独立于主流程）

### 1.2 核心评估指标（4 大类 12 个指标）

**别上来就跑 50 个指标，先跑这 12 个**：

#### A. 生成质量类（必跑）

| 指标 | 含义 | 算法 | 目标 |
|---|---|---|---|
| **Faithfulness** | 答案是否忠实于 context（没编）| NLI 模型判断"context → answer"是否蕴含 | > 0.95 |
| **Answer Relevancy** | 答案和问题的相关度 | 用 LLM 反向生成问题，再算相似度 | > 0.90 |
| **Hallucination Rate** | 幻觉率（编造内容的比例）| Faithfulness 的反义 | < 5% |
| **Toxicity** | 答案是否有毒 | Moderation API / Detoxify | < 0.01 |

#### B. RAG 检索类（必跑）

| 指标 | 含义 | 算法 | 目标 |
|---|---|---|---|
| **Context Precision** | 召回的 context 排序是否正确 | top K 中相关文档占比 | > 0.85 |
| **Context Recall** | 相关 context 召回了多少 | 标注集覆盖率 | > 0.90 |
| **Context Relevancy** | 召回的 context 和问题相关度 | LLM 判断 | > 0.85 |
| **MRR / NDCG** | 排序质量 | 倒数排名 / 归一化折损累计增益 | MRR > 0.7 |

#### C. Agent 行为类（Agent 必跑）

| 指标 | 含义 | 目标 |
|---|---|---|
| **Task Completion** | 任务完成率 | > 90% |
| **Tool Selection Accuracy** | 工具选择准确率 | > 95% |
| **Tool Call Success** | 工具调用成功率 | > 98% |
| **Steps to Completion** | 完成任务的步数 | 越少越好 |

#### D. 性能 + 成本类（生产必跑）

| 指标 | 含义 | 目标 |
|---|---|---|
| **P50 Latency** | 中位数延迟 | < 2s |
| **P99 Latency** | 99 分位延迟 | < 8s |
| **Cost per Request** | 每次请求成本 | 越低越好 |
| **Cache Hit Rate** | 缓存命中率 | > 60% |

### 1.3 DeepEval 实战（Java 栈首选）

**DeepEval 是 2026 年 Java 栈首选**——开源、CI 强、有商业版但社区版够用、指标全面。

#### 安装 + 准备测试集

```bash
# Python 装 DeepEval（CI 跑评估用）
pip install deepeval pytest
```

**准备 100 条标注测试集**（`test_cases.json`）：

```json
[
  {
    "id": "case-001",
    "input": "公司年假怎么请？",
    "expected_output": "通过 OA 系统提交申请，提前 3 天，直属领导审批",
    "context": [
      "公司年假制度：员工通过 OA 提交申请，需提前 3 天。直属领导审批后生效。",
      "年假标准：工龄 1-10 年 5 天，10-20 年 10 天，20 年以上 15 天。"
    ],
    "retrieval_context": [
      "公司年假制度：员工通过 OA 提交申请，需提前 3 天。直属领导审批后生效。"
    ],
    "expected_tools": [],  // 这个 case 不需要调工具
    "difficulty": "easy",
    "tags": ["hr", "faq"]
  }
]
```

#### 写评估测试（pytest + deepeval）

```python
# tests/eval_rag.py
import pytest
import json
from deepeval import assert_test
from deepeval.test_case import LLMTestCase, LLMTestCaseParams
from deepeval.metrics import (
    FaithfulnessMetric,
    AnswerRelevancyMetric,
    ContextualPrecisionMetric,
    ContextualRecallMetric,
    ContextualRelevancyMetric,
    HallucinationMetric,
    ToxicityMetric
)
from your_agent import AgentClient  # 你的 Java Agent 客户端

# 1. 加载测试集
with open("test_cases.json") as f:
    test_cases = json.load(f)

# 2. 配置评估指标
metrics = [
    FaithfulnessMetric(threshold=0.95, model="gpt-5.4"),
    AnswerRelevancyMetric(threshold=0.90, model="gpt-5.4"),
    ContextualPrecisionMetric(threshold=0.85, model="gpt-5.4"),
    ContextualRecallMetric(threshold=0.90, model="gpt-5.4"),
    ContextualRelevancyMetric(threshold=0.85, model="gpt-5.4"),
    HallucinationMetric(threshold=0.05),  # < 5% 幻觉
    ToxicityMetric(threshold=0.01)
]

@pytest.mark.parametrize("case", test_cases)
def test_rag_quality(case):
    """跑 RAG 质量评估"""
    # 1. 调你的 Agent（实际生产 HTTP 调用）
    agent = AgentClient(base_url="http://localhost:8080")
    result = agent.ask(
        question=case["input"],
        conversation_id=f"test-{case['id']}"
    )

    # 2. 构造 DeepEval 测试用例
    test_case = LLMTestCase(
        input=case["input"],
        actual_output=result["answer"],
        expected_output=case["expected_output"],
        context=case["context"],
        retrieval_context=result["retrieved_docs"]  # 你的 Agent 也要返回这个
    )

    # 3. 跑评估
    assert_test(test_case, metrics)
```

**运行评估**：

```bash
deepeval test run tests/eval_rag.py \
  --model gpt-5.4 \
  --output ./eval-results
```

**输出**：
```
✓ test_rag_quality[case-001] PASSED
  - Faithfulness: 0.98 ✓
  - Answer Relevancy: 0.92 ✓
  - Contextual Precision: 0.89 ✓
  ...
✗ test_rag_quality[case-023] FAILED
  - Faithfulness: 0.82 ✗ (threshold: 0.95)
  - Hallucination: 0.18 ✗ (threshold: 0.05)
  ⚠ AI 在第 3 句编了"年假最多 20 天"（实际是 15 天）
```

### 1.4 评估的"生产化"——测试集持续扩充

**真实场景**：用户点了"答得不对"→ 自动加入测试集 → 下次发版自动回归。

```python
# services/feedback_to_testset.py
from services.feedback_db import FeedbackDB
from services.testset_store import TestsetStore

class FeedbackToTestset:
    """把用户差评自动转成难例测试集"""

    def __init__(self):
        self.feedback_db = FeedbackDB()
        self.testset_store = TestsetStore()

    def weekly_sync(self):
        """每周跑一次（cron 触发）"""
        # 1. 拉本周所有差评（rating <= 2）
        bad_feedbacks = self.feedback_db.find(rating__lte=2, days=7)
        print(f"本周差评：{len(bad_feedbacks)} 条")

        # 2. LLM 帮我们生成"为什么错 + 应该怎么答"
        for fb in bad_feedbacks:
            analysis = self._analyze_failure(fb)
            # 3. 写入难例测试集
            self.testset_store.add({
                "id": f"hard-{fb['trace_id']}",
                "input": fb["question"],
                "actual_output": fb["answer"],
                "expected_output": analysis["expected"],
                "failure_reason": analysis["reason"],
                "tags": ["hard_case", "from_production"] + fb.get("tags", []),
                "added_at": datetime.now().isoformat()
            })

    def _analyze_failure(self, fb):
        """用 LLM 诊断失败原因"""
        prompt = f"""
        用户问题：{fb['question']}
        AI 回答：{fb['answer']}
        正确回答：{fb.get('expected', '(用户提供)')}
        用户反馈：{fb['feedback_text']}

        请分析：
        1. AI 错在哪里？
        2. 正确回答应该是什么？
        3. 这是哪类问题？（检索失败/生成错误/工具调用失败）
        """
        return llm.complete(prompt)
```

**效果**：**测试集每月自动 +20%，3 个月后从 100 条涨到 700 条**，评估越来越准。

### 1.5 评估的"四个反模式"（90% 团队都犯过）

**反模式 1：只跑 happy path**——只测"理想问题"，不测"刁钻问题"。

**反模式 2：只测"是不是对"，不测"为什么错"**——评估完只看到分数，不知道改进方向。

**反模式 3：测试集永远不更新**——3 个月前的测试集跟现在业务不匹配，评估没意义。

**反模式 4：单看一个指标**——Faithfulness 高但 Answer Relevancy 低 → 答得对但答非所问。

---

## 第二部分：CI/CD 流水线（决定能不能快上）

### 2.1 完整 CI/CD 流程

```
代码提交 → 单元测试 → Lint → 类型检查
   ↓
构建 Docker 镜像
   ↓
跑 LLM 评估（关键！）
   ↓
评估通过 → 推镜像到 Registry
   ↓
部署到 Staging 环境
   ↓
跑回归测试 + 性能测试
   ↓
人工 Review（可选）
   ↓
Canary 发布（5% → 25% → 100%）
   ↓
监控 + 自动回滚
```

### 2.2 GitHub Actions 完整配置

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Job 1: 单元测试 + LLM 评估
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python（跑评估）
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build Java
        run: mvn clean package -DskipTests

      - name: Run unit tests
        run: mvn test

      - name: Install deepeval
        run: pip install deepeval

      - name: Start Agent service
        run: |
          java -jar target/agent-1.0.0.jar &
          sleep 30  # 等待服务启动
          curl -f http://localhost:8080/api/chat/health || exit 1

      - name: Run LLM evaluation
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          deepeval test run tests/eval_rag.py \
            --model gpt-5.4 \
            --output ./eval-results

      - name: Check evaluation results
        run: |
          python scripts/check_eval_results.py \
            --input ./eval-results \
            --fail-on-regression

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: ./eval-results/

  # Job 2: 构建 + 推送镜像
  build:
    needs: test  # 测试通过才能构建
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t agent:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push agent:${{ github.sha }}
          docker tag agent:${{ github.sha }} agent:latest
          docker push agent:latest

  # Job 3: 部署到 Staging
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to K8s staging
        run: |
          kubectl set image deployment/agent \
            agent=agent:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/agent -n staging

      - name: Smoke test
        run: |
          python scripts/smoke_test.py --env staging

  # Job 4: Canary 发布到生产
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Canary deploy 5%
        run: |
          kubectl apply -f k8s/canary-5percent.yaml
          echo "5% 流量切到新版本，观察 10 分钟..."

      - name: Check canary metrics
        run: python scripts/check_canary_metrics.py --threshold 0.05

      - name: Promote to 100%
        if: success()
        run: |
          kubectl apply -f k8s/production-100percent.yaml
          echo "✅ 部署完成"
```

### 2.3 评估结果对比（防止回归）

```python
# scripts/check_eval_results.py
"""
对比本次评估和上次基线，发现回归直接 fail CI
"""
import json
import sys
import argparse

def check_regression(current_path, baseline_path, threshold=0.02):
    with open(current_path) as f:
        current = json.load(f)
    with open(baseline_path) as f:
        baseline = json.load(f)

    regressions = []
    for metric in current["metrics"]:
        baseline_score = baseline["metrics"].get(metric["name"], {}).get("score", 0)
        current_score = metric["score"]
        diff = current_score - baseline_score

        if diff < -threshold:
            regressions.append({
                "metric": metric["name"],
                "baseline": baseline_score,
                "current": current_score,
                "diff": diff
            })

    if regressions:
        print("❌ 发现回归！")
        for r in regressions:
            print(f"  {r['metric']}: {r['baseline']:.3f} → {r['current']:.3f} (差 {r['diff']:.3f})")
        sys.exit(1)
    else:
        print("✅ 无回归，本次评估通过")
```

**实际效果**：
```
=== 评估对比 ===
Baseline (commit abc123):
  - Faithfulness: 0.96
  - Answer Relevancy: 0.92
  - Contextual Precision: 0.87

Current (commit def456):
  - Faithfulness: 0.97 ✓
  - Answer Relevancy: 0.88 ✗ (回归 -0.04)
  - Contextual Precision: 0.88 ✓

❌ 发现回归：Answer Relevancy 从 0.92 降到 0.88
⚠ 你的新 Prompt 改了答案格式，让用户觉得"答非所问"
```

**这就是 LLM 评估 + CI 的价值——在你发版前拦住回归。**

### 2.4 Prompt 评测（跟代码评测同等重要）

**Prompt 跟代码一样要版本管理、测试、发版**。

```yaml
# prompts/customer-service/v2.1.0.md 的元数据
---
name: customer-service
version: 2.1.0
changelog: "调整输出格式，每行不超过 30 字"
test_pass_rate: 0.94
author: 张三
reviewed_by: 李四
deprecated: false
---

你是客服助手。回答问题时要：
1. 简洁（每行 < 30 字）
2. 有礼貌
3. 严格根据知识库回答
```

**Prompt 评估测试**（跟代码测试一样）：

```python
# tests/eval_prompts.py
import pytest
from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import GEval

def test_customer_service_prompt_v2_1_0():
    """验证 v2.1.0 prompt 的效果没退步"""
    prompt_loader = PromptLoader()

    # 对比老版本
    for case in customer_service_cases:
        for version in ["v2.0.0", "v2.1.0"]:
            prompt = prompt_loader.get("customer-service", version)
            response = llm.complete(prompt + case["input"])

            test_case = LLMTestCase(
                input=case["input"],
                actual_output=response,
                expected_output=case["expected"]
            )

            # 自定义评估：简洁度（每行 < 30 字）
            conciseness = GEval(
                name="Conciseness",
                criteria="答案每行不超过 30 字",
                threshold=0.9
            )
            assert_test(test_case, [conciseness])
```

**Prompt 发布流程**：
```
1. 工程师改 prompts/customer-service/v2.2.0.md
2. 提交 PR
3. CI 自动跑评估（对比 v2.1.0）
4. 评估通过 → 人工 review → merge
5. CD 流程加载新 Prompt 到生产
```

---

## 第三部分：A/B 测试（决定能不能验证上了更好）

### 3.1 A/B 测试在 Agent 场景的 4 个特殊点

**跟传统 A/B 相比，Agent A/B 难点**：

1. **结果有随机性**——同一个 Prompt 跑两次，输出不一样
2. **评估指标不是二元的**——不是"点了/没点"，是"满意度""准确性"这种连续值
3. **需要冷启动时间**——新版本要跑够 N 次才有统计意义
4. **业务指标滞后**——"用户满意"要几小时甚至几天才反映

### 3.2 A/B 测试系统设计

```
┌────────────────────────────────────────────────────┐
│ 客户端请求                                           │
│      ↓                                              │
│ 用户带 conversation_id 进入                          │
│      ↓                                              │
│ AB Router：取 hash(conversation_id) % 100          │
│      ├── 0-4  → 5% 流量 → 实验组 B（新 Prompt）      │
│      └── 5-99 → 95% 流量 → 对照组 A（老 Prompt）      │
│      ↓                                              │
│ 记录到 Metrics（用户 ID、组别、问题、答案、评分）      │
│      ↓                                              │
│ 每小时统计：转化率 / 满意度 / 延迟 / 成本             │
│      ↓                                              │
│ 达到统计显著 → 自动/人工决定全量                      │
└────────────────────────────────────────────────────┘
```

### 3.3 自研 A/B 路由（Java 实现）

```java
package com.example.abtest;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.security.MessageDigest;
import java.util.Map;

/**
 * A/B 路由服务
 */
@Service
public class AbRouter {

    @Value("${ab.experiments.customer-service.v3:0.05}")
    private double v3Ratio;  // 5% 流量到 v3

    public ExperimentDecision route(String userId, String experiment) {
        // 1. 用户 ID hash → 0-9999
        int hash = stableHash(userId + ":" + experiment) % 10000;

        // 2. 按比例路由
        String variant;
        if (hash < v3Ratio * 10000) {
            variant = "v3";
        } else {
            variant = "control";
        }

        // 3. 强制白名单（QA / 内部员工始终走新版本）
        if (isInWhitelist(userId)) {
            variant = "v3";
        }

        return new ExperimentDecision(variant, Map.of(
            "experiment", experiment,
            "hash", hash,
            "user_id", userId
        ));
    }

    private int stableHash(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(input.getBytes());
            // 取前 4 字节
            return Math.abs((hash[0] << 24) | (hash[1] << 16) | (hash[2] << 8) | hash[3]);
        } catch (Exception e) {
            return 0;
        }
    }
}

record ExperimentDecision(String variant, Map<String, Object> context) {}
```

### 3.4 集成到 ChatClient

```java
@Service
public class AbChatService {

    @Autowired private AbRouter abRouter;
    @Autowired private Map<String, ChatClient> promptClients;  // 每个 prompt 一个 ChatClient

    public String chat(String userId, String conversationId, String message) {
        // 1. A/B 路由
        ExperimentDecision decision = abRouter.route(userId, "customer-service-prompt");
        String variant = decision.variant();

        // 2. 选对应 Prompt 的 ChatClient
        ChatClient client = promptClients.get("customer-service:" + variant);

        // 3. 调 LLM
        long start = System.currentTimeMillis();
        String response = client.prompt()
            .user(message)
            .advisors(/* ... */)
            .call()
            .content();
        long latency = System.currentTimeMillis() - start;

        // 4. 埋点（异步发 Kafka，不阻塞主流程）
        metricsCollector.recordAsync(Map.of(
            "user_id", userId,
            "experiment", "customer-service-prompt",
            "variant", variant,
            "latency_ms", latency,
            "message_length", message.length(),
            "response_length", response.length()
        ));

        return response;
    }
}
```

### 3.5 评估指标埋点（关键）

**A/B 测试要有"对照组 vs 实验组"的统计指标**：

```java
@Component
public class AbMetricsCollector {

    @Async("metricsExecutor")
    public void recordAsync(Map<String, Object> event) {
        // 1. 发 Kafka
        kafkaTemplate.send("ab-metrics", objectMapper.writeValueAsString(event));

        // 2. 实时入 ClickHouse（OLAP，秒级聚合）
        clickhouseClient.insert("ab_events", event);
    }
}

// 用户反馈
@PostMapping("/feedback")
public void feedback(@RequestBody FeedbackRequest req) {
    Map<String, Object> event = Map.of(
        "event_type", "feedback",
        "user_id", req.userId,
        "conversation_id", req.conversationId,
        "rating", req.rating,
        "experiment", req.experiment,  // 从 cookie/session 取
        "variant", req.variant
    );
    collector.recordAsync(event);
}
```

**ClickHouse 聚合查询**（看 A/B 结果）：

```sql
-- 每个变体的用户满意度
SELECT
    variant,
    count() AS total,
    avg(rating) AS avg_rating,
    countIf(rating >= 4) / count() AS positive_rate
FROM ab_events
WHERE event_type = 'feedback'
  AND timestamp > now() - INTERVAL 7 DAY
GROUP BY variant;

-- 变体对比
SELECT
    variant,
    quantile(0.5)(latency_ms) AS p50,
    quantile(0.99)(latency_ms) AS p99,
    avg(cost) AS avg_cost
FROM ab_events
WHERE timestamp > now() - INTERVAL 7 DAY
GROUP BY variant;
```

### 3.6 统计显著性判断

**别被"看着像有提升"骗了**——必须做假设检验。

```python
# scripts/ab_significance.py
"""
A/B 测试显著性检验（用贝叶斯方法 + 频率派方法双验证）
"""
import numpy as np
from scipy import stats

def check_significance(control_conversions, treatment_conversions,
                        control_total, treatment_total):
    """
    输入：对照组的转化列表 + 实验组的转化列表
    输出：是否显著、置信度、提升幅度
    """
    # 1. 转化率
    p_c = control_conversions / control_total
    p_t = treatment_conversions / treatment_total

    # 2. Z 检验（双比例）
    p_pooled = (control_conversions + treatment_conversions) / (control_total + treatment_total)
    se = np.sqrt(p_pooled * (1 - p_pooled) * (1/control_total + 1/treatment_total))
    z = (p_t - p_c) / se
    p_value = 2 * (1 - stats.norm.cdf(abs(z)))  # 双尾

    # 3. 提升幅度
    lift = (p_t - p_c) / p_c * 100

    # 4. 决策
    is_significant = p_value < 0.05

    return {
        "control_rate": p_c,
        "treatment_rate": p_t,
        "lift_percent": lift,
        "p_value": p_value,
        "is_significant": is_significant,
        "recommendation": "全量" if is_significant and lift > 0 else "继续实验" if not is_significant else "回滚"
    }

# 示例
result = check_significance(
    control_conversions=420, control_total=5000,    # 8.4%
    treatment_conversions=510, treatment_total=5000   # 10.2%
)
print(result)
# {'control_rate': 0.084, 'treatment_rate': 0.102, 'lift_percent': 21.4,
#  'p_value': 0.003, 'is_significant': True, 'recommendation': '全量'}
```

### 3.7 A/B 测试 Checklist

| 阶段 | 检查项 |
|---|---|
| **实验设计** | 假设清晰（"v3 提升 X 指标 Y%"）|
| **实验设计** | 样本量预估（power analysis）|
| **实验设计** | 流量分配合理（5-10% 起步）|
| **实验设计** | 一个实验只改一个变量 |
| **执行** | 用户 ID hash 稳定（不会切组）|
| **执行** | 强制白名单（QA / 内部）|
| **执行** | 埋点完整（曝光/点击/转化/反馈）|
| **分析** | 跑够最小样本量再下结论 |
| **分析** | 统计显著性检验（p < 0.05）|
| **分析** | 长期效果（不能只看 1 天）|
| **决策** | 显著提升 → 全量 |
| **决策** | 无显著差异 → 保持原版或下线 |
| **决策** | 显著下降 → 立即回滚 |

---

## 第四部分：生产部署（Blue-Green / Canary）

### 4.1 三种部署策略对比

| 策略 | 切换速度 | 风险 | 资源消耗 | 适合场景 |
|---|---|---|---|---|
| **Rolling Update** | 慢 | 中 | 1 套 | 常规发布 |
| **Blue-Green** | 瞬间 | 低（立即回滚）| **2 套** | 大版本发布 |
| **Canary** | 渐进 | **最低** | 1.x 套 | **LLM 应用首选** |

**LLM 应用**推荐 **Canary**——风险最低（5% 流量先验证，失败立刻回滚）。

### 4.2 K8s Canary 部署（用 Argo Rollouts）

```yaml
# k8s/rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: agent
  namespace: production
spec:
  replicas: 10

  selector:
    matchLabels:
      app: agent

  strategy:
    canary:
      # 分 5 步灰度
      steps:
        - setWeight: 5      # 5%
        - pause: { duration: 10m }  # 观察 10 分钟
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 15m }
        - setWeight: 100

      # 自动化分析（关键！）
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
        startingStep: 2  # 从 25% 开始分析
        args:
          - name: service-name
            value: agent

  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
        - name: agent
          image: agent:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "2Gi"
              cpu: "1000m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          readinessProbe:
            httpGet:
              path: /api/chat/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10

---
# 自动化分析模板 1: 成功率
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="agent",status=~"2.."}[2m]))
            /
            sum(rate(http_requests_total{service="agent"}[2m]))

---
# 自动化分析模板 2: 延迟
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
  namespace: production
spec:
  metrics:
    - name: p99-latency
      interval: 1m
      count: 5
      successCondition: result[0] < 5000
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="agent"}[2m])) by (le)
            ) * 1000
```

**效果**：
- 部署新版本 → 自动 5% → 10 分钟 → 自动分析指标
- **成功率 < 99% 或 P99 > 5s → 自动暂停 / 回滚**
- 通过 → 25% → 50% → 100%

### 4.3 Istio 流量切分（细粒度控制）

```yaml
# istio/virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: agent
  namespace: production
spec:
  hosts:
    - agent
  http:
    # 5% 流量到新版本
    - match:
        - headers:
            cookie:
              regex: ".*canary=true.*"
      route:
        - destination:
            host: agent
            subset: v2
    - weight:
        - destination:
            host: agent
            subset: v1
          weight: 95
        - destination:
            host: agent
            subset: v2
          weight: 5
```

**5% 流量切到 v2 后，如果监控发现**：
- 错误率飙升 → Istio 一行命令切回 v1
- 一切正常 → 调整 weight 到 25% → 50% → 100%

### 4.4 LLM 应用的"配置热更新"（避免每次发版）

**很多 LLM 配置不需要重启服务**——Prompt / 模型参数 / 工具白名单都应该热更新。

```java
/**
 * 用 Nacos / Apollo / Spring Cloud Config 做配置中心
 */
@RestController
@RequestMapping("/api/admin/config")
public class PromptConfigController {

    @Autowired private NacosConfigService nacos;

    /**
     * 改 Prompt 不需要重启服务
     */
    @PostMapping("/prompt/{name}")
    public String updatePrompt(@PathVariable String name, @RequestBody String content) {
        // 1. 写到 Nacos
        nacos.publishConfig("prompts/" + name + ".md", "DEFAULT_GROUP", content);

        // 2. 推送到所有 Agent 实例（通过 Redis Pub/Sub）
        redisTemplate.convertAndSend("config:prompt:update", name);

        return "Prompt updated, will take effect in 5s";
    }
}

// Agent 实例订阅
@Component
public class ConfigChangeListener {
    @Autowired private PromptLoader promptLoader;

    @EventListener
    public void onPromptUpdate(String promptName) {
        // 重新加载 Prompt
        promptLoader.reload(promptName);
    }
}
```

**效果**：
- PM 改 Prompt → 后台点击发布 → 5 秒内全网生效
- **不需要发版、不需要重启、不需要走 CI/CD**
- 出了问题一键回滚

### 4.5 部署 Checklist

| 类别 | 检查项 |
|---|---|
| **镜像** | 多阶段构建（最终镜像 < 500MB）|
| **镜像** | 非 root 用户运行 |
| **镜像** | 健康检查端点 |
| **镜像** | JVM 调优参数固化 |
| **K8s** | 资源 requests/limits 设置 |
| **K8s** | HPA 自动扩缩容 |
| **K8s** | PodDisruptionBudget（防止驱逐）|
| **K8s** | NetworkPolicy（只允许内网访问）|
| **部署** | Canary 灰度策略 |
| **部署** | 自动回滚机制 |
| **部署** | 配置热更新（不重启）|
| **部署** | 蓝绿环境（生产 + 预发）|
| **数据** | 数据库 migration 自动化 |
| **数据** | 向量库版本管理 |

---

## 第五部分：监控告警（决定挂了能不能知道）

### 5.1 监控的 4 个维度

**LLM 应用监控 = 传统监控 + LLM 特有指标**

```
┌─────────────────────────────────────────────────────┐
│ 1. 基础设施（CPU / 内存 / 磁盘 / 网络）               │
│ 2. 应用指标（QPS / 延迟 / 错误率 / JVM）              │
│ 3. LLM 特有（Token 成本 / 缓存命中率 / 幻觉率）       │
│ 4. 业务指标（用户满意度 / 任务完成率 / 转化率）        │
└─────────────────────────────────────────────────────┘
```

### 5.2 OpenTelemetry + Prometheus + Grafana（事实标准）

**OpenTelemetry 是 2026 年可观测性事实标准**——所有主流厂商（Datadog / New Relic / 阿里云 / 腾讯云）都接。

#### Agent 端埋点

```xml
<!-- OpenTelemetry Java Agent -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.40.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <version>1.40.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.5.0</version>
</dependency>
```

```yaml
# application.yml
otel:
  service:
    name: agent
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  metrics:
    exporter: otlp
  traces:
    exporter: otlp
  logs:
    exporter: otlp
```

#### LLM 专用埋点

```java
@Component
public class LlmTelemetry {

    @Autowired private Tracer tracer;
    @Autowired private Meter meter;

    private final Counter llmCallCounter;
    private final Counter tokenCounter;
    private final Histogram latencyHistogram;

    public LlmTelemetry(Meter meter) {
        this.llmCallCounter = meter.counterBuilder("llm.calls")
            .setDescription("LLM 调用次数")
            .build();
        this.tokenCounter = meter.counterBuilder("llm.tokens")
            .setDescription("Token 用量")
            .build();
        this.latencyHistogram = meter.histogramBuilder("llm.latency")
            .setDescription("LLM 调用延迟")
            .setUnit("ms")
            .build();
    }

    public void recordCall(String model, String tenantId,
                            long promptTokens, long completionTokens,
                            long latencyMs, boolean success) {
        // 1. Counter：调用次数
        llmCallCounter.add(1,
            Attributes.of(
                AttributeKey.stringKey("model"), model,
                AttributeKey.stringKey("tenant"), tenantId,
                AttributeKey.stringKey("status"), success ? "success" : "error"
            )
        );

        // 2. Counter：Token 用量
        tokenCounter.add(promptTokens, Attributes.of(
            AttributeKey.stringKey("model"), model,
            AttributeKey.stringKey("type"), "prompt"
        ));
        tokenCounter.add(completionTokens, Attributes.of(
            AttributeKey.stringKey("model"), model,
            AttributeKey.stringKey("type"), "completion"
        ));

        // 3. Histogram：延迟
        latencyHistogram.record(latencyMs, Attributes.of(
            AttributeKey.stringKey("model"), model
        ));
    }
}
```

#### Trace 链路追踪

```java
@Service
public class TracedChatService {

    @Autowired private Tracer tracer;
    @Autowired private ChatClient chatClient;

    public String chat(String userId, String message) {
        // 创建根 Span
        Span span = tracer.spanBuilder("chat")
            .setAttribute("user.id", userId)
            .setAttribute("message.length", message.length())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // 1. RAG 检索 Span
            Span ragSpan = tracer.spanBuilder("rag.retrieve").startSpan();
            List<Document> docs = vectorStore.similaritySearch(message);
            ragSpan.setAttribute("retrieved.count", docs.size());
            ragSpan.end();

            // 2. LLM 调用 Span
            Span llmSpan = tracer.spanBuilder("llm.call")
                .setAttribute("model", "deepseek-chat")
                .startSpan();
            String response = chatClient.prompt()
                .user(message + "\n\nContext: " + formatDocs(docs))
                .call()
                .content();
            llmSpan.setAttribute("response.length", response.length());
            llmSpan.end();

            span.setAttribute("response.length", response.length());
            return response;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Grafana 看到的效果**（Trace Explorer）：

```
POST /api/agent/chat (1.8s)
├── rag.retrieve (220ms)
│   ├── embed.query (15ms)
│   └── vector.search (200ms, topK=5)
├── llm.call (1.5s)
│   ├── tokenize (5ms)
│   ├── api.request (1.4s, deepseek-chat)
│   └── tokenize (95ms)
└── cache.write (5ms, hit=false)
```

### 5.3 Langfuse（LLM 专用监控）

**Langfuse 是 2026 年 LLM 监控的事实标准**——专门为 LLM 设计，能看到 prompt / completion / 工具调用细节。

```java
@Component
public class LangfuseIntegration {

    @Autowired private LangfuseClient langfuse;

    public void record(String traceId, String userId, String model,
                       String prompt, String completion,
                       long promptTokens, long completionTokens,
                       long latencyMs) {
        langfuse.createTrace(LangfuseTrace.builder()
            .id(traceId)
            .name("llm-chat")
            .userId(userId)
            .model(model)
            .input(prompt)
            .output(completion)
            .usage(LangfuseUsage.builder()
                .promptTokens(promptTokens)
                .completionTokens(completionTokens)
                .build())
            .metadata(Map.of("latency_ms", latencyMs))
            .build());
    }

    public void score(String traceId, String name, double value, String comment) {
        // 用户反馈 / 自动评估 → 评分
        langfuse.createScore(traceId, name, value, comment);
    }
}
```

**Langfuse 看板能看到**：
- 按模型/用户/时间分组的成本趋势
- 每次调用的完整 Prompt 和 Completion
- 用户反馈（点👍/👎）
- A/B 实验对比

### 5.4 告警规则

```yaml
# Prometheus alert rules
groups:
  - name: llm_alerts
    rules:
      # 错误率 > 5%
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Agent 错误率超 5%"

      # P99 延迟 > 8s
      - alert: HighLatency
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent P99 延迟超 8 秒"

      # LLM API 失败
      - alert: LLMApiFailure
        expr: rate(llm_calls_total{status="error"}[5m]) > 0.1
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "LLM API 失败率飙升"

      # 单租户成本异常
      - alert: TenantCostSpike
        expr: increase(llm_cost_usd[1h]) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "租户 {{ $labels.tenant }} 1 小时成本超 $100"

      # 缓存命中率下降
      - alert: LowCacheHitRate
        expr: rate(llm_cache_hits[10m]) / (rate(llm_cache_hits[10m]) + rate(llm_cache_misses[10m])) < 0.3
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "缓存命中率 < 30%"
```

**告警分级**：
- **P0 紧急**（5 分钟内响应）：错误率 > 5%、LLM API 挂
- **P1 重要**（30 分钟内响应）：P99 延迟 > 8s、成本异常
- **P2 一般**（4 小时内响应）：缓存命中率下降
- **P3 提示**（次日处理）：趋势异常

### 5.5 用户反馈闭环

```java
@RestController
@RequestMapping("/api/feedback")
public class FeedbackController {

    @Autowired private LangfuseClient langfuse;
    @Autowired private ClickHouseClient ch;

    /**
     * 用户在 UI 点了"答得不对"
     */
    @PostMapping("/answer")
    public ResponseEntity<?> feedback(@RequestBody FeedbackRequest req) {
        // 1. 存到 ClickHouse（分析用）
        ch.insert("user_feedback", Map.of(
            "trace_id", req.traceId,
            "user_id", req.userId,
            "rating", req.rating,         // 1-5
            "comment", req.comment,
            "timestamp", Instant.now()
        ));

        // 2. 给 Langfuse 评分（关联到具体 trace）
        langfuse.createScore(req.traceId, "user_rating",
            (double) req.rating, req.comment);

        // 3. 差评触发"难例"流程
        if (req.rating <= 2) {
            hardCaseService.addToTestSet(req);
            // 通知值班人
            if (req.rating == 1) {
                alertService.notify("用户给了一星差评", req);
            }
        }

        return ResponseEntity.ok().build();
    }
}
```

**月度复盘**：
- 看 Langfuse 差评 Top 10 的问题类型
- 跟 Ragas 评估对比，看"自动评估 vs 用户感受"是否一致
- 把差评 Top 10 写成"产品需求"反馈给 PM

---

## 第六部分：真实事故复盘——5 个"没做工程化"导致的灾难

### 事故 1：改了一个 Prompt，3 个核心问题答错

**时间**：2024 年某电商 Agent
**症状**：618 大促前，PM 改了客服 Prompt，**没跑评估直接上线**
**结果**：618 当天 3 个高频问题（"发货时间""退换货""发票"）答错，**客服投诉激增 200%**
**复盘**：没有 LLM 评估 + CI 拦截
**解法**：见本文第一部分 + 第二部分

### 事故 2：新版本上线，Token 成本翻 5 倍

**时间**：2025 年某 SaaS
**症状**：开发改了 RAG 召回数量（topK 5 → 50），**没看成本监控**
**结果**：**月度账单从 $15K 涨到 $80K**，CEO 紧急叫停
**复盘**：没有 Token 成本监控 + 告警
**解法**：见本文深挖版 3 第四部分 + 第五部分告警

### 事故 3：Canary 没自动回滚，挂了 1 小时

**时间**：2025 年某金融 Agent
**症状**：新版本有 bug，**Canary 5% 时已经能看出来**，但没自动回滚
**结果**：**整个生产挂了 1 小时**（金融场景合规罚款 50 万）
**复盘**：Canary 部署没有自动分析 + 自动回滚
**解法**：见本文 4.2 节的 Argo Rollouts 配置

### 事故 4：缓存被刷爆，DB 被打挂

**时间**：2024 年某客服 Agent
**症状**：Redis 缓存没设最大内存，**用户问重复问题时缓存无限增长**
**结果**：Redis OOM → 应用连不上 Redis → 所有请求打 DB → **DB 挂了**
**复盘**：缓存没有内存上限 + 监控
**解法**：见本文深挖版 3 第二部分（缓存 L1/L2 配置）

### 事故 5：评估指标全过，但用户吐槽"答得不对"

**时间**：2025 年某法律 Agent
**症状**：Faithfulness 0.96、Answer Relevancy 0.92 都达标，**但用户大量反馈"答得不对"**
**根因**：评估集是工程师自己写的，跟真实用户问题分布差 10 倍
**复盘**：评估集没反映真实业务
**解法**：见本文 1.4 节——**把用户差评自动转成难例测试集**

---

## 第七部分：完整 Checklist

### LLM 评估（决定能不能上）
- [ ] 100+ 条离线测试集（覆盖易/中/难）
- [ ] 测试集每月自动扩充（用户差评 → 难例）
- [ ] 4 类指标全跑：生成质量 + RAG 检索 + Agent 行为 + 性能成本
- [ ] 评估结果写入 CI（回归 fail 直接拦截）
- [ ] Prompt 跟代码一样版本管理 + 评估

### CI/CD（决定能不能快上）
- [ ] PR 触发自动评估
- [ ] 评估通过才能 merge
- [ ] 自动化构建 Docker 镜像
- [ ] 镜像推送到私有 Registry
- [ ] Staging 环境自动部署
- [ ] 回归测试 + 性能测试在 Staging 跑

### A/B 测试（决定能不能验证更好）
- [ ] 用户 ID hash 稳定路由
- [ ] 一个实验只改一个变量
- [ ] 强制白名单（QA / 内部）
- [ ] 完整埋点（曝光/点击/转化/反馈）
- [ ] 统计显著性检验（p < 0.05）
- [ ] 长期效果追踪（不能只看 1 天）

### 生产部署（Blue-Green / Canary）
- [ ] Canary 灰度策略（5% → 25% → 100%）
- [ ] 自动分析指标（成功率、延迟）
- [ ] 自动回滚机制
- [ ] 配置热更新（Prompt / 参数不重启）
- [ ] 蓝绿环境（生产 + 预发）
- [ ] 数据库 migration 自动化
- [ ] K8s HPA 自动扩缩容
- [ ] K8s PodDisruptionBudget
- [ ] K8s NetworkPolicy

### 监控告警（决定挂了能不能知道）
- [ ] OpenTelemetry 全链路追踪
- [ ] 4 维度监控：基础设施 + 应用 + LLM 特有 + 业务
- [ ] Langfuse LLM 专用看板
- [ ] P0/P1/P2/P3 告警分级
- [ ] 用户反馈闭环（差评自动入测试集）
- [ ] 月度复盘（自动评估 vs 用户感受）

---

## 写在最后

**这 5 篇深挖版合在一起，构成了企业级 Agent 工程师的"完整技术栈"**：

| 你要解决的问题 | 看哪一篇 |
|---|---|
| 怎么用 Java 写 Agent | 深挖版 1（Spring AI）|
| 怎么保证企业级安全 | 深挖版 2（权限）|
| 怎么省 70% 成本 | 深挖版 3（性能成本）|
| 怎么让 RAG 答得准 | 深挖版 4（RAG 调优）|
| 怎么上线 + 怎么不发版就出事故 | **深挖版 5（评估+部署+A/B）** ← 你刚看的 |

**最后送你 3 句话，是这 5 篇深挖版的核心精神**：

1. **没有评估的 Agent 调优，是在猜**——评估是一切的起点
2. **没有 A/B 的版本更新，是在赌**——赌你的新版本更好，输不起
3. **没有监控的生产部署，是在裸奔**——出事故了你都不知道

**这 3 件事，今天就可以开始做**：
- **今天**：写 20 条测试集，跑第一次评估
- **这周**：把评估加到 CI
- **这个月**：做一次完整的 Canary 发布

**从今天起，你就是"用工程化思维做 LLM 应用"的工程师了。**

---

