# 企业级 Agent 面试题专题 2：RAG 深度 + 向量检索（30 题）

> 日期：2026-06-10
> 适用：Java 后端转 Agent 工程师 / 1-3 年 RAG 经验
> 配套：深挖版 4（RAG 调优）、深挖版 10（向量数据库）

---

## 第一部分：RAG 基础（8 题）

### ⭐ Q1：什么是 RAG？为什么需要 RAG？

**参考答案**：

**RAG（Retrieval-Augmented Generation，检索增强生成）** = 先从知识库**检索**相关内容，再把内容 + 问题一起给 LLM **生成**回答。

**为什么需要 RAG**——LLM 自身的三大局限：

| 局限 | 说明 | RAG 怎么解决 |
|---|---|---|
| **幻觉（Hallucination）** | LLM 一本正经胡说 | **用真实数据回答** |
| **知识陈旧** | 训练数据有截止时间 | **实时检索**最新数据 |
| **私域知识** | 不知道企业内部数据 | **外挂企业知识库** |
| **专业深度** | 通用模型不够专 | **外挂专业文档** |

**对比 Fine-tuning（微调）**：

| 维度 | RAG | Fine-tuning |
|---|---|---|
| **成本** | 低（不用训练）| 高（GPU 训练）|
| **数据更新** | **实时**（加新文档就行）| 重新训练 |
| **可解释** | ✅ 看得到检索了啥 | ❌ 黑盒 |
| **幻觉率** | 低（基于事实）| 仍可能幻觉 |
| **适用** | 知识密集型（客服/法律/医疗）| **风格 / 格式适配** |

**加分项**：
- 提到 **RAG 不是银弹**——仍有失败模式（检索不到 / 检索不准 / 上下文超长）
- 提到 **GraphRAG / Agentic RAG** 是 2026 年的演进方向
- 提到 **RAG 评估**（Ragas / DeepEval）

---

### ⭐ Q2：RAG 的完整流程是什么？每一步的输入输出？

**参考答案**：

**完整流程**（5 步）：

```
1. 文档加载 (Document Loading)
   输入：PDF/Word/Markdown/网页/数据库
   输出：原始文本
       ↓
2. 文档切分 (Chunking)
   输入：长文本
   输出：短 chunks (典型 200-1000 token)
       ↓
3. 向量化 (Embedding)
   输入：chunks
   输出：1024 维向量
       ↓
4. 索引存储 (Indexing)
   输入：向量 + 元数据
   输出：写入向量数据库
       ↓
[用户提问时]
5. 检索 + 生成 (Retrieval + Generation)
   用户问题 → 向量化 → 检索 topK → 拼 Prompt → LLM 生成
```

**每步关键决策**：

| 步骤 | 关键决策 | 经验值 |
|---|---|---|
| **加载** | 选什么 Loader | PDF 用 PDFBox，Word 用 POI |
| **切分** | 切多大、是否带 overlap | **512 token + 50 char overlap** |
| **向量化** | 用什么模型 | 中文 BGE-M3，英文 OpenAI |
| **索引** | 用什么索引 | HNSW（默认）+ 标量字段 |
| **检索** | 召回 topK + 过滤 | **top 50 + 多路召回** |
| **生成** | Prompt 怎么写 | **"基于以下上下文..."+ 引用编号** |

**加分项**：
- 提到 **离线 / 在线两套 pipeline**——离线处理新文档，在线响应用户
- 提到 **增量更新**——避免每次全量重建
- 提到 **失败回退**——检索为空时怎么办

---

### ⭐⭐ Q3：RAG 跟 Fine-tuning、RAG + Fine-tuning 怎么选？

**参考答案**：

**决策树**：

```
你的需求是？
    │
    ├── 需要"实时知识"（新闻 / 政策 / 内部数据）
    │   └── ✅ 用 RAG
    │
    ├── 需要"专业风格"（客服话术 / 法务口吻）
    │   └── ✅ 用 Fine-tuning
    │
    ├── 需要"既要专业风格，又要有实时知识"
    │   └── ✅ 用 RAG + Fine-tuning（组合）
    │
    ├── 通用任务（写作 / 翻译 / 总结）
    │   └── ✅ 直接用基础模型 + Few-shot Prompt
    │
    └── 完全冷门 / 没人写过
        └── ✅ 先用 RAG（冷启动快），再考虑 Fine-tuning
```

**RAG + Fine-tuning 组合**：
- **Fine-tuning**学"说话方式"（语气、格式、思维模式）
- **RAG**给"实时信息"（事实、数据）

