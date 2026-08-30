# 代码评审报告

> **评审对象**：`src/code/ai/practice/agent/plan.md` (Agent Plan 机制文档)
> **评审角色**：资深架构师 / P7 技术专家
> **评审性质**：技术文档与系统设计评审（非传统代码 CR，但遵循同等严格的逻辑严谨性与工程落地标准）

---

## 1. 总体评价

*   **是否可用于生产环境**：**否**。当前文档属于“概念科普”或“初级设计笔记”阶段，缺乏工程落地的关键细节（如状态机定义、数据持久化策略、失败降级策略、具体协议定义）。直接照此实现极易导致 Agent 陷入死循环或状态不可控。
*   **综合风险等级**：**高**。
    *   主要风险在于 **Correction（纠偏）机制的模糊性** 和 **DAG 执行引擎的缺失**。
    *   LLM Agent 的核心难点在于**状态一致性**和**终止条件**，当前文档对这些核心难点的覆盖过于浅显。
*   **是否达到 P6 / P7 工程能力预期**：**否**。
    *   **P6 级别**：能写出 ReAct 循环，理解 Prompt Engineering，但缺乏对复杂状态管理的认知。
    *   **P7 级别**：需要设计具备**幂等性、可恢复性、可观测性**的 Plan 执行引擎。当前文档仅描述了“想做什么”，未描述“怎么做才稳健”。

---

## 2. 代码正确性与潜在 Bug（逻辑/设计缺陷表格）

| 严重级别 | 问题位置 | 问题描述 | 技术依据 | 可能后果 | 生产风险标签 | 修复建议 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **致命** | `Correction` 章节 / 整体架构 | **纠偏机制缺乏“最大重试/循环次数限制”与“终止条件”定义** | LLM 输出具有随机性。若 `Correction` 中的 `Replan` 或 `Clarify` 没有全局计数器限制，LLM 可能在“生成-评估-修正”的死循环中无限消耗 Token 和计算资源。 | 系统雪崩：API 调用费用失控，用户请求超时（Timeout），CPU/内存耗尽。 | **线上事故** / **资损** | 引入全局 `LoopCounter` 和 `MaxIterations`。在 Prompt 或代码层硬性限制最大 Replan 次数（如 3 次），超限后强制进入 `Fallback`（返回错误给用户）或 `Interruption`。 |
| **严重** | `DAG式 Plan` 章节 | **DAG 数据结构定义不完整，缺乏状态字段与错误传播机制** | 文档中 DAG 仅定义了 `nodes` 和 `edges`，未定义节点状态（`PENDING`, `RUNNING`, `SUCCESS`, `FAILED`）。在分布式或异步执行中，若无状态机，无法判断依赖节点是否真正完成，也无法实现“失败短路”（Fail-fast）。 | 数据不一致：下游节点可能读取未生成或错误的上游数据；死锁：若上游失败但状态未更新，下游永远等待。 | **数据不一致** / **SLA 下降** | 完善 DAG 节点模型：`{ id, status, result, error, depends_on }`。实现标准 DAG 执行引擎逻辑：仅当所有 `depends_on` 状态均为 `SUCCESS` 时，才将节点置为 `READY`。 |
| **严重** | `Checkpoint` 章节 | **检查点（Checkpoint）缺乏机器可读的“通过/失败”判定标准** | 文档中的 `checks` 是自然语言描述（如 "是否区分事实和推测"）。LLM 评估这些自然语言标准时，输出往往是非确定性的（True/False 边界模糊）。若直接用于程序逻辑分支，会导致流程跳跃或误判。 | 逻辑错误：Agent 可能在一个未通过验证的步骤继续执行，导致最终结果违背硬约束（如虚构内容）。 | **数据错误** / **用户体验极差** | 将 Checkpoint 转化为**结构化断言（Structured Assertions）**。例如，要求 LLM 输出 JSON：`{ "check_id": "fact_check", "passed": false, "reason": "..." }`。代码层根据 `passed` 布尔值决定流程走向，而非依赖 LLM 的“自觉”。 |
| **一般** | `Context` 章节 | **上下文窗口管理缺失，未提及 Memory 压缩或切片策略** | 随着 Plan 步骤增加，对话历史（Context）会迅速膨胀。文档未提及如何管理长上下文（如滑动窗口、摘要压缩、RAG 检索增强）。 | 性能下降：Token 消耗指数级增长；幻觉增加：由于上下文过长，LLM 注意力分散，导致关键约束被忽略。 | **成本上升** / **质量下降** | 引入 Context Manager。对 `known_facts` 和 `missing_info` 进行定期摘要压缩；将长对话历史转化为向量存入向量数据库，仅在需要时检索 Top-K。 |
| **一般** | `Goal` 章节 | **目标（Goal）缺乏量化指标与验收标准（Acceptance Criteria）** | "优化项目经历" 是模糊目标。`success_criteria` 列表虽好，但缺乏可计算的量化指标（如：字数限制、关键词覆盖率、相似度阈值）。 | 评估失效：Plan Verifier 难以准确打分，导致迭代优化方向偏离。 | **功能无效** | 为每个 Goal 绑定至少一个可量化的 KPI。例如：`"metric": "keyword_coverage", "threshold": 0.8, "keywords": ["微服务", "高并发"]`。 |
| **可优化** | `DAG式 Plan` 章节 | **并行执行策略未提及** | DAG 天然支持并行（无依赖关系的节点可并发执行）。文档未提及利用 DAG 结构进行并行调用以降低整体 Latency。 | 性能浪费：串行执行独立步骤，导致整体响应时间变长。 | **SLA 下降** | 在执行引擎中实现任务调度器，识别无依赖节点，通过 `CompletableFuture` (Java) 或 `async/await` (Python/JS) 并发执行。 |

