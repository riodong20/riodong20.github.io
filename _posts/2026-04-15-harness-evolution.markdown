---
layout: post
title:  "主流Harness工作流对比分析"
date:   2026-04-15 21:52:31 +0800
categories: harness
---

# 主流 Harness工作流对比分析

## TL;DR

### 五大框架的核心理念

五大 Harness 工作流框架的出发点各不相同，核心理念差异显著：

- **OpenSpec**（39.1K ⭐）——Spec-Driven Development（SDD，规范驱动开发）的原生实现 [[11]](#ref-openspec)。它从设计之初就将规范定位为 source of truth（事实来源），代码只是规范的自动化表达。其他框架是在工程压力下演化向 SDD 实践模式，而 OpenSpec 本身就是 SDD 的规范定义。

- **Superpowers**（147K ⭐）——可组合技能框架，核心理念是"Process over Prompt"（流程大于提示）[[12]](#ref-superpowers)。它最初不输出 spec，只输出 design 和 plan；但从 v3.2 起正式将 spec 纳入工作流，v5.0 建立 spec 审查循环——这是独立演化出 SDD 核心特征的最典型案例。

- **gStack**（69.6K ⭐）——虚拟工程团队，核心理念是"一人即团队" [[13]](#ref-gstack)。它通过 23 个专家角色和 Autoplan 四维审查（CEO→设计→工程→DX）实现规范驱动的强制约束力。

- **Compound Engineering / CE**（14K ⭐）——复利工程循环，核心理念是"80% 规划审查，20% 执行" [[14]](#ref-ce)。Brainstorm→Plan→Work→Review→Compound 的循环完整实现了 SDD 的核心流程，走"轻仪式感"路线（spec 合并到 plan 而非独立输出）。

- **ECC**（151K ⭐）——Harness 性能优化系统，核心理念是"Skills, instincts, memory, security, and research-first development" [[16]](#ref-ecc)。五层架构（Agents+Skills+Commands+Hooks+Memory）加上 Instinct 持续学习系统——跨会话提取模式、置信度评分、本能→技能聚类，使规范从静态文档进化为持续积累的知识体系。

### 核心论点

**尽管核心理念各异，五家框架在工作流的实践模式上呈现出独立演化，但殊途同归——都在向 SDD 靠拢：**

1. **SDD 实践已可用，唯独功能验证不成熟**：五家在需求澄清、设计持久化、计划原子化、子代理执行+审查、Lint/单测门禁五个维度上，实践均已达到生产可用成熟度。OpenSpec 本身就是 SDD 的规范定义；其他四家并非主动选择 SDD，而是面对相同的工程选择压力，独立演化出与 SDD 方向一致的实践模式。

2. **功能测试是唯一不成熟的环节**：五家都有某种功能验证手段，但无一达到生产可用——从纯提示词约束（Superpowers）到规格合规检查（OpenSpec）到浏览器自动化（gStack/CE/ECC），方案各异且可靠性不足。问题不是各家方案还没统一，而是没有一家的方案可靠到可以信任。

3. **本文归纳的差异化两维度**：优化程度差异（弱可持续，方案公开后竞品快速跟进）vs 有无差异（强可持续，涉及根本性技术难题）。功能测试是当前最大有无差异缺口。

---

## 1. 五大Harness工作流的实践都逐步走向SDD

**主流Harness框架在 Brainstorm → Design → Plan → Execute & Verify 四个阶段的实践模式，独立演化出与 Spec-Driven Development（SDD）高度重合的特征**。这是**实践中走到了相似的道路**——OpenSpec 本身就是 SDD 的规范定义 [[11]](#ref-openspec)，其他四家也并非主动选择 SDD，而是面对相同的工程选择压力——不持久化设计，子代理之间无法传递上下文；不结构化规划，循环无法闭环——独立演化出与 SDD 方向一致的实践模式。这与生物学中的趋同进化（convergent evolution）同理：鲸鱼和鲨鱼并非因为有共同目标才长出流线型身体，而是面对相同的水流阻力，最优解趋同。本文基于五家框架的版本演进对比得出这一结论 [[15]](#ref-sdd-trend)。

### 1.1 SDD：五大Harness工作流进化的方向

SDD 的根本思想是**权力倒置**——传统范式中代码是真相（source of truth），规范只是辅助材料；SDD 范式中**规范是真相，代码只是规范的自动化表达** [[11]](#ref-openspec)。

| 传统范式 | SDD 范式 |
|---------|---------|
| 代码是 source of truth | **规范是 source of truth** |
| 规范服务于代码（辅助材料） | **代码服务于规范**（规范的自动化表达） |
| PRD 是"指导实现"的参考文件 | PRD 是**直接生成实现的源头** |
| 文档写完即可抛弃 | 规范是**持续维护、持续演进**的核心制品 |

五家框架虽然定位不同（技能框架/虚拟团队/复利循环/性能优化系统/轻量规范），但都在独立演化出同一个方向的实践模式：**先对齐，再构建（Agree before you build）**。

### 1.2 五大框架概览

TL;DR 已介绍各框架的核心理念。下表提供结构化数据，作为后续各节分析时的参考锚点。各框架的详细演进历程和版本变更，见[附录 A：五大框架详细 Profile](#附录-a五大框架详细-profile)。

| 框架                                       | Stars | 与 SDD 的关系                     | 功能测试方案                                         |
| ---------------------------------------- | ----- | ----------------------------- | ---------------------------------------------- |
| **OpenSpec** [[11]](#ref-openspec)       | 39.1K | SDD 原生实现（设计初衷即为 SDD）          | 🟡 `/opsx:verify`（规格合规检查）                      |
| **Superpowers** [[12]](#ref-superpowers) | 147K  | 独立演化出 SDD 核心特征（v3.2 起输出 spec） | 🟡 TDD + Spec 审查 + 验证门控（提示词驱动）                 |
| **gStack** [[13]](#ref-gstack)           | 69.6K | 通过审查+门禁实现规范强制约束力              | 🟡 QA 角色 + 浏览器                                 |
| **CE** [[14]](#ref-ce)                   | 14K   | 循环完整实现 SDD 核心流程               | 🟡 Agent Browser                               |
| **ECC** [[16]](#ref-ecc)                 | 151K  | Instinct 使规范从静态文档进化为知识体系      | 🟡 E2E Runner + Playwright + verification-loop |

### 1.3 Brainstorm 阶段：先想清楚再动手

所有框架都认同：**不让 AI 直接动手写代码，先通过结构化对话澄清需求**。

| 框架 | Brainstorm 机制 | 核心输出 | 演进轨迹 |
|------|----------------|---------|---------|
| **OpenSpec** | `/opsx:explore` + `/opsx:propose` | proposal.md（意图层） | 早期只有 propose，v1.0 新增 explore 探索阶段 |
| **Superpowers** | brainstorming 技能（苏格拉底式提问） | **spec 文档**（`docs/superpowers/specs/`） | v3.2 前只输出 design+plan；**v3.2 起正式输出 spec 文档**；v5.0 加入自动化 spec 审查循环 |
| **gStack** | `/office-hours`（6 个核心问题） | 产品方向 spec（4 种范围模式） | 早期无结构化 brainstorm；v0.9 新增 Office Hours |
| **CE** | `/ce:brainstorm`（交互式 Q&A） | 需求计划文档 | v2.28 新增 brainstorm 命令；智能短路（简单需求自动跳过仪式） |

**关键演化趋同点**：
- 五家都在 brainstorm 阶段设置**人类确认门控**——不允许 AI 从模糊想法直接跳到实现
- 五家都采用**结构化对话**而非自由聊天——有明确的提问框架和输出格式
- Superpowers 的演进最说明问题：从只输出 design+plan，到 v3.2 正式输出 spec 文档——spec 从 design 的附属品变为独立的正式制品（详见附录 A.2 的版本演进表）

### 1.4 Design 阶段：设计即规范

所有框架都在将设计从"聊天记录中的隐式决策"提升为**可审查、可追溯的显式制品**。

| 框架 | Design 机制 | 制品形态 | 审查方式 |
|------|-----------|---------|---------|
| **OpenSpec** | `specs/` + `design.md` 分层 | specs/（需求场景）+ design.md（技术方案） | 流动式迭代，无刚性门禁 |
| **Superpowers** | brainstorming Phase 4 → spec reviewer | `specs/YYYY-MM-DD-<topic>-design.md` | v5.0 子代理审查循环（最多5轮）→ v5.0.7 内联自审查（30s） |
| **gStack** | `/plan-design-review`（0-10 评分）+ Autoplan | 4 维度 spec（CEO/设计/工程/DX） | 自动评分 + AI Slop 检测 + 编辑达标 |
| **CE** | `/ce:plan` + `/ce:deepen-plan` | 技术计划文档 | plan-beta 高级设计 + deepen-plan 并行研究 |

**关键演化趋同点**：
- 五家都将设计文档**持久化到文件系统**而非仅存于聊天上下文
- 五家都设置了**设计审查机制**——从 OpenSpec 的轻量迭代到 gStack 的多维度评分
- **gStack 的 Autoplan 最具代表性**：一键串联 CEO→设计→工程→DX 四维审查，将设计审查从"可选项"变为"默认行为"

### 1.5 Plan 阶段：计划即规范

所有框架都在将计划从"AI 的待办列表"提升为**足够精确、可由"热情但缺乏判断力的初级工程师"独立执行的可执行规范**（Superpowers 原文表述）。

| 框架 | Plan 机制 | 粒度标准 | 与 Spec 的关系 |
|------|----------|---------|--------------|
| **OpenSpec** | `tasks.md`（复选框追踪） | 每步可在 AI 工具中执行 | spec → design → tasks 三层分离，plan 是 spec 的展开 |
| **Superpowers** | writing-plans 技能 | 2-5 分钟/任务，含精确文件路径和完整代码 | plan 必须清晰到"初级工程师也能执行"；v4.0 起 spec 合规是代码审查的第一道门 |
| **gStack** | `/plan-eng-review` + Autoplan | 含 ASCII 数据流图、状态机、测试矩阵 | 4 维 spec 自动流入 plan，plan 产出测试计划供 `/qa` 拾取 |
| **CE** | `/ce:plan` + deepen-plan | 蒸馏为 Agent 可执行的技术计划 | brainstorm 产出 → plan 精炼 → work 执行；compound 阶段沉淀学习 |

**关键演化趋同点**：
- 五家都将 plan 设计为**可由独立 Agent 执行的原子任务序列**，而非人类阅读的设计文档
- 五家都要求 plan 中的每个任务**包含验证步骤**（断言、测试命令、预期输出）
- **Superpowers 和 gStack 引入子代理执行**——plan 的消费者不是人类，而是另一个 AI Agent，这是更深层的趋同

### 1.6 Execute & Verify 阶段：执行与验证

Plan 做好后，进入 SDD 实践中最关键的一环——执行与验证。想得再好，不执行、不验证，都是空谈。五家框架在 Execute & Verify 阶段的实践可以归纳为四个递进层面。

#### 1.6.1 任务追踪与执行记录

各家都以不同方式让 Agent 的执行过程可追溯——核心思路一致：**不让 Agent "黑盒式"地写完代码就结束，每一步都要有记录**。

| 框架 | 执行追踪机制 | 特点 |
|------|------------|------|
| **OpenSpec** | `tasks.md` 复选框逐项打勾 | 细粒度追踪，任务完成状态可视化 |
| **Superpowers** | Plan 精确到文件路径和代码片段 | 子代理可独立执行，执行边界清晰 |
| **gStack** | Autoplan 产出测试计划，QA 角色拾取执行 | 角色分工明确，测试与实现分离 |
| **CE** | Brainstorm→Plan→Work→Review→Compound 循环 | 每个循环沉淀经验，执行记录积累为知识资产 |
| **ECC** | Plan 聚焦功能拆解与边界定义 | 开发完成后统一进入验证闭环 |

**关键趋同点**：五家都将执行过程结构化，从 Plan 直接进入可追踪的执行阶段，而非让 Agent 自由发挥。

#### 1.6.2 Code Review —— 不允许写完就停

五家都在执行链路中嵌入了审查节点。严格程度不同，但无一例外要求 Agent 写完代码后必须经过审查，而非"写完即停"。

| 审查严格程度 | 代表框架 | 具体方式 |
|------------|---------|---------|
| 轻量级：流动式迭代审查 | OpenSpec | 无刚性门禁，但持续校验，迭代中发现问题即修复 |
| 中等级：评分 / 嵌入式评审 | gStack, CE | gStack 的四维审查（CEO/设计/工程/DX）自动评分；CE 的 Review 阶段嵌入循环 |
| 严格级：两阶段审查 | Superpowers | 先审 Spec 合规性（实现是否匹配设计规范），再审代码质量 |
| 覆盖级：多 Agent 并行审查 + 静态规则 | ECC | 8 个审查 Agent 并行审查 + AgentShield + 静态规则扫描 |

#### 1.6.3 Lint 和单元测试硬门禁

这是所有框架无一例外的共同底线——**代码跑不过就不允许继续**。

五家都将 Lint 和单测作为执行流程中的硬门禁：OpenSpec 通过 verify 命令串联；Superpowers 和 ECC 在此基础上更进一步，采用经典 TDD 的 RED-GREEN-REFACTOR 循环——先写测试，再写实现；gStack 和 CE 则将单元测试作为事后验证手段嵌入流程。

**关键差异**：经典 TDD 的做法是先写测试用例、再写实现代码，通过红绿灯循环驱动开发。但在当前 AI Agent 工作流中，真正落地这种机制的只有 Superpowers 和 ECC 两家，其余三家更多是将单元测试作为事后验证手段而非驱动手段。

#### 1.6.4 规格一致性验证

实现是否匹配设计规范？这比"代码能否跑通"更深一层。各家实现方式不同，但指向同一个目标：**确保产出的代码不仅仅是"能跑"，而是"符合设计"**。

| 框架              | 规格一致性验证方式                              |
| --------------- | -------------------------------------- |
| **OpenSpec**    | `/opsx:verify` 专门的合规检查命令               |
| **Superpowers** | Spec 合规嵌入审查流程第一道门                      |
| **gStack**      | 四维审查（CEO/设计/工程/DX）交叉验证                 |
| **CE**          | Compound 循环中沉淀验证经验，迭代改进                |
| **ECC**         | Playwright 断言 + verification-loop 双重保障 |

#### 1.6.5 共通设计考量

Execute & Verify 阶段，五家框架的实践呈现出三个明确的趋同方向：

1. **Lint 和单测作为硬门禁**——代码跑不过就不允许继续，技术栈成熟，五家无一例外；
2. **Code Review 节点必设**——不允许 Agent 写完代码就停，必须经过审查，审查的严格程度构成了框架间的差异化；
3. **规格一致性验证**——实现必须匹配设计规范，而不仅仅是"能跑"。这是 SDD 范式"规范是真相"在执行阶段的核心体现。

---

### 1.7 成熟度矩阵：SDD 可用，功能验证不足

上文 1.3-1.6 逐阶段分析了各框架向 SDD 靠拢的实践。本节将这些实践提炼为六个维度，从**成熟度**角度评估。

| 维度 | 成熟度 | 说明 |
|------|--------|------|
| 需求澄清再动手 | ✅ 成熟 | 五家都采用结构化对话+人类确认后才动手，机制稳定可靠 |
| 设计显式持久化 | ✅ 成熟 | 都持久化到文件系统，制品形态虽有差异但都达到可用水平（1.4 节详述） |
| 计划原子化可执行 | ✅ 成熟 | 都将 plan 设计为可由独立 Agent 执行的原子任务序列，实践稳定 |
| 子代理执行+审查 | ✅ 成熟 | 都用子代理执行+审查机制，编排方式虽有差异但都可在生产中使用（1.5 节详述） |
| Lint/单测门禁 | ✅ 成熟 | 通用实践，五家都将 Lint 和单测作为硬门禁，技术栈成熟 |
| **功能测试自动化** | 🔴 **不成熟** | 五家都有某种功能验证手段，但无一达到生产可用——从纯提示词约束（Superpowers）到规格合规检查（OpenSpec）到浏览器自动化（gStack/CE/ECC），方案各异且可靠性不足；详见 1.8 节 |

**矩阵解读**：前五个维度是 SDD 实践的核心环节，五家都已达到可用成熟度——这意味着"用 Harness 做 SDD"已经可行。唯独功能测试自动化成熟度不足——不是缺统一方案，而是没有一家的方案可靠到可以信任。

### 1.8 功能测试：唯一不成熟的环节

前五个维度成熟度足够，唯独功能测试自动化成熟度不足——五家框架在功能测试上的真实状态：

| 框架 | 功能测试方案 | 方案本质 | 核心局限 |
|------|------------|---------|---------|
| OpenSpec | `/opsx:verify` | 规格合规检查（"实现是否匹配 spec"） | 不验证"功能是否真正工作"，只验证"是否按 spec 写了" |
| Superpowers | TDD 技能 + Spec 审查 + 验证门控 | 提示词驱动的行为约束（非自动化工具）：TDD 强制 RED-GREEN-REFACTOR 循环、Spec Compliance Reviewer 独立审查代码、verification-before-completion 五步门控函数 | 对用户项目无功能测试自动化工具（无浏览器自动化、无 E2E 测试框架）；所有验证依赖 Agent 遵守提示词规则而非技术强制执行 |
| gStack | `/qa` + `/browse` | Agent 操作浏览器做 QA | 非确定性——Agent 自己看自己做的产品，Oracle 问题未解决 |
| CE | Agent Browser + `test-browser` | 最激进的浏览器自动化集成 | Agent Browser 可靠性不足，需频繁人工重提示 |
| ECC | `e2e-runner` + Playwright + `verification-loop` | 最完整的验证体系（TDD→验证循环→评估→覆盖率门禁） | Playwright 断言确定性高，但 Agent 编写 Playwright 测试的行为仍概率性；浏览器自动化可靠性同方案 C |

ECC 是覆盖验证环节最全面的框架（8 个审查 Agent + AgentShield + verification-loop + eval-harness + E2E），但它的全面性恰恰强化了论点——**即使投入最多资源在验证上，功能测试自动化仍然是最薄弱的环节**。

> 1.7 节的成熟度矩阵表明，SDD 实践的五个核心环节已达到可用成熟度，唯独功能测试自动化成熟度不足。**问题不是各家方案还没统一，而是没有一家的方案可靠到可以信任**。本文将差异化归纳为两个维度——优化程度差异（弱可持续，方案公开后竞品快速跟进）vs 有无差异（强可持续，涉及根本性技术难题）；功能测试是当前最大的"有无差异"缺口。

---

### 1.9 核心认知：验证反馈是 Harness 的灵魂

#### 1.9.1 根本问题

> "The most common failure pattern was that the agent wrote a solution, re-read its own code, confirmed it looks ok, and stopped." — LangChain [[3]](#ref-langchain)

模型天然存在**首个合理方案偏见**（本文将 LangChain 描述的现象概括为 "First Plausible Solution Bias"）：写完代码 → 读一遍 → 觉得没问题 → 停止。这是开环控制，不是闭环。

**没有验证反馈的 Agent = 没有仪表盘的飞机。**

#### 1.9.2 验证反馈的五大闭环

本文将验证反馈划分为五个递进的闭环 [[1]](#ref-openai) [[2]](#ref-anthropic) [[3]](#ref-langchain)。五个闭环都在回答同一个问题——方案是否成功——但在不同的验证层次上回答，从代码能否运行到方案在真实环境中是否正确，逐层递进：

| 闭环 | 核心问题 | 判定确定性 | 闭环手段 |
|------|----------|-----------|---------|
| **执行闭环** | 代码能跑吗？ | 完全确定 | Lint + 编译通过 = 闭环（编译器自动判定） |
| **治理闭环** | 代码符合编码规范吗？ | 确定 / 半确定 | Rules + Code Review = 闭环（规则自动判定 + 人工审查） |
| **规格闭环** | 代码符合功能规格吗？ | 完全确定 | TDD 的 RED-GREEN-REFACTOR 循环 = 闭环（断言自动判定），规格合规审查作为补充 |
| **功能测试闭环** | 整体功能做对了吗？ | 模糊 | **目前难以闭环**——功能正确性需人类判断，浏览器自动化可靠性未达阈值，提示词约束无法强制执行 |
| **灰度闭环** | 方案在真实环境中正确吗？ | 模糊 | **目前难以闭环**——部署安全性可自动化（渐进发布+回滚），但方案在真实用户环境中的功能正确性继承了功能测试的全部难题

> **判定确定性的含义**：正确性判定是否可以完全机械化执行。"完全确定"指判定标准是二值的（通过/不通过），无需人类介入；"模糊"指判定标准无法机械化，必须引入人类判断。

五者的关系是递进的：**执行闭环是基础，治理闭环是保障，规格闭环验证单元级规格符合性，功能测试闭环验证端到端功能正确性，灰度闭环验证方案在真实环境中的正确性**。其中执行闭环、治理闭环、规格闭环有成熟的工具支撑，判断数据易于获取；功能测试闭环和灰度闭环需要复杂的测试环境和监控系统，判断数据采集难度高，目前难以闭环——这也是后续需要深入讨论的核心问题。

---

## 附录 A：五大框架详细 Profile

> 本附录收录五大框架（OpenSpec / Superpowers / gStack / CE / ECC）的详细演进历程和版本变更。正文 1.2 节提供概览表格，此处为完整参考。

### A.1 OpenSpec（Fission AI，39.1K ⭐）

**定位**：Spec-Driven Development（SDD）轻量框架——"让规范跨平台延续"

**核心分层体系**：

```text
openspec/changes/add-dark-mode/
  ├── proposal.md   — 为什么做，改什么（意图层）
  ├── specs/        — 需求与场景（规格层）
  ├── design.md     — 技术方案（设计层）
  └── tasks.md      — 实现清单（执行层）
```

**2025-2026 关键演进**：

| 时间 | 版本 | 核心改进 |
|------|------|---------|
| 2025 H2 | v0.x → v1.0 | 从静态提示迁移到动态指令系统；线性 proposal→apply→archive 变为灵活操作体系（explore/new/continue/ff/apply/verify/sync/archive）；21+ AI 工具集成；语义 Delta Specs 同步 |
| 2026.02 | v1.2.0 | Profile 配置系统（core/custom）；一步式 Propose 工作流；Pi + Kiro 工具支持；AI 工具自动检测 |

**与 SDD 的关系**：OpenSpec 是 SDD 的原生实现，支持 20+ AI 工具，体现"规范是通用语言"原则 [[11]](#ref-openspec)。

---

### A.2 Superpowers（obra，147K ⭐）

**定位**：可组合技能框架 + 软件开发方法论——"流程大于提示（Process over Prompt）"

**2025-2026 关键演进**（重点关注 spec 输出的演进）：

| 时间 | 版本 | Spec/Design/Plan 核心改进 |
|------|------|--------------------------|
| 2025.10 | v3.2 | 🔑 **首次将 spec/设计文档正式纳入 brainstorming 工作流**——新增 Phase 4: Design Documentation；设计文档写入 `docs/plans/YYYY-MM-DD-<topic>-design.md` |
| 2025.12 | v4.0 | **两阶段代码审查**：第一阶段为 spec 合规审阅（怀疑型审阅者验证实现是否完全匹配 spec），第二阶段才做代码质量审阅 |
| 2026.03 | v5.0 | 🔑 **Spec 目录重构**（Breaking Change）：spec 输出路径改为 `docs/superpowers/specs/`，与 plans 目录分离；**引入自动化 spec 审查系统**（spec-document-reviewer + 最多5轮迭代）；架构指导贯穿技能管道；新增范围评估防止单 spec 过大 |
| 2026.03 | v5.0.5 | **用户审阅门控**：spec 完成与 plan 交接之间添加明确的用户审阅步骤，用户必须在实现计划开始前批准 spec |
| 2026.03 | v5.0.6 | Spec 审阅循环精简（7类→5类检查，最大迭代5→3）；提高阻塞门槛——措辞/格式偏好不再阻塞审批 |
| 2026.03 | v5.0.7 | **内联 Spec Self-Review 替代子代理审查循环**（30s vs 25min，质量相当）；检查项：占位符扫描、内部一致性、范围检查、歧义检查 |

**与 SDD 的关系**：从 v3.2 首次输出 spec，到 v5.0 spec 成为独立制品并有自动化审查循环，再到 v5.0.7 自审优化——Superpowers 经历了最完整的 SDD 演进路径。它将"规范驱动"从人驱动的文档流程，演进为 AI 自主感知和执行的工程行为 [[12]](#ref-superpowers)。

---

### A.3 gStack（Garry Tan / YC CEO，69.6K ⭐）

**定位**：虚拟工程团队（23 个专家角色）——"一人即团队"

**2025-2026 关键演进**：

| 时间 | 版本 | Spec/Design/Plan 核心改进 |
|------|------|--------------------------|
| 2026.03 | v0.6-v0.9 | `/office-hours`（6 个核心问题重塑产品方向）；Boil the Lake 原则 |
| 2026.03 | v0.10-v0.12 | 🔑 **Autoplan**（一键串联 CEO→设计→工程→DX 审查）；设计系统（`$D` 13 个设计命令） |
| 2026.03-04 | v0.13-v0.15 | 设计→代码管道（`/design-html`）；**plan-design-review**（0-10 评分 + AI Slop 检测）；DX 审查维度 |
| 2026.04 | v0.16 | AI Slop 清理；Office Hours 记忆功能 |

**Autoplan 的 SDD 意义**：Autoplan 是 gStack 向 SDD 演进的核心标志——它将四个独立的 spec 审查维度（产品/设计/工程/DX）自动化串联，每个维度的产出自动流入下游维度。这是"规范驱动的自动化编排"的典型案例 [[13]](#ref-gstack)。

```text
/autoplan
  ├── /plan-ceo-review    → 产品方向 spec（4 种范围模式）
  ├── /plan-design-review → 设计 spec（0-10 评分 + AI Slop 检测）
  ├── /plan-eng-review    → 工程 spec（ASCII 数据流图 + 测试矩阵）
  └── /plan-devex-review  → DX spec（TTHW 基准 + 魔法时刻设计）
```

**与 SDD 的关系**：gStack 通过多维度评分和 Autoplan 自动审查，实现规范的强制约束力——规范不仅被写出，还被门禁系统强制执行。

---

### A.4 Compound Engineering（EveryInc，14K ⭐）

**定位**：复利工程循环（Brainstorm → Plan → Work → Review → Compound → Repeat）——"80% 规划审查，20% 执行"

**2025-2026 关键演进**：

| 时间 | 版本 | Spec/Design/Plan 核心改进 |
|------|------|--------------------------|
| 2025.12 | v2.15-v2.19 | `/deepen-plan` 并行研究子代理；agent-native 架构技能 |
| 2026.01 | v2.20-v2.26 | `agent-browser` 技能；`/lfg` 全自主工作流 |
| 2026.02 | v2.28-v2.35 | 🔑 **`/workflows:brainstorm` 命令**（交互式 Plan 问答精炼）；上下文 token 优化（减少79%）；`schema-drift-detector` 代理 |
| 2026.03 | v2.38-v2.51 | 命令重命名 → `ce:*`；`ce:review-beta` 结构化角色管线；`ce:work` Codex 委托模式 |
| 2026.04 | v2.65（最新） | 持续迭代，94 个 release；12 平台支持 |

**与 SDD 的关系**：CE 的 brainstorm → plan → work → review → compound 循环本身就是 SDD 的完整实现。compound 阶段更是独创——将学习成果沉淀为可复用知识，体现 SDD"规范持续演进"的原则 [[14]](#ref-ce)。

> **注意**：CE 是五家中**唯一将 spec 功能合并到 plan 而非独立输出 spec 制品**的框架。其 `/ce:plan` 直接产出技术规格，不再区分 spec 和 plan。这是"轻仪式感"路线——与 OpenSpec 的四层分离形成对比。

---

### A.5 Everything Claude Code / ECC（Affaan Mustafa，151K ⭐）

**定位**：Agent Harness 性能优化系统——"Skills, instincts, memory, security, and research-first development"

Anthropic 黑客松冠军项目，151K GitHub Stars（截至 2026.04）。与前面四家"流程框架"不同，ECC 的核心差异化是**五层架构 + 跨工具适配 + 持续学习系统（Instinct）**。

**2025-2026 关键演进**：

| 时间 | 版本 | Spec/Design/Plan 核心改进 |
|------|------|--------------------------|
| 2025 | v1.1.x-v1.2 | 基础架构建立；Python/Django 支持；Instinct 学习 v2 |
| 2026.02 | v1.3-v1.4 | OpenCode 插件支持；12 agents/24 commands/16 skills；交互式安装向导；多语言规则架构 |
| 2026.02 | v1.6-v1.7 | Codex CLI+App 支持；AgentShield 集成（102 静态规则）；GitHub Marketplace 上线；5 个商业内容技能 |
| 2026.03 | v1.8-v1.9 | 🔑 **正式定位为"Harness 性能系统"**（非配置包）；Hook 可靠性重构；选择性安装架构；6 新 agents 扩展至 10 语言；SQLite 状态存储 |
| 2026.04 | v1.10.0（最新） | Surface 刷新/Operator 工作流；**ECC 2.0 Alpha（Rust 控制平面）**；38 agents/156 skills/72 legacy shims |

**各环节核心机制**（TL;DR 已概述核心理念，此处补充关键实现细节）：

| 环节 | 核心机制 |
|------|---------|
| **Brainstorm** | 研究优先：`search-first` + `deep-research` + `/learn`；Instinct 从历史会话提取可复用模式 |
| **Design** | `architect.md` Agent 做高层决策 → 181 个领域 Skills 提供模式知识 |
| **Plan** | `planner.md` Agent 产出功能蓝图 → `tdd-guide` 测试先行 → `/multi-plan` 多 Agent 分解 |
| **Review** | 8 个审查 Agent + AgentShield（102 静态规则）+ `--opus` 红蓝对抗 |
| **Verify** | `verification-loop`（build/test/lint/typecheck/security）→ `eval-harness`（pass@k）→ 80%+ 覆盖率门禁 |

**与 SDD 的关系**：ECC 的独特之处在于**Instinct 持续学习系统**——跨会话提取模式、置信度评分、本能→技能聚类（`/evolve`），使规范从静态文档进化为持续积累的知识体系 [[16]](#ref-ecc)。

**功能测试方案**：`e2e-runner` + Playwright MCP + `verification-loop`。验证体系最完整（TDD→验证循环→评估→覆盖率门禁），但**同样没有解决浏览器自动化的可靠性问题**——Playwright 断言确定性高，但 Agent 编写测试的行为仍概率性。


---
> **后续讨论**：功能测试自动化是 Harness 工程当前最不成熟的环节——各家方案跨度大、可靠性不足。后续将深入分析功能测试自动化的现状与可行路径。


> 来源：OpenAI [[1]](#ref-openai)、Anthropic [[2]](#ref-anthropic)[[8]](#ref-anthropic2)、LangChain [[3]](#ref-langchain)、Stripe [[4]](#ref-stripe)、OpenSpec [[11]](#ref-openspec)、Superpowers [[12]](#ref-superpowers)、gStack [[13]](#ref-gstack)、Compound Engineering [[14]](#ref-ce)、ECC [[16]](#ref-ecc) 等官方团队与业界公认专家的一线经验


---

## 参考来源

<a id="ref-openai"></a>**[1] OpenAI** — *Harness Engineering: Leveraging Codex in an Agent-First World* (2026.02.11)
- 作者：Ryan Lopopolo, Member of Technical Staff
- 原文：https://openai.com/index/harness-engineering-leveraging-codex-in-an-agent-first-world/

<a id="ref-anthropic"></a>**[2] Anthropic** — *Effective Harnesses for Long-Running Agents* (2025.11.26)
- 原文：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

<a id="ref-langchain"></a>**[3] LangChain** — *Improving Deep Agents with Harness Engineering* (2026.02.17)
- 原文：https://blog.langchain.com/improving-deep-agents-with-harness-engineering/

<a id="ref-stripe"></a>**[4] Stripe** — *Minions: Stripe's One-Shot, End-to-End Coding Agents* (2026.02.09)
- Part 1：https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents
- Part 2（Blueprint 架构详解）：https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2

<a id="ref-anthropic2"></a>**[8] Anthropic** — *Harness Design for Long-Running Application Development* (2026.03.24)
- 作者：Prithvi Rajasekaran（Anthropic Labs）
- 原文：https://www.anthropic.com/engineering/harness-design-long-running-apps

<a id="ref-openspec"></a>**[11] OpenSpec** — *Spec-Driven Development for AI Coding Assistants*
- 原文（GitHub 仓库）：https://github.com/Fission-AI/OpenSpec
- CHANGELOG：https://github.com/Fission-AI/OpenSpec/blob/main/CHANGELOG.md

<a id="ref-superpowers"></a>**[12] Superpowers** — *Agentic Skills Framework & Software Development Methodology*
- 原文（GitHub 仓库）：https://github.com/obra/superpowers
- RELEASE-NOTES：https://github.com/obra/superpowers/blob/main/RELEASE-NOTES.md

<a id="ref-gstack"></a>**[13] gStack** — *Garry Tan's Claude Code Setup: 23 Opinionated Tools*
- 原文（GitHub 仓库）：https://github.com/garrytan/gstack
- CHANGELOG：https://github.com/garrytan/gstack/blob/main/CHANGELOG.md

<a id="ref-ce"></a>**[14] Compound Engineering** — *AI Skills That Make Each Unit of Engineering Work Easier*
- 原文（GitHub 仓库）：https://github.com/EveryInc/compound-engineering-plugin
- CHANGELOG：https://github.com/64labs/compound-engineering-plugin-fork/blob/main/plugins/compound-engineering/CHANGELOG.md

<a id="ref-sdd-trend"></a>**[15] 各家代码仓** 
- 原文（GitHub 仓库）：
  - OpenSpec：https://github.com/Fission-AI/OpenSpec
  - Superpowers：https://github.com/obra/superpowers
  - gStack：https://github.com/garrytan/gstack
  - Compound Engineering：https://github.com/EveryInc/compound-engineering-plugin
  - Everything Claude Code：https://github.com/affaan-m/everything-claude-code

<a id="ref-ecc"></a>**[16] Everything Claude Code (ECC)** — *Agent Harness Performance Optimization System* (2025-2026)
- 原文（GitHub 仓库）：https://github.com/affaan-m/everything-claude-code
- Anthropic 黑客松冠军项目，151K GitHub Stars
- 架构分析：https://byteiota.com/everything-claude-code-production-agent-framework/