**真实案例**：
- **客服 Agent**：FT 学会客服话术 + RAG 查知识库
- **法务 Agent**：FT 学法务严谨语气 + RAG 查最新判例
- **代码 Agent**：FT 学代码风格 + RAG 查内部库 API

**加分项**：
- 提到 **Fine-tuning 的成本**——百万级 token 起步，几万元 / 次
- 提到 **Fine-tuning 的失败模式**——过拟合、灾难性遗忘
- 提到 **2026 年趋势**——**RAG 为主，FT 为辅**（Anthropic / OpenAI 都推荐）

---

### ⭐⭐ Q4：什么是 Chunking？常见的分块策略有哪些？

**参考答案**：

**Chunking = 把长文档切成短块**——RAG 最关键的步骤之一。

**为什么需要 Chunking？**
1. **超 LLM 上下文窗口**——一个 PDF 可能几百万 token
2. **检索粒度**——大段检索不准确
3. **成本**——大块浪费 token

**8 种分块策略**：

| 策略 | 原理 | 适合 |
|---|---|---|
| **Fixed-size** | 固定 token 数切分 | **快速原型** |
| **Recursive** | 按段落 / 句子 / 词 递归切 | **通用默认** |
| **Document-based** | 按章节 / 页 | 有清晰结构的文档 |
| **Semantic** | 按语义相似度切 | 质量要求高 |
| **Sentence-based** | 按句子 | 短文本 |
| **LLM-based** | 用 LLM 智能切 | **SOTA 质量**（但贵 100x）|
| **Late Chunking** | 先 embed 再按 token 位置切 | **2026 新趋势** |
| **Hierarchical** | 多层级（段 / 子段）| 复杂文档 |

**经验值**：
- **chunk_size = 512 token**
- **overlap = 50 char**（**注意：2026 年新研究表明 overlap 提升不明显**）
- **Recursive** 是 2026 年默认推荐

**加分项**：
- 提到 **2026 年 1 月 Weaviate 研究**——**分块策略 > Embedding 模型**（同 retriever 差距 9%）
- 提到 **chunk overlap 反直觉发现**——重叠对召回率提升不明显
- 提到 **按文档类型路由分块**——PDF 用 page-level，代码用 code-aware

**Java 视角**：
```java
// Spring AI 1.0 内置 TokenTextSplitter
TokenTextSplitter splitter = new TokenTextSplitter(512, 50, 5, 10000);
List<Document> chunks = splitter.split(document);

// 自定义 recursive splitter
RecursiveCharacterTextSplitter recursive = new RecursiveCharacterTextSplitter(
    512, 50, List.of("\n\n", "\n", "。", ".", " ")
);
List<Document> chunks = recursive.split(document);
```

---

### ⭐⭐ Q5：什么是 Embedding？常用的 Embedding 模型有哪些？中文用什么？

**参考答案**：

**Embedding = 把文本映射到稠密向量**（详见专题 1 Q5）。

**2026 年主流 Embedding 模型**：

| 模型 | 维度 | 厂商 | 适合 |
|---|---|---|---|
| **BGE-M3** | 1024 | BAAI | **中文首选**（多语言、8192 输入）|
| **BGE-large-zh-v1.5** | 1024 | BAAI | 中文 |
| **text-embedding-3-large** | 3072 | OpenAI | 英文、多语言 |
| **text-embedding-v3** | 1024 | Qwen（DashScope）| 中文 |
| **GTE-Qwen2-7B** | 3584 | Alibaba | 大模型嵌入 |
| **bge-m3** | 1024 | BAAI | 2026 多语言首选 |
| **Cohere embed-v3** | 1024 | Cohere | 英文 |
| **Jina v3** | 1024 | Jina | 长文本（8K）|

**中文场景实测对比**（2026 年 MTEB 中文榜）：

| 模型 | Recall@5 | 速度 |
|---|---|---|
| **BGE-M3** | **92%** | 300 docs/s |
| text-embedding-3-large | 76% | 500 docs/s |
| text-embedding-v3 | 88% | 400 docs/s |
| BGE-large-zh-v1.5 | 85% | 280 docs/s |

**怎么选？**
- **中文为主** → BGE-M3（开源、可本地部署）
- **英文为主** → OpenAI text-embedding-3-large
- **多语言** → BGE-M3（100+ 语言）
- **不想自己部署** → Qwen text-embedding-v3（DashScope API）

