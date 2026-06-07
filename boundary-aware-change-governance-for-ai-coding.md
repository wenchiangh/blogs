# Boundary-Aware Change Governance for AI Coding
> [!zh]
> 基于预期边界与实际影响面的 AI Coding 变更治理

## 问题背景

在实际项目中，AI Coding 的风险并不只是“代码写错”，还有一个更隐蔽的问题：AI 为了完成一个局部需求，可能修改了超出需求必要范围的代码，或者通过共享函数、公共组件、模块依赖、页面复用等方式产生了传递影响。这个问题在传统 PR review 中也存在，但 AI Coding 会放大它，因为模型往往具备较强的生成和重构能力，却未必稳定理解项目中的业务边界、模块边界和风险边界。

因此，质量评估不能只看最终结果是否通过测试、PR 是否合并、线上是否回滚。更关键的是：这次 AI 修改是否被限制在预期范围内；如果实际影响超过了预期范围，是否被及时发现、解释、收敛、升级验证或触发更高等级审核。

这个问题的本质不是要证明 AI 代码绝对安全，而是要让“AI 是否越界修改”从模糊感受变成可观测、可讨论、可迭代的工程问题。

## 核心思路

在 AI Coding 前建立“预期修改范围”，在 Coding 后通过代码结构和依赖关系计算“实际影响范围”，再比较两者之间的偏差。

这个机制可以概括为：

> 在 spec / plan 阶段，由模型基于需求和初步代码理解声明预期变更边界；
> 在 coding 结束后，由工程系统基于真实 diff、代码结构、模块关系和依赖图计算实际影响面；
> 如果实际影响超过预期边界，则触发自动收敛、范围升级、验证升级或人工裁决。

这里的“预期修改范围”不是施工图，也不是绝对限制，而是一个风险假设和比较基线。它不要求模型提前精确预测每个函数或文件，而是要求模型说明：这个需求预计应该影响哪些页面、模块、组件或包；不应该触碰哪些共享能力、平台能力或基础设施；最大预期影响等级是什么。

Coding 后的“实际影响范围”则不依赖模型自述，而由真实代码变更和依赖传播计算得到。二者之间的偏差，就是质量评估中非常重要的信号。

## 为什么这个机制有价值

这个方案的价值不在于替代 PR review，而在于让 PR review 更有效。传统 review 很大程度依靠 reviewer 从 diff 中肉眼发现风险；而边界机制可以先把变更范围、传递影响和验证缺口结构化整理出来，让 reviewer 不再从零开始扫雷，而是审查更高价值的问题：越界是否必要，影响面是否完整，验证是否充分，残余风险是否可接受。

它也让 AI Coding 自动化有了立足点。如果没有预期边界，coding 后系统只能说“这次改了这些文件”；有了预期边界，系统就可以进一步判断“这些改动是否超出了原计划”。这让自动化不再只是结果检查，而可以变成过程中的质量锚点。

这个机制还符合一个更通用的方法论：面对 AI Coding 这种智能、工程和现实混沌交织的问题，不应幻想用一个机制消灭不确定性，而是识别其中可结构化、可程序化的部分，并把它们建设成能力。模型负责开放式理解、计划、生成和解释；程序系统负责代码结构、依赖关系、边界计算、测试映射和结果验证；人负责最终的语义判断和风险裁决。

## 关键概念

### Expected Change Boundary

Expected Change Boundary 指 coding 前声明的预期变更边界。它通常在 plan 阶段产生，用来说明需求预期影响的最大范围。

它可以包含：

- 主要影响页面、模块或业务域
- 允许修改的目录、组件、hook、service、package
- 不应修改的共享能力、平台能力或基础设施
- 预期最大影响等级
- 预计受影响的页面、用户路径或业务入口
- 初步验证计划

示例：

