> 大家好，这里是 Goodnote（好评笔记）。本文详细介绍我们对 Anthropic 技术文章《The New Rules of Context Engineering for Claude 5 Generation Models》的理解。

> Anthropic 梳理了 新模型带来的上下文工程变化。新模型出来之后，他们删掉了 CC 中超过 80% 的系统 prompt，编码能力也没有发现明显的性能下降。背后的工程主线是：**随着模型判断力增强，系统不必继续堆叠大量规则、示例和重复说明；上下文应精简约束，通过清晰的工具接口、渐进式披露、自动记忆和丰富参考资料，让模型在目标与边界内结合实际环境作出判断**。

来源：[The new rules of context engineering for Claude 5 generation models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models)


用户向 Claude 发送消息时，当前 prompt 只占完整上下文的一小部分。系统 prompt、Skills、`CLAUDE.md`、记忆和其他来源会合并在一起，成为模型本轮真正看到的输入。组织这些信息的过程就是上下文工程。

Prompt 面向眼前的一次请求，可以写得很具体。上下文要跨请求使用，无法预知之后会遇到怎样的用户 prompt。给这类通用信息定规则，比写一次性的 prompt 更难。

Claude Code 为 Claude Opus 5、Claude Fable 5 等新一代模型删除了超过 80% 的系统 prompt，编码评估没有发现明显的性能下降。新模型能够自己处理一部分过去必须写成规则的判断。

这些最佳实践已经整合进 `claude doctor`。在 Claude Code 中运行 `/doctor`，就可以检查并简化 Skills 与 `CLAUDE.md`。

## 解除 Claude 的束缚（Unhobbling Claude）

Claude Code 过去会同时通过系统 prompt、`CLAUDE.md` 和 Skills 约束模型。每一处要求单独看都可能合理，合在一起却可能互相冲突。

![组装后的上下文包含冲突指令](<images/Claude-5-context-engineering-01.png>)

图中同时出现了三条要求：
- 系统 prompt 要求“在合适的地方保留文档”，
- Skill 要求“不要添加注释”，
- 用户又说“像旧版本一样工作”。

Claude 会同时看到这些信息，必须先处理其中的重叠和冲突，才能决定怎么做。

Claude 通常能理解用户意图并给出合适结果，但这些冲突会迫使它花更多精力权衡要求。

早期的强约束用于避开最坏情况，误删文件就是其中之一。**新模型的判断力提高后，许多约束可以删掉，让 Claude 结合周围环境和用户意图作决定**。

过去，Claude 主要依赖 `CLAUDE.md` 保存记忆、信息和指导。现在，这些内容可以分开放置。自动记忆会保存与用户和工作相关的信息。遇到特定任务时，Claude 可以读取对应的 Skills。Artifacts 创建的 HTML 也能作为参考资料。各种信息不用再全部塞进 `CLAUDE.md`。

## 过去与现在（Then and now）

一些曾经有效的上下文工程经验，随着模型和工具能力变化，已经不适合继续当成默认规则。

![上下文工程从旧规则转向新规则](<images/Claude-5-context-engineering-02.png>)

这 6 组变化分别涉及规则设计、工具接口、信息加载、工具说明、记忆管理和参考资料。

### 1. 规则设计：从写死要求到结合上下文判断

#### 过去：给 Claude 规则（Then: Give Claude rules）

Claude Code 刚推出时，旧模型还需要依靠强规则避免严重错误。系统 prompt 因此对注释和中间文档作出了接近绝对的限制：

```text
写代码时默认不写注释。不要写多段 docstring 或多行注释块，最多保留一行短注释。
除非用户明确要求，否则不要创建规划、决策或分析文档；应直接使用对话上下文，不依赖中间文件。
```

这些限制无法覆盖所有任务。用户可能有自己的注释偏好，某些复杂代码也确实需要多行注释。

旧模型缺少这些严格限制时，经常会写出不准确的注释。强规则在当时有用，即使它偶尔会限制合理的做法。

#### 现在：让 Claude 使用判断（Now: Let Claude use judgement）

新系统 prompt 改为要求 Claude 参考周围代码：

```text
让生成的代码读起来像周围现有代码，匹配原仓库的注释密度、命名和惯用写法。
```

这条指令要求 Claude 观察附近代码，再决定注释写多少、名称怎么取、使用什么惯用写法。它给出了判断标准，没有提前把所有情况写死。

### 2. 工具设计：从调用示例到清晰接口

#### 过去：给 Claude 示例（Then: Give Claude examples）

早期的工具使用经验把调用示例放在首位。到了新模型上，示例反而容易把 Claude 框在已经展示过的几种用法里。

#### 现在：设计接口（Now: Design interfaces）

现在更需要把工具、脚本和文件的接口设计清楚：Claude 能看到哪些参数，这些参数是否足以覆盖工具的不同用法。

![TodoWrite 从长示例变成简洁接口](<images/Claude-5-context-engineering-03.png>)

图左是 TodoWrite 的旧描述，约 9,100 个字符，包含使用时机和大量示例。图右的新接口保留了三类信息：

1. 工具职责：为当前会话创建和更新任务列表。
2. 状态枚举：`pending`、`in_progress`、`completed`。
3. 行为约束：任何时刻只能有一个任务处于 `in_progress`。

`pending`、`in_progress` 和 `completed` 让 Claude 知道任务会经历哪些状态。“同时只能有一个 `in_progress`”进一步规定了工具的行为。Claude 只看接口就能理解这个工具该怎么用。

### 3. 信息加载：从全部前置到按需读取

#### 过去：把所有信息提前放入上下文（Then: Put it all upfront）