**加分项**：
- 提到 **MTEB Leaderboard**——权威评测
- 提到 **BGE-M3 的 3 大特性**：多语言（100+）、多功能（dense/sparse/colbert）、长文本（8192）
- 提到 **BGE-M3 在企业场景的 Recall 优势**（比 OpenAI 高 16 个百分点）
- 提到 **OpenAI v3 支持维度缩减**——3072 → 256 维，损失 < 1%

**Java 视角**：
```java
// Spring AI 用 BGE-M3
@Bean
public EmbeddingModel bgeM3Model() {
    return new OllamaEmbeddingModel(
        OllamaApi.builder().baseUrl("http://localhost:11434").build(),
        OllamaOptions.builder()
            .model("bge-m3")
            .build()
    );
}

// 用 Qwen DashScope
@Bean
public EmbeddingModel qwenEmbeddingModel() {
    return new DashScopeEmbeddingModel(
        new DashScopeApi(System.getenv("DASHSCOPE_API_KEY")),
        MetadataMode.EMBED,
        DashScopeEmbeddingOptions.builder()
            .withModel("text-embedding-v3")
            .withDimensions(1024)
            .build()
    );
}
```

---

### ⭐⭐ Q6：什么是 HNSW？为什么大家都用 HNSW？

**参考答案**：

**HNSW（Hierarchical Navigable Small World）= 分层导航小世界图**——一种**近似最近邻（ANN）** 索引算法。

**为什么需要 ANN？**
- **暴力搜索**：100 万向量 × 1024 维 = 10 亿次运算（**10 秒**）
- **HNSW 索引**：预计算"近邻关系"，查询时只走图（**< 10ms**）

**HNSW 原理**（类比跳表）：

```
        层级 3:  ●─────────●
                 │         │
        层级 2:  ●───●───●───●───●
                 │   │   │   │   │
        层级 1:  ●─●─●─●─●─●─●─●─●─●─●
                 │ │ │ │ │ │ │ │ │ │ │
        层级 0:  ●●●●●●●●●●●●●●●●●●●●●●●（所有向量）

查询时：从最顶层开始走，每层贪心往下跳
```

**类比跳表（Skip List）**：
- 顶层稀疏，跳跃快
- 底层稠密，精确搜
- **O(log N) 时间复杂度**

**HNSW 三大参数**：

| 参数 | 含义 | 推荐值 | 影响 |
|---|---|---|---|
| **M** | 每个节点的边数 | 8-32 | 越大越准，但内存越大 |
| **efConstruction** | 建索引时的搜索深度 | 100-400 | 越大索引质量越好 |
| **ef** | 查询时的搜索深度 | 50-300 | 越大召回率越高 |

**实战**：

```python
# Milvus 配置
index_params = {
    "index_type": "HNSW",
    "metric_type": "COSINE",
    "params": {
        "M": 16,
        "efConstruction": 200
    }
}

# 查询时
search_params = {
    "params": {"ef": 100}
}
```

**为什么大家都用 HNSW？**

| 优势 | 说明 |
|---|---|
| **速度快** | 100-1000x 加速 |
| **准确率高** | > 99% recall@10 |
| **支持增删** | IVF 增量更新麻烦 |
| **工业验证** | Milvus / Qdrant / Weaviate 都用 |
| **成熟** | 2016 年提出，10 年优化 |

**加分项**：
- 提到 **HNSW 内存大**——1000 万向量 × 1024 维 × 4B = **40 GB**
- 提到 **HNSW 被刷爆是常见问题**——一定要加 `max_size` 限制
- 提到 **HNSW 不适合超大规模**——10 亿级用 IVF / ScaNN
- 提到 **HNSW 不支持标量预过滤**——这是 Qdrant / Weaviate 优化的方向

---

### ⭐⭐ Q7：什么是混合检索？为什么需要混合检索？

**参考答案**：

**混合检索 = 向量检索 + 关键词检索（BM25）+ 融合排序**。

**为什么需要？**——向量检索的 3 大失败模式：

| 失败 | 例子 | 向量检索 | 关键词检索 |
|---|---|---|---|
| **专有名词** | "iPhone 15 Pro Max 256G" | 召回弱（语义匹配不到）| ✅ 命中 |
| **错误拼写** | "GPT-55"（用户写错）| 召回弱 | ❌ 失败 |
| **精确型号** | "型号 XYZ-1234" | 召回弱 | ✅ 命中 |
| **概念性查询** | "怎么退款" | ✅ 命中 | ❌ 召回弱 |
| **同义词** | "汽车 ↔ 车辆" | ✅ 命中 | ❌ 召回弱 |

