> 大家好，这里是 Goodnote（好评笔记）。本文详细介绍我们对 OpenAI 技术文章《Codex as a Platform: Build on the Open Agent Harness》的理解。
![Codex 作为平台封面](<images/Harness-15.-OpenAI-Codex-as-a-Platform-01.png>)
> OpenAI 的 Nicolas Bonamy 和 Derrick Choi 刚刚发布的这篇文章，把 Codex 从独立的编程助手进一步放到“可嵌入业务软件的 Agent runtime”这个位置上。它系统梳理了 `codex exec`、Codex SDK 和 App Server 三种集成入口，并通过物流运营示例 Relay 展示 Codex 如何进入真实产品：**业务应用继续掌握界面、上下文、规则、工具和审批，Codex harness 则承担 Agent loop、会话状态、工具交互与沙箱执行**。这次的重点是平台定位、集成分层和参考架构，Codex CLI、SDK 与 App Server 在此前已经可以被开发者使用。

来源：[Codex as a platform: build on the open agent harness](https://developers.openai.com/blog/codex-as-a-platform)


Codex App、CLI 和 IDE 扩展只是同一底层系统的不同入口。驱动这些体验的是开源 Codex harness，它让模型能够收集上下文、分析任务、使用工具、遵守预先配置的边界、请求人工批准，并将工作持续推进下去。

这种定位改变了 Codex 的集成方向。团队不必把所有工作迁移到通用编程助手中，也可以把 Agent 放进已有的软件：工程工作流、运维看板、安全调查平台、客服控制台或专门服务某个团队的内部应用。

需要注意，**平台定位不等于一次新的开源发布**。Codex 源码、SDK 和 App Server 在此之前已经存在。这里完成的是能力归纳、集成分层和产品架构说明，帮助开发者把已有的开放组件组合成业务系统。

## 可复用的核心是 Agent loop（The reusable part is the agent loop）

一个有能力的 Agent 不能简化为“一段 prompt 加一次模型响应”。真正可运行的 Agent 至少需要完成以下工作：

1. 理解任务目标和当前约束。
2. 在任务推进过程中维护上下文。
3. 检查与问题相关的文件、数据和系统状态。
4. 调用工具，并根据工具结果决定下一步行动。
5. 向用户持续暴露执行进度。
6. 处理工具失败、环境异常和中断。
7. 在高风险操作前请求人工批准。
8. 返回能够进入后续工作流的结果。

包围模型并负责这些工作的执行系统就是 **Agent harness**。模型提供推理能力，harness 决定模型能看到什么、可以做什么、任务状态如何保存，以及行动怎样受到控制。

Harness 并非普通的工程包装层，它会显著影响模型的最终表现。OpenAI 在 ARC-AGI-3 上采用保留推理过程与上下文压缩后，GPT-5.6 Sol 的得分从 13.3% 上升到 38.3%，同时将输出 token 减少约六倍。这个结果说明，改进模型周围的状态管理和上下文机制，有时能够在不更换模型的情况下大幅提高任务成绩与执行效率。

Codex harness 主要负责：

- 管理对话状态；
- 流式返回执行过程；
- 调用和协调工具；
- 实施沙箱与审批策略；
- 让任务跨多轮对话继续运行。

Codex App Server 通过文档化客户端协议暴露这些能力。客户端可以创建 thread、开始一轮任务、接收执行事件，并响应 Codex 发起的审批请求。开发 Agent 产品时，可以直接复用这套 runtime，再决定哪些职责应该留给外围业务应用。

## 开放且可检查、可调整的 Harness（An open harness developers can inspect and adapt）

Harness 开源后，开发者可以检查应用与模型之间的执行层，理解它怎样维护状态、调用工具和限制行动，并根据产品需要调整集成方式。

产品需要控制的内容可以归纳为三部分：

1. **交互界面**

   团队可以保留已有的看板、编辑器、任务队列、地图、业务记录和审批流程，无需把所有工作压缩成通用聊天窗口。

2. **上下文与工具**

   应用负责向 Codex 暴露当前工作需要的系统、资料、数据与操作，也可以提供由应用自身管理的模型上下文协议（Model Context Protocol, MCP）服务。

3. **运行边界**

   宿主应用决定 Agent 在哪里运行、能够访问哪些文件和工具、哪些操作必须审批、执行过程如何被观察，以及结果怎样写回业务系统。

Codex CLI、App Server 和官方 Codex SDK 都作为开源组件发布。**开源层覆盖 harness 与集成接口，模型访问和 OpenAI 托管服务仍然分离。**拿到 Codex 源码不等于拿到模型权重，也不等于 Codex IDE 扩展和 Codex Cloud 全部开源。

## 选择合适的集成层（Choose the right integration layer）

同一种 Codex 能力可以通过不同入口接入，选择取决于任务是否长期运行、是否需要交互，以及应用要控制多少生命周期细节。

| 集成方式 | 适合场景 | 应用获得的能力 |
|---|---|---|
| `codex exec` | 脚本、CI、一次性后台任务 | 运行有明确边界的 Agent 工作并取得结构化结果 |
| Codex SDK | 由程序启动、继续、恢复或流式接收任务 | 用较简单的编程接口控制常见 Codex 工作流 |
| Codex App Server | Agent 是产品自身的一部分 | 维持对话、接收事件、中断任务、暴露工具、处理审批并控制完整交互体验 |

`codex exec` 是 Codex CLI 中面向脚本和 CI 的非交互模式，交互式 `codex` 则面向人在终端中直接使用。因此，这里对比的是 `codex exec`、Codex SDK 和 App Server 三种程序集成入口，不是将整个 Codex CLI 与后两者严格并列。`codex exec` 面向“运行一次任务”，Codex SDK 面向“在应用代码中控制任务”，App Server 面向“围绕 Agent 构建完整产品”。三者底层都在复用 Codex harness，区别主要在控制粒度和产品集成深度。

Codex SDK 已经能够将驱动 CLI 的同一个 Agent 接入内部工具、CI/CD 和应用代码。App Server 继续向下暴露 Codex 的客户端协议，适合需要长期会话、双向事件和审批交互的产品。**这里没有新增一个远程 Codex REST API；应用连接的是 Codex runtime，runtime 再访问模型服务。**

## 围绕工作流构建软件（Build software around the workflow）

最有价值的方向并非复制 Codex App，再换一个产品名称和界面。业务软件应该保持目标用户原本理解工作的方式，让 Agent 进入现有流程。

安全分析师需要看到调查队列、近期告警和受影响服务，并在创建修复工单前完成审批。客服工程师需要结合账户历史、产品日志和内部资料生成回复。产品团队可能希望在任务看板中把 issue 移到就绪状态后，自动启动范围明确的实现流程。

在这些场景中，界面承担着实际的上下文工程职责。它让 Agent 知道用户正在查看什么，向 Agent 提供当前工作需要的工具，也为用户保留检查行动和决定下一步的位置。

![业务应用、Codex App Server 与 MCP 的分工](<images/Harness-15.-OpenAI-Codex-as-a-Platform-02.png>)

图中的架构可以拆成三块：

1. **应用掌握产品上下文、业务规则和用户授权。**当前页面、选中记录、业务状态和允许执行的动作都来自应用。
2. **Codex App Server 提供 Agent loop 和沙箱执行。**它维护会话、协调推理与工具调用，并将执行进度和审批请求返回应用。
3. **应用拥有 MCP 数据与操作。**Codex 可以通过 MCP 查询业务数据或发起修改，但工具定义、权限范围和最终业务记录仍由应用控制。

这套分工把通用 Agent runtime 与具体业务产品分开。开发者可以省去重复建设 Agent loop 的工作，但仍然要认真设计领域上下文、工具接口、业务规则、授权流程、结果呈现、审计与评估。外围应用绝非只有视觉“表皮”，它掌握的是 Agent 能否在真实业务中安全工作的关键部分。

## 示例：Relay（Example: Relay）

Relay 是一个基于 Codex App Server 构建的物流运营示例应用，使用虚构的预置数据。它将 Agent 放在货运控制台旁边，让 Codex 调查延误、比较恢复方案，并在获得批准后执行会改变业务记录的操作。

![Relay 物流运营工作台](<images/Harness-15.-OpenAI-Codex-as-a-Platform-03.png>)

Relay 没有要求用户从空白聊天框开始写 prompt。用户先在业务界面中选中一笔货运，再点击“比较恢复方案”等操作。应用知道用户当前关注哪笔运输任务，因此可以自动把货运状态和相关上下文交给 Codex。

完整流程可以拆成以下步骤：

1. 用户从异常运输队列中选中一笔延误货运。
2. 用户触发“比较恢复方案”等业务动作。
3. Relay 将选中货运的上下文传给 Codex。
4. Codex 通过 Relay 管理的 MCP 工具查询最新运营数据。
5. Agent 比较可行方案并解释各自影响。
6. 如果行动会改变真实记录，例如重新订舱，Codex 向应用发起审批请求。
7. 用户在 Relay 界面中批准操作。
8. Codex 调用 MCP 工具修改底层记录。
9. Relay 刷新业务视图，显示操作后的最新状态。

这条链路中，Codex harness 管理 Agent loop、对话状态、流式活动和工具交互；Relay 管理运输看板、业务记录、MCP 工具和审批控件。**调查与推理由 Codex 承担，业务事实和最终控制权留在业务系统。**

Relay 的价值不在物流业务本身，而在可复用的组合方式。相同模式可以用于事件响应、账户运营、研究工作流，以及任何需要 Agent 在现有产品中调查信息、提出建议并经过授权执行操作的场景。

## 开发者正在构建什么（What developers are building）

类似架构已经进入公开产品和真实业务：

1. GitHub 和 JetBrains 将 Codex 接入已有 IDE 工作流，让开发者在熟悉的开发环境中使用 Agent。
2. Cisco 在 Cisco Cloud Control 的 App Builder 中使用 Codex SDK。
3. Thrive Holdings 和 Crete 将 Codex 用于包含税务专业人员反馈的报税流程。试点处理了 7,000 份申报材料，准备时间缩短约三分之一。

这些案例覆盖的范围已经超出软件工程。客服团队可以用 Agent 调查客户问题，运营团队可以协调业务流程，安全团队可以分诊事件，销售团队可以研究账户，市场团队可以生成和推进营销工作。应用提供业务上下文、工具与审批，Codex 在底层驱动 Agent loop。

## 构建不止于显而易见的产品（Build beyond the obvious）

大量工作依赖看板、时间线、地图、资料或系统记录。它们承担着信息组织和决策功能，让人能够理解当前状态、判断风险并保持控制。因此，Agent 产品的方向不应是用一个万能聊天框替换这些界面。

更合适的方式是增强现有软件：让 Agent 理解用户正在处理的工作，调查相关上下文，提出下一步行动，并在得到批准后执行操作。界面继续承担业务表达、风险沟通与控制职责，Agent 则负责在背后推进复杂任务。

Codex App、CLI 和 IDE 扩展已经展示了 harness 能做到什么。开发者可以从开源 Codex 仓库开始，根据产品需求选择 `codex exec`、Codex SDK 或 App Server。真正的变化不在于再次开源已有代码，而在于**Codex 被明确定位为可嵌入业务软件的通用 Agent runtime**，并通过一套完整参考架构展示它怎样进入真实工作流。

# 讨论补充：这次到底新增了什么

围绕“Codex 本来就开源，这次究竟新在哪里”，可以得到以下结论。

1. **这不是 Codex 首次开源**。Codex CLI 和底层 harness 的源代码此前已经可以在 GitHub 查看、修改和编译。把这次发布概括成“OpenAI 全面开源 Codex Harness”，会让人误以为原本闭源的核心代码刚刚开放。

2. **Codex SDK 也不是这次才出现**。OpenAI 已在 2025 年 10 月发布 TypeScript 版 Codex SDK。开发者此前就能在自己的程序中启动、继续和恢复 Codex 任务，也能把它接入内部工具与 CI/CD 工作流。

3. **此前已经可以把 Codex 嵌入软件**。有源码时，开发者可以自行改造和封装 Codex；有 SDK 后，也可以通过正式的编程接口调用驱动 Codex CLI 的同一个 Agent。App Server 同样不是一个突然出现的新概念。

4. **这次没有新增一个远程 Codex API**。Codex SDK 和 App Server 控制的是 Codex Agent runtime，runtime 仍然需要访问模型服务。开源的是 harness 与集成层，模型权重、Codex Cloud 和其他托管服务并未因此开源。

5. **真正向前推进的是平台定位**。OpenAI 将 `codex exec`、Codex SDK 和 App Server 统一整理成三种集成层级，并明确鼓励开发者把 Codex 当作通用 Agent 后端。开发者可以少写一套通用 Agent loop，把精力放在领域上下文、MCP 工具、业务规则、权限、审批和界面上。

6. **Relay 提供了完整的业务应用参考架构**。用户无需从空白聊天框开始写 prompt，而是在物流面板中选择一笔货运并触发业务动作。应用自动提供上下文，Codex 调查数据、比较方案并发起审批，MCP 工具在批准后修改记录，业务界面随之刷新。

7. **业务应用远不止一层“表皮”**。Codex 可以承担 Agent loop、状态维护、工具交互和沙箱执行；业务应用仍需掌握数据、工具定义、权限边界、业务规则、人工审批、审计、评估与结果展示。这些部分决定 Agent 能否安全进入真实工作。

8. **已有 App Server 用户几乎没有获得新的底层能力**。如果开发者此前已经直接使用 App Server，这次更接近一篇 Codex 平台架构宣言、集成指南与案例说明，主要价值是把已有组件的组合方式讲清楚。

一句话概括：**代码和 SDK 早已存在，这次主要是 OpenAI 正式把 Codex 定位为可嵌入业务软件的 Agent 平台，并通过 Relay 展示一套完整落地方式。**
