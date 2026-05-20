---
layout: post
title:  "如何用好Claude Code"
date:   2026-03-18 21:52:31 +0800
categories: harness
---

# 先看看你遇到过哪些坑 ?
使用claude code/open code 做实际项目开发的时候，你有没有踩过这些坑
- 薛定谔的完成，agent说解决了问题，实际上还有一堆bug
- 模型不遵守规则，比如明明要求使用uv，刚开始还能遵守，过了一会儿就忘了
- 有skill 就是不用 ? 如果让模型自行判断，只有25%的机会执行
- 压缩上下文时删掉关键信息，比如把你的架构准则从上下文删除，你又得重新要求一遍
- 执行危险操作，比如删库跑路(可能你在允许模型删除临时测试文件的时候进行了授权)
- 注意力不集中，要求明明在上下文中，就是看不到 !

是模型能力不行么 ? 不，是你没有驾驭好他！

>最近， harness engineering 火了，在英语中，**Harness** 最初指的是**马具**（由皮革、带子和金属件组成的成套装备），如果没有 Harness，马的力量不受控制；有了它，马的力量才能被**定向输出**并**安全利用**。**在 LLM 出现之前，码农们已经用了几十年的 **Test Harness。**>从**物理世界的“马具”**，到**电子工程的“线束”**，再到**软件工程的“测试支架”**，至现如今 **Agent 领域** 兴起harness engineering，“千里马” 和 驾驭“千里马” 的“马具” 同样重要！
> 
> 起初使用的是scaffold ，脚手架，但是脚手架临时性太强，EleutherAI 在LLM评估领域使用了harness之后，逐渐被LLM领域接受并扩大到整个工程领域的范围

当下已经有了“千里马”LLM，我们需要有一整套驾驭LLM的机制，才让模型发挥他真正的实力。
要解决开头提到的使用Claude等Coding agent时遇到的一系列问题，我们需要需要做的是:
- 管理好提供给模型的上下文
- 约束、控制好模型的行为
- 为每一个任务，定义清晰的验证标准
下面我们来看看如何用好Claude，避免上述问题

>这些问题的本质是因为模型本身只有一堆权重参数，没有记忆功能，每一次与模型的交互都需要把完整的上下文信息提供给模型，如果提供的上下文信息不准确，模型就很可能放飞自我。