**混合检索的精髓**：**用向量召回"概念"，用关键词召回"精确"**。

**完整架构**：

```
        用户查询
          │
   ┌──────┴──────┐
   ↓             ↓
向量检索     关键词检索 (BM25)
(Milvus)     (Elasticsearch)
   ↓             ↓
  Top 50       Top 50
   └──────┬──────┘
          ↓
     RRF 融合 / Cross-Encoder Rerank
          ↓
        Top 10
```

**3 种融合方法**：

| 方法 | 公式 | 优点 | 缺点 |
|---|---|---|---|
| **RRF** | `score = Σ 1/(k + rank)` | 简单，无参数 | 没考虑原始分数 |
| **加权平均** | `score = α·vector + β·bm25` | 灵活 | α / β 难调 |
| **Cross-Encoder** | 用模型重排序 | 准确 | 慢 |

**加分项**：
- 提到 **RRF 中 k = 60 是经验值**（Cormack et al. 2009）
- 提到 **Milvus 2.5+ 内置 hybrid search**（v0.4+）
- 提到 **混合检索在企业 RAG 中** 比纯向量高 **15-20% 召回率**

**Java 视角**：
```java
// Spring AI 1.0 混合检索
HybridQuery hybridQuery = new HybridQuery(query, 50, 50, 10);
List<Document> results = hybridSearchService.hybridSearch(hybridQuery);
```

---

### ⭐⭐ Q8：什么是 Rerank？为什么需要 Rerank？

**参考答案**：

**Rerank = 在向量检索后，用更精确（但更贵）的模型重新排序**。

**为什么需要？**

| 阶段 | 模型 | 速度 | 准确率 |
|---|---|---|---|
| **Bi-Encoder 召回** | Embedding 模型（BGE-M3）| **快**（10ms/千条）| 中等（80-90%）|
| **Cross-Encoder 精排** | Cross-Encoder | **慢**（300ms/千条）| 高（> 95%）|

**原理**：

```
Bi-Encoder：q 和 d 独立编码 → 算 cosine 相似度
            快但不准（不知道 query 和 doc 的"关系"）

Cross-Encoder：q 和 d 一起输入 → 输出 0-1 相关性分数
              慢但准（能学到 query-doc 交互）
```

**实操**：

```java
// 第一阶段：Bi-Encoder 召回 top 50
List<Document> recalled = vectorStore.similaritySearch(query, 50);

// 第二阶段：Cross-Encoder 精排 top 10
CrossEncoderReranker reranker = new BgeReranker("BAAI/bge-reranker-large");
List<Document> reranked = reranker.rerank(query, recalled, 10);
```

**2026 年主流 Rerank 模型**：

| 模型 | 厂商 | 速度 | 准确率 |
|---|---|---|---|
| **bge-reranker-large** | BAAI | 中 | 高 |
| **bge-reranker-v2-m3** | BAAI | **快** | 高 |
| **cohere-rerank-3** | Cohere | 快 | **最高** |
| **Jina Rerank** | Jina | 快 | 高 |
| **gte-rerank** | Alibaba | 中 | 高 |

**加分项**：
- 提到 **Cohere Rerank 准确率最高**但贵
- 提到 **本地化**用 BGE-Reranker（开源）
- 提到 **Rerank 不是越多越好**——**top 50 精排到 top 10**就够，再多收益递减
- 提到 **Lost-in-the-Middle**——Rerank 后还要重新排序，避免把重要信息放中间

---

## 第二部分：RAG 进阶（12 题）

### ⭐⭐⭐ Q9：什么是 Re-ranking 的两阶段？什么时候只用召回不用 Rerank？

**参考答案**：

**两阶段架构**：
1. **Bi-Encoder 召回**：top 50-100（覆盖广）
2. **Cross-Encoder 精排**：top 5-10（精确）

**什么时候只用召回不用 Rerank？**

| 场景 | 用 Rerank？ | 原因 |
|---|---|---|
| **知识库 < 1 万条** | ❌ 不用 | 召回 top 5 准确率 > 95%，Rerank 收益小 |
| **RAG 聊天** | ✅ 用 | 用户期待高准确率 |
| **简单搜索（FAQ）** | ❌ 不用 | 关键词命中就够 |
| **企业知识库** | ✅ 用 | 准确率敏感 |
| **大文档摘要** | ✅ 用 | 选哪些 chunk 重要 |