---

## 3. 其他问题清单

### 3.1 缺乏异常处理与幂等性设计（严重）
*   **描述**：文档中 `Correction` 提到了“回滚”，但未定义如何回滚。LLM 生成的文本不是数据库事务，无法简单 `ROLLBACK`。
*   **风险**：如果 Step 3 失败，Step 4 已经基于 Step 3 的**错误中间结果**生成了内容。回滚 Step 3 后，Step 4 的状态是什么？需要重新执行吗？
*   **建议**：明确“状态快照”机制。每个 Checkpoint 通过后，将当前上下文（Context + 已生成内容）持久化到数据库或 Redis。回滚时，从最近的快照恢复状态，并清除后续步骤的结果。

### 3.2 Prompt 工程缺乏 Anti-Hallucination（反幻觉）具体策略（严重）
*   **描述**：文档提到 `Context` 中区分“可信”与“不可信”信息，但未给出具体 Prompt 策略。
*   **风险**：LLM 倾向于“编造”以填补信息空白。
*   **建议**：在 Prompt 中强制加入 CoT（Chain of Thought）指令：
    > "你必须严格基于 <known_facts> 中的信息回答。如果 <missing_info> 中的关键信息缺失，禁止推测，必须输出 'INFO_MISSING' 并停止。"

### 3.3 评估指标（Metrics）缺乏实现细节（一般）
*   **描述**：表格中提到的 `Plan 完成率` 等指标，未定义如何埋点。
*   **风险**：开发无法落地监控，无法验证 Plan 质量。
*   **建议**：
    *   定义统一的 `AgentTrace` 日志结构，包含 `trace_id`, `step_id`, `start_time`, `end_time`, `status`, `tokens_used`。
    *   通过 AOP 或装饰器模式自动埋点，推送至 Prometheus。

### 3.4 用户交互（Human-in-the-loop）缺乏超时与默认策略（一般）
*   **描述**：`Clarify` 机制依赖用户输入。
*   **风险**：用户不回复或长时间无响应，系统挂起。
*   **建议**：设定 `ClarifyTimeout`（如 5 分钟）。超时后，根据 `Correction` 策略决定是中断任务，还是使用“保守默认值”继续执行（需明确告知用户使用了默认值）。

---

## 4. 改进建议

### 4.1 引入状态机（State Machine）模型
不要将 Plan 视为一个静态的 JSON，而应视为一个**运行时状态机**。

**建议数据结构（Java 示例）：**

```java
public class AgentPlanExecution {
    private String traceId;
    private PlanState currentState; // INIT, PLANNING, EXECUTING, VERIFYING, CORRECTING, DONE, FAILED
    private Map<String, StepStatus> stepStatusMap; // stepId -> status
    private Context currentContext;
    private int iterationCount; // 防止死循环
    private static final int MAX_ITERATIONS = 5;

    // 核心方法
    public void executeNextStep() { ... }
    public void handleCheckpointResult(CheckpointResult result) {
        if (!result.isPassed()) {
            this.state = State.CORRECTING;
            this.applyCorrectionStrategy(result.getFailedCheckId());
        }
    }
}
```