# 如何用好 Claude Code 
我们先看看claude 是如何运行和管理上下文的 .
## Claude 的运行机制和上下文窗口
![claude 工作机制](https://mintcdn.com/claude-code/c5r9_6tjPMzFdDDT/images/agentic-loop.svg?w=1650&fit=max&auto=format&n=c5r9_6tjPMzFdDDT&q=85&s=af82455f790bfd3de42cb85faeca38e7)

当你给 Claude 一个任务时，它会经历三个阶段：收集上下文、采取行动和验证结果，这三个动作组成Claude的agentic loop。

>虽然近期Claude开始支持了1M上下文窗口，我们还是以200K举例

Claude的上下文窗口的固定开销主要包括: 
- 系统指令: ~2K~3K
- 所有启用的 Skill 描述符
- MCP Server 工具定义
- Rules
- LSP 状态:
- CLAUDE.md
- Memory
- 自定义Agent.md 

其中，系统指令(System prompt) 是我们无法控制的，**CLAUDE CODE 的系统提示词** **2749** tokens

Claude.md | Skills | MCP | Rules | 自定义Agent 等占用的上下文空间，则取决于我们在项目中的定义，这部分占用的空间越少，留给用户处理实际任务的上下文窗口才越多，因此，管理好这部分会默认加载到claude上下文窗口的内容，就是管理好上下文窗口的关键。


```
200K 总上下文

**├── 固定开销 (~15-20K)**
**│ ├── 系统指令: ~2K**
**│ ├── 所有启用的 Skill 描述符: ~1-5K**
**│ ├── MCP Server 工具定义: ~10-20K ← **最大隐形杀手**
**│ |── Rules: 规则
**│	|── Agent: 自定义Agent
**| └── LSP 状态: ~2-5K** 
**│**
**├── 半固定 (~5-10K)**
**│ ├── CLAUDE.md: ~2-5K**
**│ └── Memory: ~1-2K**
│
└── 动态可用 (~160-180K)
├── 对话历史
├── 文件内容
└── 工具调用结果
```

![Pasted image 20260318075728.png|934](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/claude_context.png)
其中，系统指令(System prompt) 是我们无法控制的，
**CLAUDE CODE 的系统提示词** **2749** tokens
![CLAUDE System Prompt](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/tiktokenizer.vercel.app.jpeg)


其他默认加载到上下文窗口的内容包括: 
- **CLAUDE.md**, 时刻保持加载
- **Rules**, 没有定义路径约束，则完全加载；定义了路径约束，则按照文件类型定义加载
- **MCP**, 全部加载，占用上下文的一个隐形杀手
- **Skills**, skills说明全部加载，具体skills描述按需加载

这部分占用的空间越少，留给用户处理实际任务的上下文窗口才越多，因此，管理好这部分会默认加载到claude上下文窗口的内容，则是关键

## 管理好默认上下文
### Claude.md 应该写什么
Claude.md应该包含哪些内容，一个形象的比喻，Claude.md是地图而非百科全书，一定要保持简洁，
作为在每个会话中都默认加载的信息，只需要包括广泛适用的东西，对于Claude.md中的每一行内容，都要问自己：_“删除这个会导致 Claude 犯错吗？”_ 如果不会，删除它。膨胀的 CLAUDE.md 文件会导致 Claude 忽略你的实际指令！

业界的最佳实际是，尽量维持在200行以内，只包含如下内容 :
- how to build , how to test , how to run
- 项目结构目录，模块边界
- 代码风格，命名约束
- Never ，绝对杜绝的事情
- 上下文压缩规则，可以压缩的信息，不可以压缩的信息
- 项目里的特殊环境变量! (如果有的话)

可以通过添加强调（例如”IMPORTANT”或”YOU MUST”）来调整指令以改进遵守。

### Skill.md 应该写什么 (渐进式披露，Progressive Disclosure)
Skill 头部的 YAML fontmatter 也是会默认加载到对话的上下文的，用于让模型知道当前有哪些技能，可以在什么场景使用，因此，
- Skill 头部描述信息尽量简洁，明确何时调用；
- Skill.md 最好保持在500行以内(claude官方建议)，较长的描述用好reference；
- 副作用的skill ，禁止自动调用，设置`disable-model-invocation: true`，当成Command来使用，Claude不会读取

Skill定义参考:
```yaml
---
name: release-check
description: Use before cutting a release to verify build, version, and smoke test.
---

## Pre-flight (All must pass)
- [ ] `cargo build --release` passes
- [ ] `cargo clippy -- -D warnings` clean
- [ ] Version bumped in Cargo.toml
- [ ] CHANGELOG updated
- [ ] `kaku doctor` passes on clean env

## Output
Pass / Fail per item. Any Fail must be fixed before release.
```

包含reference的skill目录:
```
my-skill/
├── SKILL.md (required - overview and navigation)
├── reference.md (detailed API docs - loaded when needed)
├── examples.md (usage examples - loaded when needed)
└── scripts/
  └── helper.py (utility script - executed, not loaded)

```

skill.md 中说明什么时候读取reference
```
## Additional resources
- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

常见的Skill.md 内容定义包含，如下几种方式:
- 工具封装（Tool Wrapper）：让你的 Agent 成为任意库的专家
- 生成器（Generator）：从可复用模板生成结构化文档
- 审查器（Reviewer）：按严重程度对照清单评审代码
- 反转（Inversion）：Agent 先采访你，再开始行动
- 流水线（Pipeline）：强制执行带检查点的严格多步骤工作流

后续咱们再深入探讨

### 工具和MCP
#### MCP
MCP的工具描述信息会完整的加载到上下文，因此，尽量只选择你肯定会使用的MCP，
比如代码开发，你大概率会使用 context7 (他有两个工具)
#### CLI工具
CLI 工具是与外部服务交互的最高效的方式，好用的ACI工具让agent事半功倍，比如
- ripgrep替代系统默认的搜索 (win也能用)
- gh(github CLI)
- tmux(多窗口终端，win没有)
- rtk，rust重写的命令，用于避免很多bash命令的返回占用过多上下文窗口
- jq，JSON 版的 sed/awk/grep，配合 rtk 使用

越来越多的应用开始提供应用功能的CLI工具(比如 Obsidian CLI) ，design for AI .
如上，针对这些默认会占用上下文的内容，尽量精简、准确，避免长篇大论，只在项目中放你需要的信息，用不着的都扔了。
### Rules，规则
Rules 定义项目需要遵循的基本规则，每一个对话都会默认加载到上下文，通过[范围限定为特定文件路径](https://code.claude.com/docs/zh-CN/memory#path-specific-rules)来限定编辑某类文件的时候才加载，比如针对.ts 文件的规则，

```markdown
paths:
  - "src/api/**/*.ts"

# API Development Rules
- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```


所有.claude/rules/ 下的文件都可以被递归发现，所以通过目录将指令组织到多个文件夹中，比如前端、后端的Rules放在各自的文件夹，保持模块化并更容易让团队维护。

**遵循以上方法，控制默认加载到CLAUDE对话中的上下文，将更多的context空间留给实际的任务处理**
# 约束、控制 Claude
管理好默认加载到模型的上下文空间，下一步就是控制好CLAUDE的行为
## hook，约束Claude的行为
Hooks是用户定义的 shell 命令，用来强制约束Claude Code在生命周期中的特定阶段的行为。
Hooks提供确定性控制，确保某些操作始终发生，而不依赖LLM是否选择运行它们。

常见的hook时机，
- 工具调用前，防止危险命令执行，使用特别要求的环境
- 工具调用后，结果验证与质量检查，调用日志
- 完成文件编辑，自动格式化，review，跑测试
- commit前，跑测试，敏感信息扫描
- 任务完成后，要求必须完成测试
等等

在项目里，你认为需要的控制点，都应该加上hook，如果AI跑偏，立即纠正他，快速失败
![对比](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/hook_effect.jpg)

适合的hook：
阻断修改受保护文件、Edit 后自动格式化/lint/轻量校验、SessionStart 后注入动态上下文（Git 分支、环境变量）、任务完成后推送通知。

不适合的hook：
需要读大量上下文的复杂语义判断、长时间运行的业务流程、需要多步推理和权衡的决策，这些该在 Skill 或 Subagent 里。

### Hacknoon(@superorange0707) 案例，使用 hook 提升skill 的执行率
在 @superorange0707 发表的博文中提到， 在没有干预的情况下，Claude使用Skill的概率只有**25%**，Skill 只是建议，没有强制使用的要求。为了提升技能的使用率，@superorange0707所在团队的做法是，在提交用户prompt时，添加了评估Skills使用的hook添加要求评估Skills使用的prompt，将Skill 使用率提升到 90%。
 
> **Without intervention, Claude proactively activated project Skills only ~25% of the time.**
##### The “forced eval” hook (core logic)
```javascript
// .claude/hooks/skill-forced-eval.js (core idea, simplified)
const prompt = process.env.CLAUDE_USER_PROMPT ?? "";
​
// Escape hatch: if user invoked a slash command, skip forced eval
const isSlash = /^\/[^\s/]+/.test(prompt.trim());
if (isSlash) process.exit(0);
​
const skills = [
 &nbsp;"crud-development",
 &nbsp;"api-development",
 &nbsp;"database-ops",
 &nbsp;"ui-pc",
 &nbsp;"ui-mobile",
 &nbsp;// ... keep going (we have 26)
];
​
const instructions = [
 &nbsp;"## Mandatory Skill Activation Protocol (MUST FOLLOW)",
 &nbsp;"",
 &nbsp;"### Step 1 — Evaluate",
 &nbsp;"For EACH skill, output: [skill] — Yes/No — Reason",
 &nbsp;"",
 &nbsp;"### Step 2 — Activate",
 &nbsp;"If ANY skill is Yes → call Skill(<name>) immediately.",
 &nbsp;"If ALL are No → state 'No skills needed' and continue.",
 &nbsp;"",
 &nbsp;"### Step 3 — Implement",
 &nbsp;"Only after Step 2 is done, start the actual solution.",
 &nbsp;"",
 &nbsp;"Available skills:",
 &nbsp;...skills.map(s => `- ${s}`)
].join("\n");
​
console.log(instructions);
```

### [rtk](https://github.com/rtk-ai/rtk)，通过hook过滤工具噪声，避免工具执行结果污染上下文
rtk处理和过滤工具执行输出，抽取LLM关心的结果返回给LLM，避免过多的工具输出污染上下文 
```
  Without rtk:                                    With rtk:

  Claude  --git status-->  shell  -->  git         Claude  --git status-->  RTK  -->  git
    ^                                   |            ^                      |          |
    |        ~2,000 tokens (raw)        |            |   ~200 tokens        | filter   |
    +-----------------------------------+            +------- (filtered) ---+----------+
```
## Subagent，执行独立任务，避免上下文交叉污染
Subagent 拥有独立的上下文空间，明确的，适合独立完成的事情，适合分派给独立的Subagent处理，Subagent 完成之后，反馈完成结果，主子agent上下文互不污染。
比如 ， 使用Working tree 进行独立特性开发，解决一个BUG。
![msagent.png](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/msagent.png)

# 验证闭环

不定义验证标准，Claude很可能会跑偏，当 Claude 能够验证自己的工作时，例如运行测试、比较屏幕截图和验证输出，表现会显著提高。没有明确的成功标准，会产生看起来正确但实际上不起作用的东西。你成为唯一的反馈循环，每个错误都需要你的关注。
在Prompt、Claude.md、Skill.md 中都明确的定义验证的方法和要求，比如针对实现函数给出具体的边界、给出验证的参照物、给出明确的成功定义: 

| 策略                | 之前                  | 之后                                                                                                                                  |
| ----------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **提供验证标准**        | _”实现一个验证电子邮件地址的函数"_ | _"编写一个 validateEmail 函数。示例测试用例：[user@example.com](mailto:user@example.com) 为真，invalid 为假，[user@.com](mailto:user@.com) 为假。实现后运行测试”_ |
| **以视觉方式验证 UI 更改** | _”让仪表板看起来更好"_       | _"[粘贴屏幕截图] 实现此设计。对结果进行屏幕截图并与原始设计进行比较。列出差异并修复它们”_                                                                                    |
| **解决根本原因，而不是症状**  | _”构建失败"_            | _"构建失败，出现此错误：[粘贴错误]。修复它并验证构建成功。解决根本原因，不要抑制错误”_                                                                                      |

## 样例：前端组件生成验证要求
```
## 验证要求
1. **组件必须可渲染**
   - 运行：`npm run storybook -- --no-open`
   - 检查组件在Storybook中无错误渲染
   
2. **通过基础属性测试**
   - 验证组件接受`data-testid`属性
   - 验证必要的ARIA属性存在
   
3. **类型安全**
   - 运行：`npm run type-check ComponentName.tsx`
   - 必须通过TypeScript严格模式
```

验证的执行，也可以在hook中定义，比如，每次commit之前，都跑一遍测试用例

# 参考的项目目录解构

总结下来，项目结构应该包含如下信息，

```
Project/
├── CLAUDE.md
├── .claude/
│ ├── rules/
│ │ ├── core.md
│ │ ├── config.md
│ │ └── xx-rules.md
│ ├── skills/
│ │ ├── your-prefered-skill/ 涵盖编码，review，验证，提交，cd/ci 等等
| | └── xx-skills
│ ├── agents/
│ │ ├── architect.md
| | ├── prd-reviewer.md
│ │ ├── code-reviewer.md
| | └── xx-agent.md
│ └── settings.json
|── docs/ 项目文档，
| |── architect/ 系统架构，安全 等文档
| |── design/ 版本设计文档
| └── release-runbook.md
```

# 其他建议

## Do，一些值得使用的命令
- `/plan`，特性开发前，使用 /plan 或者带 /xxbrainstorm 先做计划
- `/insight`，及时更新 Claude.md，保持准确而简洁，使用 /insight
- 使用 `/context` 观察上下文情况，及时 `compact`
- 开启与前文无关的任务时，记得执行 `/clear` 重置context窗口
- `/simplify`，对刚改完的代码做三维检查，代码复用、质量和效率，发现问题直接修掉。特别适合改完一段逻辑后立刻跑一遍，代替手动 review。
- `/btw`，在不打断主任务的前提下快速问一个侧问题，适合"两个命令有什么区别"这类单轮旁路问答，不适合需要读仓库或调用工具的问题。
- 使用 `claude --worktree` ，并行开发

## Do not，应该严格避免的操作
- 同一个会话session不聚焦
- **一次又一次地改正。** Claude 做错了什么，你改正它，它仍然是错的，你再改正。Context 被失败的方法污染。在两次失败的改正后，`/clear` 并编写一个更好的初始提示
-  **过度指定的 CLAUDE.md。** 如果你的 CLAUDE.md 太长，Claude 会忽略一半，因为重要的规则在噪音中丢失。
- **缺少验证**。** Claude 产生一个看起来合理的实现，但不处理边界情况，始终提供验证（测试、脚本、屏幕截图）。如果你不能验证它，不要发布它。
- 要求 Claude “分析”某些东西而不限定范围。Claude 读取数百个文件，填充 context。

**最后，要从把CLAUDE作为一个工具，逐步把他当成你的设计目标的“执行者”，让CLAUDE在你指定的规则下独自完成任务，最终实现躺平，:  ) **

# 参考资料

[Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
[你不知道的 Claude Code：架构、治理与工程实践](https://x.com/HiTw93/status/2032091246588518683)
[# How to Build a Governance Layer for Claude Code With Hooks， Skills， and Agents](https://www.bestblogs.dev/articles?sourceid=e75fe46b)