**什么时候 Rerank 也不行？**

- **检索为空**（topK 全是垃圾）——Rerank 也救不了
- **问题太模糊**——Rerank 只能"挑好的"，不能"补缺失的"

**Rerank 失败的常见原因**：
- 召回太少（top 10 → Rerank 后还是不够）
- 模型未针对你的领域微调
- 输入太长被截断

**加分项**：
- 提到 **LLM Rerank**——用 LLM 直接打分（GPT-4 / Claude），准确率高但贵
- 提到 **ColBERT / ColPali**——多向量表示，介于 Bi-Encoder 和 Cross-Encoder 之间

---

### ⭐⭐⭐ Q10：RAG 系统的准确率怎么评估？有哪些指标？

**参考答案**：

**2026 年主流 RAG 评估指标**（4 大类）：

#### 1. 检索质量指标

| 指标 | 含义 | 公式 |
|---|---|---|
| **Recall@K** | Top K 中相关文档占比 | 相关数 / 总相关数 |
| **Precision@K** | Top K 中相关文档占比 | 相关数 / K |
| **MRR** | 第一个相关文档的倒数排名 | 1 / rank |
| **NDCG@K** | 考虑排序位置的质量 | 复杂公式 |

#### 2. 生成质量指标（4 类）

| 维度 | 指标 | 评估什么 |
|---|---|---|
| **忠实度** | Faithfulness | 答案是否基于上下文 |
| **答案相关** | Answer Relevancy | 答案是否切题 |
| **上下文相关** | Context Relevancy | 召回的是否相关 |
| **上下文召回** | Context Recall | 该召的都召了没 |

#### 3. 端到端指标

- **人工评估**（最准）：让业务人员打分
- **A/B 测试**：线上分流对比
- **用户反馈**：点赞/点踩

#### 4. 工程指标

- **延迟 P50/P95/P99**
- **Token 成本**（每次查询多少 token）
- **错误率**

**评估工具**：

| 工具 | 厂商 | 特点 |
|---|---|---|
| **Ragas** | 开源 | 学术参考、4 个核心指标 |
| **DeepEval** | Confident AI | **2026 最强**、CI/CD 集成 |
| **Promptfoo** | 开源 | 红队 + 评估 |
| **TruLens** | 开源 | 简单好用 |
| **LangSmith** | LangChain | 商业版 |
| **Braintrust** | Braintrust | 生产监控 + 评估 |

**Java 视角**：
```java
// DeepEval 集成（4 步）
1. mvn dependency: spring-boot-starter-test + deepeval-java
2. 构造测试集
3. @Test void testRag() {
       AnswerRelevancyMetric metric = new AnswerRelevancyMetric();
       double score = metric.measure(testCase);
       assertTrue(score > 0.8);
   }
4. mvn test
```

**加分项**：
- 提到 **"检索-生成"分两段评估**——别只看答案质量
- 提到 **测试集最少 100-200 个 QA**——少没统计意义
- 提到 **LLM-as-a-Judge** 的偏差——Claude 评 GPT 答案 vs GPT 评 Claude 答案分数不同

---

### ⭐⭐⭐ Q11：RAG 的失败模式有哪些？怎么诊断？

**参考答案**：

**5 大失败模式 + 诊断方法**：

#### 失败 1：检索不到（Context Miss）

**症状**：用户问的问题，召回 top 10 全是无关内容

**诊断**：
```java
// 把 top 10 全部打印，看跟问题是否相关
List<Document> retrieved = vectorStore.search(query, 10);
retrieved.forEach(d -> log.info("Score={}, Content={}", d.getScore(), d.getContent().substring(0, 50)));
```

**根因**：
- 文档没切分好（chunk 太大错过细节）
- Embedding 模型不适合
- 问题太冷门，文档里没有

**修复**：
- 检查文档摄入是否完整
- 换 Embedding 模型
- 调小 chunk_size

#### 失败 2：检索不准（Wrong Context）

**症状**：召回了内容，但跟问题不相关

**诊断**：
- 看召回了什么 vs 期待召回什么
- 是不是 chunk 边界错了
- 是不是元数据过滤错了

**修复**：
- 加 BM25 混合检索
- 加 Rerank
- 优化 chunking 策略

#### 失败 3：上下文超长（Context Overflow）

**症状**：top 50 + 完整内容 > 模型上下文

**修复**：
- 减少 topK
- 加 context 压缩
- 拆分成多步

#### 失败 4：答案不基于上下文（Hallucination）