### 4.2 细化 DAG 执行引擎逻辑
在 `DAG式 Plan` 章节，补充执行逻辑伪代码：

```python
def execute_dag(plan):
    while has_pending_nodes(plan):
        # 1. 找出所有就绪节点（依赖项全部 SUCCESS）
        ready_nodes = find_ready_nodes(plan)
        
        # 2. 并行执行就绪节点
        futures = []
        for node in ready_nodes:
            futures.append(run_step_async(node))
        
        # 3. 等待所有并行任务完成
        results = await all(futures)
        
        # 4. 更新状态
        for node, result in zip(ready_nodes, results):
            update_node_status(node, result)
            
            # 5. 如果失败，检查是否需要 Fail-Fast（中断整个 DAG）
            if node.critical and result.is_failed():
                raise CriticalExecutionError(node.id)
        
        # 6. 如果所有节点都失败或没有就绪节点且还有待执行节点 -> 死锁，报错
        if no_ready_nodes_and_pending_exists(plan):
            raise DeadlockError("DAG execution stuck")
```

### 4.3 增强 Checkpoint 的自动化验证
将 `Checkpoint` 从“LLM 自我检查”升级为“规则引擎 + LLM 混合检查”。

*   **规则检查（硬性）**：
    *   JSON Schema 验证（格式是否正确）
    *   正则匹配（是否包含敏感词/违禁词）
    *   长度限制
*   **LLM 检查（软性）**：
    *   语义一致性
    *   事实准确性（对比 RAG 检索结果）

**改进后的 Checkpoint JSON 示例：**
```json
{
  "step_id": "rewrite_project_experience",
  "hard_checks": [
    {"type": "min_length", "value": 100},
    {"type": "forbidden_words", "value": ["虚构", "吹牛"]}
  ],
  "soft_checks": [
    {
      "prompt": "请检查以下内容是否忠实于原始事实...",
      "output_schema": {"passed": "boolean", "confidence": "float"}
    }
  ]
}
```

---

## 5. 架构层面反思

### 5.1 当前设计的结构性缺陷
1.  **耦合过紧**：Plan 生成、执行、评估、纠偏都在同一个 LLM 调用或紧密循环中。这导致**可观测性差**（难以知道是哪一步出的错）和**扩展性差**（难以替换某个步骤的工具或模型）。
2.  **缺乏持久化**：Agent 执行通常是长耗时任务。如果进程重启或网络断开，当前内存态的 Plan 丢失。生产环境必须支持 **Plan 状态持久化**（Checkpoint 保存至 DB/Redis），支持断点续传。

### 5.2 更优的系统设计演进路径

**Stage 1: 同步阻塞式（当前文档隐含）**
*   适合：简单 Demo，低并发。
*   缺点：阻塞线程，资源浪费，无断点续传。

**Stage 2: 异步事件驱动式（推荐生产架构）**
*   **核心组件**：
    *   **Plan Generator Service**: 接收请求，生成初始 Plan DAG，存储状态，发布 `PlanCreated` 事件。
    *   **Step Executor Pool**: 监听事件，获取就绪步骤，调用 LLM/Tool，发布 `StepCompleted`/`StepFailed` 事件。
    *   **State Manager**: 监听步骤事件，更新 DAG 状态，判断下一步骤，触发 Checkpoint 验证。
    *   **Verifier Service**: 异步执行 Checkpoint，发布 `VerificationPassed`/`VerificationFailed` 事件。
*   **优势**：
    *   **高并发**：线程池解耦。
    *   **可恢复**：状态存于 Redis/DB，重启后从最后 Checkpoint 恢复。
    *   **可观测**：每个事件都有 Trace ID，链路清晰。

**Stage 3: 多智能体协作（Advanced）**
*   将 Plan 拆分为多个子 Agent（如：Research Agent, Writer Agent, Critic Agent）。
*   主 Agent（Orchestrator）只负责调度 DAG，不直接处理具体逻辑。
*   通过 **Message Queue** 或 **gRPC** 通信，实现物理隔离。

### 5.3 关键建议总结
1.  **立即添加**：最大迭代次数限制、DAG 状态字段、CheckPoint 结构化输出。
2.  **短期优化**：引入异步执行模型，将 Plan 状态持久化。
3.  **长期演进**：构建基于事件驱动的微服务架构，实现 Plan 的可中断、可恢复、可审计。

**结论**：当前文档作为“思维框架”是合格的，但作为“工程实现指南”是**不合格**的。必须补充上述架构级细节，才能用于生产环境。