```text
Expected Change Boundary
- Primary scope: merchant-onboarding page/module
- Allowed change areas:
  - src/pages/merchant/onboarding/**
  - src/features/merchant-onboarding/**
- Expected max impact level: Page / Module
- Should not modify:
  - shared form runtime
  - core request/auth
  - router
  - i18n runtime
- Expected affected users/pages:
  - merchant onboarding only
- Required validation:
  - onboarding form validation unit tests
  - merchant onboarding smoke test
````

### Actual Change Surface

Actual Change Surface 指 coding 后从真实 diff 和代码结构中计算出的实际变更范围。它包括直接修改范围和传递影响范围。

直接修改范围回答：

- AI 实际改了哪些文件
- 这些文件属于哪个页面、模块、组件、包或平台能力
- 是否触碰了 shared、core、infra、config、schema、router、auth、tracking、i18n 等高风险区域
- 是否修改了公共导出、公共类型、公共组件 props、公共 API contract

传递影响范围回答：

- 被修改的代码被哪些模块依赖
- 哪些页面、业务域、package、app 或测试受到影响
- 是否出现“直接改动看似局部，但实际影响共享路径”的情况
- 是否有受影响区域没有被验证覆盖

### Boundary Deviation

Boundary Deviation 指 Expected Change Boundary 与 Actual Change Surface 之间的差异。它是本机制的核心评估对象。

如果实际影响没有超过预期边界，说明改动范围基本可控。如果实际影响超过预期边界，则不必立即判定为错误，但必须触发解释和处理：

- 可能是 AI 实现越界，应尝试收敛回局部实现
- 可能是需求本身被低估，应升级任务范围
- 可能是共享能力确实需要修改，应升级验证和审核等级
- 可能是边界模型不完整，应修正 scope map 或依赖图

## 变更范围分级

为了让“越界”可计算，需要先定义代码资产的影响等级。分级不需要一开始非常复杂，但应能区分局部代码、业务模块、页面、共享包、跨业务能力和平台基础设施。

一种可用的分级方式如下：

```text
L0: Local
只影响函数内部或文件内部私有逻辑，不改变导出，不改变调用方可见行为。

L1: Module
影响同一业务模块内的多个文件，例如某个 feature 的 hook、service、component。

L2: Page / Route
影响一个页面或一组明确路由，但不影响共享能力。

L3: Package / Shared Library
影响共享组件库、公共工具、公共 hook、data-access、SDK wrapper。

L4: App / Cross-domain
影响多个业务域、多个页面、全局状态、路由、权限、埋点、i18n、请求层。