**症状**：LLM 用了常识回答，不基于检索内容

**修复**：
- Prompt 加约束："严格基于以下内容回答，不知道就说不知道"
- 用更小的 Temperature（0）
- 用 LLM-as-a-Judge 自动评估

#### 失败 5：引用错误（Wrong Attribution）

**症状**：答案正确，但引用的文档不对

**修复**：
- 让 LLM 在答案里**加引用编号**（[1]、[2]）
- 后处理验证引用

**加分项**：
- 提到 **Ragas 4 指标能自动发现失败模式**
- 提到 **Golden Test Set**——准备 100 个标准 QA 反复测
- 提到 **A/B 测试**——线上验证

---

### ⭐⭐⭐ Q12：什么是 RAG 调优的"6 轮方法"？给个真实案例

**参考答案**：

**6 轮方法**（来自深挖版 4 的法律咨询 Agent 真实案例）：

**初始状态**：准确率 60%

#### Round 1：基础检查
- 文档切分（512 token + 50 overlap）
- Embedding（BGE-M3）
- 检索 top 10
- **结果**：60%（基线）

#### Round 2：优化 Embedding
- 换了 BGE-M3 + BGE-Reranker
- **结果**：60% → 72%（+12%）

#### Round 3：优化 Chunking
- 改用 hierarchical chunking（段 + 子段）
- 答案用子段，但引用返回段
- **结果**：72% → 80%（+8%）

#### Round 4：加混合检索
- 向量 + BM25 + RRF
- **结果**：80% → 86%（+6%）

#### Round 5：加 HyDE + Multi-Query
- HyDE：LLM 先生成"假设答案"，用假设答案去检索
- Multi-Query：一个问题生成 5 个变体
- **结果**：86% → 92%（+6%）

#### Round 6：加 Cross-Encoder Rerank + Prompt 优化
- 召回到 top 50 → Rerank 到 top 5
- Prompt 加"不知道就说不知道"
- **结果**：92% → 95%（+3%）

**总提升**：60% → 95%（**+35%**）

**关键洞察**：
- **Chunking 和 Rerank 收益最大**
- **Embedding 换了收益反而小**（同领域）
- **最后 5% 提升最贵**（要改 Prompt / Fine-tune）

**加分项**：
- 提到 **HyDE 适合"问题很短"场景**（用户问"怎么退款"时）
- 提到 **Multi-Query 适合"问题模糊"**
- 提到 **2026 年新趋势**——Agentic RAG（让 LLM 决定检索几次、检索什么）

---

### ⭐⭐⭐ Q13：什么是 HyDE？什么是 Multi-Query？什么时候用？

**参考答案**：

#### HyDE（Hypothetical Document Embeddings）

**核心思想**：**让 LLM 先生成"假设答案"**——用假设答案的 embedding 去检索。

**为什么有效？**
- 用户问题短（"怎么退款"）→ embedding 弱
- 假设答案长、像真实文档 → embedding 强
- 假设答案跟真实答案"语义更接近"——因为都是"答案形态"

```java
String userQuery = "怎么退款";
String hypothetical = chatClient.prompt()
    .user("请简短回答：" + userQuery)
    .call()
    .content();
// 假设答案："要退款需要先在订单页面申请..."

// 用假设答案去检索
List<Document> results = vectorStore.similaritySearch(hypothetical, 10);
```

**适合**：问题很短、用户表达不清

#### Multi-Query

**核心思想**：**一个问题生成多个变体**，每个都检索，合并去重。

```java
String userQuery = "怎么退款";

List<String> variants = chatClient.prompt()
    .user("请把这个问题改写成 5 个不同的问法：" + userQuery)
    .call()
    .content();
// 变体：["退款流程", "如何申请退款", "退货政策", "退款条件", "退款到账时间"]

// 每个变体都检索
List<Document> all = new ArrayList<>();
for (String v : variants) {
    all.addAll(vectorStore.similaritySearch(v, 10));
}
List<Document> deduped = deduplicate(all);
```

**适合**：用户表达模糊、可能有多种问法

#### 对比

| 方法 | 优势 | 劣势 | 适合 |
|---|---|---|---|
| **HyDE** | 简单、效果稳 | 多 1 次 LLM 调用 | **短问题** |
| **Multi-Query** | 召回率高 | 多 5 次 LLM 调用 | **模糊问题** |
| **Step-Back** | 抽象问题后召回更准 | 多 1 次 LLM 调用 | **复杂/专业问题** |