Claude Code 专注编码任务时，系统 prompt 曾经包含详细的代码审查和验证方法。任务用得上时，这些信息很重要；其他时候，它们只会占用上下文。

#### 现在：使用渐进式披露（Now: Use progressive disclosure）

Claude Code 已经能够在合适的时间加载信息。代码验证和代码审查因此被移入独立 Skills，由 Claude 按需调用。

渐进式披露（progressive disclosure）也适用于工具。部分工具采用延迟加载（deferred loading）：Claude 要使用某个工具时，先通过 ToolSearch 找到完整定义。这样一来，即使继续加入 Task 等工具，它们也不会从一开始就占用上下文。

`CLAUDE.md` 和 `SKILL.md` 也可以采用这种方式。不要把所有可能用到的实践都集中在一个文件里，可以组织成一棵文件树，让 Claude 在合适的时间加载对应文件。

### 4. 工具说明：从多处重复到统一写进工具描述

#### 过去：重复提醒（Then: Repeat yourself）

较早的 Claude 模型有时需要反复提醒，也可能更容易遵循上下文末尾的指令。Claude Code 因此会先在系统 prompt 中介绍工具，再在工具描述中重复具体用法。同一份说明会在上下文中出现两次。

#### 现在：使用简洁的工具描述（Now: Simple tool descriptions）

新模型可以直接理解工具描述，系统 prompt 不用再重复工具说明。工具是做什么的、参数怎么用、调用时有哪些限制，都可以直接写在工具描述中。系统 prompt 继续说明 Claude 所处的产品环境，以及它在这个环境中负责什么。

### 5. 记忆管理：从手动写入到自动保存

#### 过去：把记忆写进 `CLAUDE.md`（Then: Memory in CLAUDE.md files）

早期 Claude Code 鼓励用户通过 `#` 快捷键把信息写进 `CLAUDE.md`。用户需要自己判断哪些内容以后还会用到，再把它们保存下来。`CLAUDE.md` 因而同时放着仓库说明，以及希望 Claude 长期记住的信息。

#### 现在：使用自动记忆（Now: Auto-memory）

Claude 现在会**自动保存**与工作和用户相关的记忆，用户无需再通过 `#` 快捷键逐条写入。

自动记忆启用后，`CLAUDE.md` 只需简要说明仓库是做什么的，并记录 Claude 无法直接从文件和代码中看出的特殊约定。**用户偏好和工作中值得保留的信息交给自动记忆**，仓库规则继续放在 `CLAUDE.md`。

### 6. 任务规格：从简单 Markdown 到丰富参考资料

#### 过去：使用简单规格（Then: Simple specs）

计划模式长期依赖 Markdown 计划文件。把计划和规格保存在代码库里，Claude 可以在长项目中随时引用。

#### 现在：使用丰富参考资料（Now: Rich references）

现在可用的参考资料已经不限于 Markdown 计划。Claude 能读取由 Artifacts 功能创建的 HTML，也可以把详细测试套件当作规格，或者参考另一个代码库中的函数并完成移植。

评分标准（rubrics）也是参考资料的一种。它可以把你对质量的判断写清楚，例如优秀的 API 设计应该具备哪些特点。动态工作流（dynamic workflows）还可以启动验证 Agent，按照这些标准检查结果。

## 把规则应用到自己的上下文（Applying this to your context）

组织上下文时，要先想清楚每类信息该放在哪里。

![Claude 上下文的组成层次](<images/Claude-5-context-engineering-04.png>)

图中最上方是用户 prompt，下面还有参考资料、系统 prompt、`CLAUDE.md`、Skills 和记忆。模型会把这些内容放在同一个上下文中使用。

### 系统 Prompt（System Prompt）

系统 prompt 与具体的产品环境紧密相关。它会告诉 Claude 当前是在 Claude Code 还是其他 Agent 产品中工作，以及它在这个环境中负责什么。

普通 Claude Code 用户通常不需要修改产品预设的系统 prompt。如果你在构建自己的 Agent harness，就需要认真设计 Agent 的系统 prompt，把运行环境和职责写清楚。

### `CLAUDE.md`

`CLAUDE.md` 应保持简短。它可以简单说明仓库用途，把大部分篇幅留给代码库里的特殊情况和易错点。

例如，某个仓库要求所有类型只能放在同一个文件里，其他地方都不能定义。这种约定很难只靠查看文件系统得知，适合写进 `CLAUDE.md`。Claude 能直接从文件和仓库看出的内容则不用重复。

如果验证工作有多条专用指令，可以创建一个验证 Skill，再从 `CLAUDE.md` 引用它。

### Skills

Skills 是简短的指南，帮助 Claude 在需要时找到信息。除非内容非常重要，否则不要加入太多硬性限制。

较长的 Skill 应使用渐进式披露，拆成多个文件，让 Claude 只在用到时读取。

Skill 最适合保存个人、团队或产品特有的观点、知识和最佳实践。

### 参考资料（References）

通过 `@` 提及文件，可以把它们加入当前任务的参考资料。Claude 能由此读取任务所需的详细信息，包括规格文件、设计原型和完整代码库。

代码是 Claude 熟悉的语言，表达更清楚，歧义也更少。要描述一个设计，HTML 原型通常比文字说明或截图效果更好。

## 尝试简化（Try simplifying）

系统 prompt、Skills 和 `CLAUDE.md` 里可能还留着为旧模型写的规则。随着模型能力提高，这些内容也需要跟着简化。Claude Code 已经把这类检查整合进 `claude doctor`。在 Claude Code 中运行 `/doctor`，就可以检查并简化 Skills 与 `CLAUDE.md`。关于新模型的 prompt 写法，还可以继续参考 Fable 使用指南（Fable field guide）。