L5: Platform / Infra
影响构建、部署、依赖、脚手架、运行时、鉴权底座、监控、发布链路。
```

越界判断可以粗略表达为：

```text
Boundary Risk = max(Direct Change Level, Transitive Impact Level) - Intended Boundary Level
```

如果结果小于等于 0，说明实际影响没有超过预期范围；如果结果为 1，说明轻微越界，需要解释和补验证；如果结果大于等于 2，说明严重越界，需要收敛、升级任务或触发更高等级审核。

## 前端场景中的可行性

在前端项目中，这个方案有较好的可行性，因为前端代码通常天然存在页面、路由、组件、模块、共享库、平台能力等结构。

最基础的边界可以从页面组件和页面模块组件开始建立：

- 页面目录下的私有组件、`hook`、`service` 默认属于该页面范围
- `feature` 目录下的逻辑默认属于业务模块范围
- `shared/components`、`shared/hooks`、`shared/utils` 属于共享能力
- `core/request`、`core/auth`、`router`、`i18n/runtime`、`tracking`、`build config` 属于更高风险的平台能力
- `package` 或 `app` 级别可以作为更粗粒度的边界单位

真正需要团队注入的，不是重新发明函数/模块关系分析，而是声明业务范围和代码范围之间的映射关系。工具可以分析 import graph、project graph、symbol references，但工具不知道某个目录代表哪个业务模块，也不知道某个共享函数在组织中属于多高风险的能力。

因此，第一版可以通过配置文件建立 scope map：

```yaml
scopes:
  merchant-onboarding:
    level: page
    owns:
      - src/pages/merchant/onboarding/**
      - src/features/merchant-onboarding/**
    may_depend_on:
      - shared-ui
      - shared-utils
    forbidden_to_modify:
      - src/core/request/**
      - src/core/auth/**
      - src/router/**
      - src/i18n/runtime/**

  shared-ui:
    level: shared-package
    owns:
      - src/shared/components/**
    impact_level: L3

  platform-auth:
    level: platform
    owns:
      - src/core/auth/**
    impact_level: L5
```

有了 scope map 之后，就可以将 changed files 和 affected files 映射回业务范围，从而判断实际影响是否超过预期范围。

## 自动化与半自动化路径

这个方案不是完全依靠人工 review。它的目标是让机器先完成结构性分析，再让人处理语义性判断。

可以分阶段建设：

### 路径级边界检查

第一阶段不需要复杂 AST，只需要基于路径和配置判断：

- 哪些文件被修改
- 这些文件属于哪个 scope
- 是否触碰 forbidden area
- 是否修改 shared、core、infra、config、schema 等高风险目录
- 是否超过 plan 阶段声明的 allowed areas

这一阶段可以快速发现大量直接越界问题。

### 文件级 / `package` 级依赖影响分析

第二阶段引入 `import graph`、`project graph` 或 `monorepo graph`，计算反向依赖：

- changed files 被哪些文件引用
- changed package 影响哪些 package
- changed project 影响哪些 app 或 project
- 是否有多个页面或模块被传递影响

这一阶段可以发现“直接改动局部，但传递影响更广”的问题。

### 测试范围对齐

第三阶段将影响面和测试计划连接起来：

- Expected Boundary 对应 Expected Validation Scope
- Actual Impact Surface 对应 Required Validation Scope
- 如果 actual impact 超过 expected boundary，测试范围也必须升级
- 如果受影响模块没有测试覆盖，应标记为 validation gap

这让测试不再只围绕需求表面，而是围绕实际影响面展开。

### 契约与公共 API 检查

第四阶段对共享能力建立 contract diff：

- API schema 是否变化
- GraphQL / OpenAPI / protobuf contract 是否变化
- TypeScript public API 是否变化
- 公共组件 props 是否变化
- 埋点字段、i18n key、feature flag、storage key 是否变化

这可以发现很多依赖图不一定能表达的外部可见行为变化。

### 语义级 AI 辅助分析

最后再引入模型做语义解释，而不是让模型独立承担安全判断。模型可以基于 diff、scope report、依赖图和测试结果解释：

- 越界是否必要
- 是否存在更局部的实现方式
- 哪些影响面需要补验证
- 是否应该升级任务范围
- 残余风险是什么

模型适合解释、归纳、生成修正方案；程序系统适合计算、对比和记录事实。

## 与 PR Review 的关系

这个机制并不替代 PR review，而是提升 review 的输入质量。

没有边界机制时，reviewer 面对的是原始 diff，需要自己判断是否越界、影响哪里、该测什么。这个过程高度依赖经验，也容易遗漏结构性风险。

有边界机制后，reviewer 面对的是结构化报告：

```text
Boundary Report

Intended Boundary: L1 Module
Direct Change Level: L1 Module
Transitive Impact Level: L3 Shared Package
Boundary Violation: +2

Reason:
- Changed function validatePhoneNumber is exported from onboarding-utils.
- It is imported by merchant-onboarding, partner-onboarding, and legacy-signup.
- No tests were run for partner-onboarding or legacy-signup.

Required Action:
- Either narrow the change to page-private validation logic,
- or upgrade the task to L3 shared behavior change,
- and add affected caller tests / owner review.
```

此时 reviewer 的职责不再是从零发现问题，而是判断：

- 这个越界是否必要
- 如果必要，需求范围是否应升级
- 影响面是否列全
- 验证范围是否足够
- 残余风险是否可接受

这不是对人审的替代，而是对人审的结构化增强。人仍然负责最终语义判断和风险裁决，但机器先完成边界计算、影响传播和验证缺口整理。

## AI Coding 工作流中的循环

完整循环可以设计为：

```text
1. Requirement Decomposition
   拆解需求点，明确目标、非目标和验收标准。

2. Plan with Expected Boundary
   为需求点或方案点声明预期修改范围、禁止范围、最大影响等级和初步验证范围。

3. Coding
   AI 按 plan 实现，但允许在实现过程中发现必要变更。

4. Actual Surface Extraction
   从 diff、scope map、依赖图、测试映射中生成实际影响面。

5. Boundary Comparison
   比较 Expected Boundary 与 Actual Surface，判断是否越界。

6. Correction / Escalation
   如果越界：
   - 先要求 AI 解释越界原因
   - 如果无必要，要求收敛 diff
   - 如果有必要，升级需求/方案边界
   - 同步升级测试计划、review 等级和风险说明

7. Review
   人审边界偏差、升级理由、验证结果和残余风险，而不是从零阅读整个 diff。

8. Feedback
   将 bad case 回填到 scope map、依赖图、边界规则、测试映射和 planning prompt 中。
```

这个循环的意义在于，它为 AI Coding 自动化增加了一个可校验位置。AI 不只是生成代码，还要在 coding 前形成一个可比较的边界假设；coding 后系统再用真实结果校验这个假设。偏差本身成为质量信号。

## 可评估指标

这套机制可以形成一些明确指标，用来评估它是否真的提高了质量和 review 效率。

### Boundary Fit Rate

实际影响面没有超过预期边界的任务比例。这个指标反映 AI plan 和实际实现之间的一致性。

### Boundary Violation Rate

实际影响超过预期边界的任务比例。这个指标不一定越低越好，早期升高可能说明过去不可见的问题被发现了。

### Unjustified Violation Rate

越界但没有合理解释、没有升级需求、没有补充验证的比例。这个指标越高，说明 AI 实现策略或流程 gate 有问题。

### Auto Containment Rate

越界后，AI 能否自动收敛回预期范围内。这个指标可以反映 AI 的自我修正能力和局部化实现能力。

### Impact Detection Recall

review 或线上后来发现的影响面，边界系统是否曾提前识别出来。这个指标非常关键：

- 如果系统提前识别但没有验证，是流程问题
- 如果系统没有识别，是依赖图、scope map 或动态影响建模问题

### Validation Alignment Rate

最终测试范围是否覆盖实际影响面，而不是只覆盖原始需求点。这个指标反映测试计划是否随实际影响面升级。

### Review Efficiency

reviewer 是否更快定位结构性风险，review comment 是否从风格和细节转向边界、影响面、验证缺口和残余风险。

## 方案边界与限制

这个方案不是银弹，不能证明 AI 一定没有改坏代码。它主要缓解的是结构性越界和传递影响不可见的问题。

它不能完全解决：

- 在允许范围内实现了错误业务逻辑
- 隐含业务规则被破坏
- 动态配置、运行时状态、A/B 实验导致的影响扩散
- 样式、交互、性能体验类的隐性回归
- 测试缺失导致的行为不可验证
- 依赖图无法覆盖的运行时调用关系

因此，它应该与测试、contract check、灰度、监控、人工语义审查一起使用。它的定位不是“自动保证安全”，而是“提高边界风险的可见性和可处理性”。

## 最小可行落地方案

MVP 不需要建设完整平台，可以从一个前端业务仓库开始：

1. 建立四级 scope：
    - page
    - feature/module
    - shared
    - core/platform
2. 用配置文件声明：
    - 每个 scope 拥有哪些目录
    - 哪些目录属于高风险区域
    - 哪些区域对普通需求默认 forbidden
    - 哪些 scope 属于共享或平台能力
3. 在 plan 阶段要求 AI 输出：
    - Expected Change Boundary
    - Expected Max Impact Level
    - Forbidden Areas
    - Expected Validation Scope
4. 在 coding 后由脚本输出：
    - changed files
    - direct scope
    - affected scope
    - direct change level
    - transitive impact level
    - boundary deviation
    - missing validation
5. 如果越界：
    - 先让 AI 尝试收敛 diff
    - 如果越界必要，升级任务范围
    - 同步升级测试和 review 要求
6. 将 review 和线上发现的问题回填：
    - 修正 scope map
    - 修正依赖图规则
    - 修正 plan prompt
    - 补充测试映射和 contract check

这个 MVP 的目标不是解决所有质量问题，而是先把“AI 修改是否超出预期范围”这件事变成可计算、可审查、可复盘的对象。

## 结论

AI Coding 中的修改范围问题，本质上是模型智能、工程结构和现实复杂性共同作用下的边界风险问题。它无法靠模型自觉完全避免，也无法靠人类 reviewer 稳定发现，更不应该被简化为“多跑测试”或“多找 owner review”。

更合理的方案是建立一个 Plan-Boundary-Impact Loop：在 coding 前形成预期边界，在 coding 后计算实际影响面，用二者偏差驱动收敛、验证、升级和人工裁决。

这个方案的价值不在于消灭不确定性，而在于把原本混沌的 AI 修改风险转化成更清晰的问题：AI 是否越界，越界是否必要，影响面是否完整，验证是否覆盖，是否需要升级。这让 AI Coding 的质量治理不再只依赖结果和经验，而有了一个可观测、可度量、可迭代的中间层。
