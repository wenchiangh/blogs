# Knowledge Recovery：关于 AI Coding 中上下文恢复机制的一次设计探索

## 引言

随着 Claude Code、Codex、Cursor 等 AI Coding Agent 在实际研发工作中的广泛使用，一个此前并不显眼的问题开始逐渐暴露出来：当 Agent 在长任务执行过程中发生崩溃、上下文压缩、会话终止、用户主动中断或其他异常情况时，应该如何恢复工作状态，并尽可能减少此前已经投入的时间、Token 与推理成本损失。

目前业界对此问题的讨论，大多集中在 Workflow Recovery（工作流恢复）层面。例如通过 Checkpoint、状态持久化（Persistence）、Durable Execution、Task Resume 等机制，使 Agent 能够在中断后继续执行后续步骤。然而在实际 AI Coding 场景中，仅仅恢复执行流程往往并不能真正恢复 Agent 的工作状态。

因为 AI Coding 的主要成本并不一定来自代码生成本身，而更多来自于代码理解、业务背景理解、文档阅读、依赖分析、架构推理以及设计方案形成等活动。这些活动通常消耗了绝大多数 Token、工具调用次数以及推理资源。

因此，一个更值得研究的问题开始出现：

> Agent 崩溃后，除了恢复任务进度之外，是否应该恢复 Agent 已经获得的知识？

本文记录了一次围绕该问题展开的系统性讨论，并尝试给出一个工程可落地的设计边界。

---

# 问题背景

## Task Recovery 已经在实践中自然出现

在实际使用 Claude Code、Codex 等 Agent 的过程中，许多开发者已经自然演化出了类似的实践。

常见做法包括：

- 将任务拆分为较小粒度
- 使用 Checklist 管理任务状态
- 在阶段结束时更新任务状态
- 中断后从未完成任务继续执行

例如，一个典型任务可能被拆分为：

- 分析登录流程
- 阅读组件文档
- 实现登录逻辑
- 执行测试

当 Agent 崩溃后，系统读取 Checklist，找到未完成任务，并从对应位置继续执行。

这种方式实际上已经构成了一个轻量级的 Task Recovery 机制。

其核心解决的问题是：

> Agent 做到了哪里？

而不是：

> Agent 已经知道什么？

---

# Checklist 的局限性

考虑如下场景：

Agent 在执行某个任务过程中已经：

- 阅读了数十到数百个代码文件
- 查询了组件库文档
- 搜索了业务规范
- 分析了依赖关系
- 建立了代码库结构认知

随后发生崩溃。

Checklist 只能告诉系统：

> 当前任务尚未完成。

却无法回答：

> Agent 为完成该任务已经获得了哪些知识？

于是恢复后的 Agent 往往需要重新执行大量此前已经完成的工作：

- 再次读取代码
- 再次读取文档
- 再次分析业务
- 再次建立理解

从而重复支付此前已经支付过的成本。

这也是本文讨论的真正起点。

---

# 从 Workflow Recovery 到 Knowledge Recovery

讨论过程中逐渐形成了一个重要认识：

Checklist 解决的是 Workflow Recovery。

而 AI Coding 场景中的核心问题实际上更接近于：

**Context Recovery（上下文恢复）**

或者更准确地说：

**Knowledge Recovery（知识恢复）**

两者关注的问题存在本质区别。

| 类型 | 关注的问题 |
|--------|--------|
| Workflow Recovery | 做到哪里了 |
| Knowledge Recovery | 已经知道什么了 |

在 AI Coding 场景下，后者往往比前者更昂贵。

因为代码生成通常只占任务总成本的一小部分，而代码理解与业务理解往往占据了绝大多数成本。

---

# 最初设想：Tool Replay

一个自然的想法是：

既然知识来自 Tool Call，那么是否可以记录所有 Tool Call，并在恢复时进行重放？

例如：

- read_file
- grep
- search
- code search
- documentation lookup

恢复时重新执行这些操作，从而恢复 Agent 状态。

这被称为 Tool Replay。

---

# Tool Replay 的问题

随着讨论深入，一个关键问题逐渐显现：

Tool Call 本身并不重要。

真正重要的是 Tool Call 所产生的知识。

例如，读取一个文件的价值并不在于执行了读取动作，而在于 Agent 因此获得了对某段代码的理解。

因此讨论逐渐从：

**Replay Tool**

转向：

**Replay Knowledge**

---

# Summary 的诱惑

接下来出现了第二种思路。

既然 Tool Result 往往较大，是否可以直接保存：

- Summary
- Inference
- Decision Log

例如：

- LoginForm 依赖 useAuth
- Admin 存在特殊逻辑
- Button 支持 danger 属性

恢复时直接注入这些内容，从而快速恢复 Agent 状态。

从表面上看，这似乎能够以极低 Token 成本恢复大量上下文。

---

# Summary 的根本问题

随着讨论继续，一个更深层的问题暴露出来：

Summary 是谁生成的？

什么时候生成的？

为什么在这个时刻生成？

如何维护？

实际上，Claude Code、Codex、Cursor 等工具虽然提供了 Tool Hooks，但并没有提供：

- Post Reasoning Hook
- Post Planning Hook
- Post Decision Hook

Agent 的推理过程不会天然产生结构化的 Decision Log。

因此：

**Summary 并不是现成存在的数据。**

它必须被额外创造。

而一旦开始创造 Summary，又会引入新的问题：

- 信息压缩损失
- 幻觉风险
- 过期风险
- 维护成本
- 一致性问题

最终讨论得出一个重要结论：

> Summary 本身就是新的风险源。

---

# 一个关键发现：Reasoning State 无法可靠获取

讨论过程中出现了一个重要转折。

最初希望恢复的是 Agent 的推理状态。

但逐渐发现，这种状态实际上很难可靠获取。

Agent 的工作过程更接近如下模型：

```mermaid
flowchart LR

A[Evidence] --> B[Inference]
B --> C[Decision]
```

其中：

- Evidence 可以获取
- Inference 通常不可见
- Decision 并不天然存在

真正驱动后续行为的往往不是原始文件，而是 Agent 已经形成的理解。

但这种理解又无法被稳定捕获。

因此：

> 恢复 Agent 的推理状态，远比恢复 Agent 看到过的事实困难得多。

---

# 重新划定问题边界

至此讨论开始收敛。

一个核心共识逐渐形成：

> 不应该试图恢复 Agent 的脑内状态。

而应该恢复：

- Task State
- Evidence
- Stage Artifacts

即：

```mermaid
flowchart LR

A[Task State] --> D[Recovery Context]
B[Evidence] --> D
C[Stage Artifacts] --> D
```

恢复时，让模型重新推理。

而不是恢复此前的推理过程。

---

# 最终方案

最终方案可以抽象为三层。

## 第一层：Task Recovery

保存：

- Task ID
- Checklist
- Task Status

其目标是回答：

> 做到哪里了？

---

## 第二层：Evidence Recovery

保存：

- Tool Call Journal
- Tool 参数
- Tool 结果来源
- 文件 Hash
- Replay Recipe

其目标是回答：

> 已经看过什么？

恢复时优先验证。

必要时重新获取。

而不是盲目复用旧结果。

---

## 第三层：Stage Artifact Recovery

保存：

- Spec
- Plan
- ADR
- Review
- 已批准设计结论

其目标是回答：

> 已经明确形成了哪些阶段成果？

这些内容本身就是工作流产物，因此可信度远高于自动生成的 Summary。

---

# 为什么不恢复推理状态

讨论最终形成了一个重要原则：

> 尽量持久化事实（Fact），不要持久化理解（Understanding）。

例如：

保存文件路径、文件 Hash、工具调用记录等属于事实。

而：

- 文件核心逻辑是什么
- 为什么采用某种方案
- 应该如何设计

已经属于理解与决策。

从事实到决策：

- 信息价值越来越高
- 可信度越来越低
- 维护成本越来越高
- 错误代价越来越高

---

# 风险分析

## Fact Recovery

优点：

- 易获取
- 易验证
- 易维护
- 易追溯

缺点：

- 恢复后需要重新推理

错误代价：

较低。

通常只是额外消耗时间与 Token。

属于典型的 Fail Safe 设计。

---

## Inference Recovery

优点：

- 恢复速度快
- Token 消耗低

缺点：

- 难获取
- 难验证
- 易过期
- 易污染上下文

错误代价：

极高。

因为错误的理解会持续影响后续决策，并且通常难以发现。

属于典型的 Silent Corruption（静默污染）。

---

# 方案价值

该方案最大的价值并不是实现 Durable Execution。

而是：

> 降低 Agent 在长任务中的知识获取成本。

具体体现在：

- 减少重复阅读代码
- 减少重复阅读文档
- 减少重复搜索
- 减少重复分析
- 降低上下文重建成本
- 降低 Token 消耗
- 提高长任务连续性

同时：

- 不需要修改 Claude Code
- 不需要修改 Codex
- 不需要重写 Agent Runtime
- 不需要改变现有工作流

属于对现有 AI Coding 流程的增强，而非替代。

---

# 当前方案的不足

尽管该方案已经能够解决大部分实际问题，但仍存在明显边界。

目前方案已经回答：

- 如何恢复任务状态
- 如何恢复知识来源
- 如何恢复阶段成果

但仍未回答：

- 如何恢复推理状态
- 如何判断两个推理结果是否等价
- 如何验证知识是否仍然有效
- 如何构建可信的长期 Agent Memory

这些问题已经超出了 Recovery 的范畴，更接近于：

- Agent Memory
- Context Engineering
- Execution Lineage
- Long-Horizon Agent Systems

等更广泛的研究方向。 

---

# 结论

本次讨论最终形成的核心观点可以概括为：

**AI Coding 中最昂贵的成本并不是代码生成，而是知识获取。**

因此：

**Task Recovery 解决的是“做到哪里了”。**

而：

**Knowledge Recovery 解决的是“已经知道什么了”。**

对于当前的 Claude Code、Codex 等 Agent 体系而言，一个高性价比且现实可落地的方案，并不是试图恢复 Agent 的思维状态，而是：

- 恢复任务状态
- 恢复知识来源
- 恢复阶段成果
- 恢复上下文构建能力

然后依赖当前模型重新完成推理过程。

换句话说：

> 不要试图恢复 Agent 曾经是怎么想的。

而应该帮助 Agent 重新获得它曾经看到过的重要事实。

这既符合当前 Agent Runtime 的能力边界，也符合工程系统中“优先恢复事实，而非恢复理解”的设计原则。
