> 大家好，这里是 Goodnote（好评笔记）。本文详细介绍我们对 Lilian Weng 技术长文《Harness Engineering for Self-Improvement》的理解。

> Lilian Weng 在 2026 年 7 月从递归自我改进出发，把关注点从模型直接修改自身参数扩展到模型周围的执行系统。工具、上下文、持久状态、评估器、权限和工作流都可以成为自动改进对象：**更现实的自我改进路径可能先发生在 Harness 层，Agent 通过提出、评估和验证修改来优化自己的运行机制，同时必须把可靠评估、权限边界与人工审查留在自我修改 loop 之外**。

来源：[Harness Engineering for Self-Improvement](https://lilianweng.github.io/posts/2026-07-04-harness/)


递归自我改进（Recursive Self-Improvement, RSI）的起点来自 I. J. Good 对“超智能机器”的设想：机器如果能超过人类所有智力活动，就可能设计出更好的机器，从而继续提升自己。Yudkowsky 后来把 RSI 明确为一个反馈 loop：AI 使用当前智能去改进未来产生智能的认知机制。

放到现代 AI 系统里，反馈 loop 有两种含义。它可以是模型直接改写自己的权重，也可以是更宽泛的系统改进：模型改进训练 pipeline 和部署系统，部署系统再支撑一个更强的后继模型。“部署系统”不能当外围小配件看。它是基础模型和真实世界上下文之间的关键层。

Harness 就是这个中间层。它围绕基础模型，编排执行过程，并决定模型如何思考、规划、调用工具、采取行动、感知上下文、管理状态、保存产物和评估结果。Claude Code 和 Codex 这类 Coding Agent 产品的成功，正说明 harness 已经是 AI 部署系统的重要组成部分。

# Harness 设计模式（Harness Design Patterns）

早期 Agent 框架常用 `agent = LLM + memory + tools + planning + action` 概括基本组成，但这个表达还不够工程化。Harness engineering 额外加入工作流设计、评估、权限控制和持久状态管理。它关心模型能不能回答，也关心模型在真实系统里如何观察、行动、记忆、检查自己并改进。

Harness 设计应该有意保持简单和通用。越简单，越容易泛化，也越容易借用既有软件工程实践。它和操作系统（Operating System, OS）有相似性：OS 把复杂逻辑封装起来，同时向用户和程序暴露简单接口；harness 也应该把复杂的工具、上下文、状态和评估机制封装起来，让模型通过稳定接口行动。

随着行业成熟，配置、工具接口和协议也会趋于标准化。MCP、Agent Skills、统一工具 schema、工作区状态文件，都可以理解为这种标准化趋势的一部分。

## 模式 1：工作流自动化（Pattern 1: Workflow Automation）

工作流自动化的关键，是定义一个模型可以在其中操作、测试和迭代的工作流。Karpathy 的 autoresearch 仓库是一个很好的例子：系统会让模型围绕目标持续计划、执行、观察/测试、改进，再继续执行，避免一次性写完结果。

![简化版 Codex Agent loop](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-01.png>)

图中有 4 个核心节点：用户输入、模型推理、Agent 响应、工具调用。用户输入进入模型推理后，模型可以直接生成 Agent 响应，也可以发起工具调用。工具返回结果会回到模型推理节点，影响下一轮生成。

工具调用把一次性回答变成了 loop 系统，这才是图里最有用的地方。模型可以根据工具反馈改变下一步行动，形成一个持续运行的 Agent 运行时。工作流还可能主动向用户请求澄清，例如任务规格不明确、执行偏好不清楚时，Agent 不应该盲目继续，最好把不确定性显式暴露出来。

工作流图也强调轨迹分析。模型执行任务后，还要分析自己的轨迹和失败案例，再基于这些反馈迭代。静态 prompt 做不到这一点，因为静态 prompt 不会自动保留行动轨迹、错误日志和测试反馈。

## 模式 2：文件系统作为持久记忆（Pattern 2: File System as Persistent Memory）

长周期 Agent 系统反复出现一个模式：状态和产物非常丰富，但控制接口要尽量简单。Harness 不应该把整个工作流和所有日志一直塞进上下文，而应该把持久状态写入文件。

长周期运行会产生很多超出上下文窗口的信息，例如实验日志、代码 diff、论文摘要、错误轨迹、过去运行的轨迹。它们如果都被追加进上下文，很快会挤掉真正重要的信息；如果存成文件，模型可以按需搜索、读取、压缩和引用。

文件系统作为记忆的优势在于：LLM 已经越来越擅长读写文件和使用 `bash`。读、写、编辑文件是基础工程能力，所以用文件管理持久状态能自然受益于核心模型能力提升。它不需要复杂 memory architecture 才能工作，反而因为简单而可靠。

**长期记忆不一定要从复杂数据库开始**。很多 Agent harness 先用文件系统就够了：能恢复、能审计，也能搜索。

## 模式 3：子 Agent 和后台任务（Pattern 3: Sub-agent and Backend Jobs）

Harness 可以启动多个子 Agent 并行执行，也可以监控后台任务。主 Agent 需要搜索多个假设、同时跑实验、或者把隔离子任务交出去时，子 Agent 很适合。

主 Agent 还需要一个小型进程管理器，负责 launch jobs、inspect logs、cancel failed runs、merge results。否则并行任务很容易变成一堆不可追踪的聊天分支。

并行性要显式、可检查。如果子 Agent 输出只存在临时聊天上下文里，结果很快会过期、隐藏或丢失。如果输出被存成文件、日志和状态记录，主 Agent 就可以在中断后恢复，也可以回看自己的执行历史。

这也是 harness 和普通“多开几个聊天窗口”的区别。真正的 sub-agent harness 需要任务隔离、状态记录、日志检查、失败取消和结果合并机制。

## 案例：Coding Agent Harness（Case study: Coding Agent Harness）

主流 Coding Agent 的核心接口已经逐渐稳定。Claude Code、Codex、OpenCode 和 Cursor 风格 Agent 都在使用相似的 loop 结构进行开发：观察仓库、计划、搜索和读取文件、编辑或写补丁、运行测试、检查错误、重复迭代。

![Coding Agent harness loop](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-02.png>)

图中的顺序是：观察仓库 → 制定计划 → 搜索/读取文件 → 编辑/写补丁 → 运行测试。如果测试通过，就进入完成状态；如果测试失败，就检查错误，然后回到制定计划或搜索/读取文件，继续重复。

这个 loop 很像人类开发者的工作方式。人类通常也要先理解仓库，计划改动，查文件，编辑，跑测试，再根据错误定位问题，很少看一眼仓库就写完所有代码。Harness 把这种工程节奏变成 Agent 的默认执行路径。

工具表如下：

| 工具组 | 工具定义 |
|---|---|
| 文件系统 | 文件发现：`glob`、`grep`、`ls`；文件读取：`read`、`read_many`；文件修改：`write`、`edit`、`multi_edit`、`apply_patch` |
| Shell 执行 | 运行命令：`bash`、`PowerShell` |
| 输入输出 | `lsp`，以及 `git_status`、`git_diff`、`git_commit` 等 git 工具 |
| 外部上下文 | MCP tools、Skills |
| Web 搜索 | `web_search`、`web_fetch`、browser tools |
| 产物 | 读取文档和图片，生成 HTML、图片等 |
| 后台进程 | `CronCreate`、`CronDelete`、`CronList` |
| Agent 委派 | `spawn_agent`、`resume_agent`、`wait_agent`、`list_agents`、`close_agent`、`interrupt_agent` 等 |

这张表把工具按工程角色分组，没有平铺成一堆 API。文件系统负责定位和修改代码；shell 负责执行；LSP 和 git 负责开发上下文；MCP 和 Skills 扩展外部知识和专门流程；后台进程支持异步任务；Agent 委派支持并行探索。

Coding Agent harness 的本质已经超出“模型会写代码”。模型被放进了一个接近 IDE + shell + git + 测试系统 + 任务管理器的环境中，**工具和 loop 共同决定 Coding Agent 的实际能力上限**。

## Harness 层与核心智能（Harness Layer vs Core Intelligence?）

RSI 未来有多少依赖 harness engineering 很难预测，但近期路径不太可能让模型直接改写自己的权重。更现实的路径有 2 点：

1. Harness engineering 会走向 meta-methodology，也就是改进“获得更好答案的机器”，而不只是改进单个答案。Harness 系统本身会成为优化目标，减少启发式规则，增加更通用的机制。
2. 成熟 harness 会支撑自动化研究和模型自我改进 loop；更聪明的模型又能防止 harness 过度工程化，让系统保持可持续。

很多 harness 改进未来可能被内化进核心模型行为，但外部上下文和工具接口仍然会保留。Prompt engineering 已经出现过类似变化：手写 prompt 技巧随着 instruction tuning 和模型推理能力增强而变得没那么中心，但目标、约束、上下文和评估仍然必须被明确指定。

这里可以直接下一个判断：**harness 不会替代模型智能，但它会决定模型智能如何落地**。模型越强，越能利用好 harness；harness 越成熟，越能放大模型能力。

# Harness 优化（Harness Optimization）

Harness 系统中的优化**大致可以分成几个阶段**：指令 prompt → 结构化上下文 → 工作流 → harness 代码 → 优化器代码。模型越智能，优化目标就越复杂，方法也越通用。

这条递进关系值得单独记住。早期只是调 prompt；后来开始管理结构化上下文；再后来搜索工作流；再往后可以让 Agent 修改 harness 代码；最终甚至可以优化优化器代码，也就是优化“如何优化 harness”的系统。

## 上下文工程（Context Engineering）

长周期任务里，把所有工具返回和模型生成直接追加进上下文会迅速失控。上下文管理要做的是构造更结构化、更简洁的 LLM 上下文，同时管理持久状态。长上下文研究会继续进步，但 long-context intelligence 和 context engineering 在现实系统中经常交织在一起：模型窗口变长不等于信息管理问题消失。

智能体上下文工程（Agentic Context Engineering, ACE）把上下文看成一个会演化的操作手册。它维护一个由要点组成的上下文操作手册，每个条目有标识符和描述，避免把上下文一路堆成长 prompt。

ACE 有 3 个组件：

1. 生成器（Generator）：参考要点生成任务轨迹。
2. 反思器（Reflector）：从成功和失败轨迹中提炼洞察，得到有价值的见解。
3. 整理器（Curator）：用增量、条目化方式更新结构化上下文。

![智能体上下文工程框架](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-03.png>)

图中左侧是查询（Query）和上下文操作手册（Context Playbook），它们一起进入大语言模型生成器，生成轨迹。轨迹进入大语言模型反思器，反思器通过迭代优化提炼洞察。洞察再进入大语言模型整理器，生成增量上下文条目（Delta Context Items），更新回上下文操作手册。

ACE 的整理器不会重写整段 prompt 大块文本。它输出 `(identifier, description)` 形式的结构化条目，再用确定性逻辑合并进上下文日志。这样可以避免反复重写带来的上下文坍缩（context collapse）和简短偏置（brevity bias）：
- 上下文坍缩是多轮压缩后信息越变越扁；
- 简短偏置是系统为了简短而不断丢掉细节。

ACE 能从运行轨迹中学习洞察，向自管理记忆迈进一步。但它的**更新规则和整体工作流仍然是手工设计的**。为了让 loop 更像自我改进，元上下文工程（Meta Context Engineering, MCE）把管理机制和具体内容分开：
- 元层优化“如何管理上下文”的 Skill；
- 基础层优化具体任务里的上下文。

MCE 中，一个 Skill $s \in \mathcal{S}$ 定义的上下文函数由静态组件和动态算子两部分组成：

$$c_s=(\rho_s,F_s)$$

其中：

1. $s$ 表示某个具体 Skill，$\mathcal{S}$ 表示所有候选 Skill 的集合。
2. $c_s$ 表示由这个 Skill 定义的上下文函数，也就是“给定任务输入时，怎样构造上下文”。
3. $\rho_s$ 表示静态组件，负责保存相对稳定的知识、prompt、代码库和参考材料。
4. $F_s$ 表示动态算子，负责在执行时搜索、筛选、过滤、格式化和组合信息。

Skill 在这里可以理解为**一套“静态知识 + 动态操作”的上下文生成机制**，不能只看成一段静态说明。

这个上下文函数会根据当前输入 $x$ 动态生成上下文 $c$：

$$c = F_s(x;\rho_s)$$

其中：

1. $x$ 是当前任务输入，例如用户问题、任务描述、代码仓库状态或实验目标。
2. $F_s$ 是 Skill 提供的上下文构造方法。
3. $\rho_s$ 是 $F_s$ 可以调用的静态资源。
4. $c$ 是最终交给模型使用的上下文。

其中：

1. $\rho_s = \{\rho_1,\dots,\rho_m\}$ 是静态组件，例如 prompts、knowledge bases、code libraries。
2. $F_s = \{F_1,\dots,F_k\}$ 是动态算子，例如 search、selection、filtering、formatting。

$m$ 和 $k$ 分别表示静态组件数量和动态算子数量。静态组件解决“系统长期知道什么”，动态算子解决“系统当下怎么取、怎么选、怎么组织”。MCE 比普通 prompt 优化更像 harness 优化，因为它还会优化取上下文的流程，不只改文字内容。

MCE 的双层优化分为内层和外层：内层优化给定 Skill 下的上下文函数，外层选择验证集表现最好的 Skill：

$$ \text{Inner: }c_s^*=\arg\max_{c_s}J_\text{train}(c_s;s)\quad \text{Outer: }s^*=\arg\max_{s\in\mathcal{S}}J_\text{val}(c_s^*) $$

其中：

1. 内层目标：在固定 Skill $s$ 的前提下，寻找训练集上表现最好的上下文函数 $c_s^*$。
2. $J_\text{train}(c_s;s)$ 表示这个上下文函数在训练任务上的表现分数。
3. $\arg\max_{c_s}$ 表示从所有可能的上下文函数中，选出让训练分数最高的那个。
4. 外层目标：在所有候选 Skill $\mathcal{S}$ 中，寻找验证集表现最好的 Skill $s^*$。
5. $J_\text{val}(c_s^*)$ 表示内层已经找到的最佳上下文函数，在验证集上的表现。

内层回答“给定一个 Skill，怎样构造上下文更好”；外层回答“哪一种 Skill 本身更值得保留”。验证集负责筛选真正能泛化的 Skill，避免系统只在训练任务上调上下文。

生成第 $k$ 轮 Skill 之前，Skill database 使用 $\mathcal{H}_{k-1}$ 记录已有的 Skill、上下文函数及其训练集和验证集指标：

$$\mathcal{H}_{k-1} = \{(s_i,c_i,J_i^\text{train}, J_i^\text{val})\}_{i=1}^{k-1}$$

其中：

1. $\mathcal{H}_{k-1}$ 表示前 $k-1$ 轮积累下来的历史记录。
2. $s_i$ 是第 $i$ 个历史 Skill。
3. $c_i$ 是这个 Skill 对应的上下文函数。
4. $J_i^\text{train}$ 是训练集表现。
5. $J_i^\text{val}$ 是验证集表现。

这相当于一个“进化档案”：系统不只保存最后的好结果，也保存哪些 Skill、哪些上下文函数、哪些分数曾经出现过。后续生成新 Skill 时，可以从这批历史经验里做组合和筛选。

第 $k$ 轮中，元层 Agent 会对历史 Skill 做 Agent 式交叉，根据任务 $\tau$ 和历史数据库 $\mathcal{H}_{k-1}$ 生成新 Skill $s_k$：

$$s_k=\text{crossover}(\tau,\mathcal{H}_{k-1})$$

其中：

1. $\tau$ 是当前任务。
2. $\mathcal{H}_{k-1}$ 是历史 Skill 数据库。
3. $\text{crossover}$ 表示把历史上有效的 Skill 片段、策略或结构重新组合。
4. $s_k$ 是第 $k$ 轮生成的新 Skill。

crossover 更像由 Agent 阅读历史经验后进行的重组，不限于遗传算法里的机械交叉。哪些策略有效，哪些失败模式需要规避，哪些上下文组织方式值得继承，都可以进入新 Skill。

基础层上下文工程器随后执行新 Skill，把当前任务、上一轮最佳上下文函数和本轮运行反馈结合起来，更新第 $k$ 轮的上下文函数：

$$c_k=\text{engineer}(\tau,s_k;c_{k-1}^*,\mathcal{R}_k)$$

其中：

1. $c_k$ 是第 $k$ 轮得到的新上下文函数。
2. $\tau$ 仍然是任务目标。
3. $s_k$ 是刚刚由元层生成的新 Skill。
4. $c_{k-1}^*$ 是上一轮已经找到的最佳上下文函数，可以作为起点或参考。
5. $\mathcal{R}_k$ 是第 $k$ 轮运行产生的反馈、轨迹和结果。
6. $\text{engineer}$ 表示基础层上下文工程过程，它会把任务、Skill、旧上下文函数和新运行反馈结合起来，得到新的上下文构造方式。

MCE 的整体逻辑因此变成一个双层 loop：元层负责演化 Skill，基础层负责让 Skill 变成可执行的上下文函数。它优化的是一套能持续生成上下文的机制，写出更好的 prompt 只是其中一小部分。

![元上下文工程框架](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-04.png>)

图中左侧是工作目录，不同迭代会保存 `SKILL.md`、`context/`、`data/rollouts` 等文件：
- 上半部分是元层 Skill 演化：Skill 数据库、指标、任务规格、元 Agent、Agent 式交叉共同生成新 Skill。
- 下半部分是基础层上下文优化：基础 Agent 根据任务规格、新 Skill、上下文函数和训练运行轨迹优化上下文。
右侧的评估与反馈用训练数据和验证数据反馈性能。

MCE 没有强制使用 ACE 那种结构化要点启发式方法。它使用自由形式的 Skills 保存任务中最重要的知识，并让 Skill 和由 Skill 条件化的上下文一起演化。实现上，上下文函数 $c$ 是一个目录里的文件集合，包含静态的 `skill.md`，也包含动态上下文和数据运行轨迹。

元层和基础层都在 Agent 式代码环境中执行，其可用的标准工具集合记为 $\mathcal{T}$：

$$ \mathcal{T}=\{\texttt{Read},\texttt{Write},\texttt{Edit},\texttt{Bash},\texttt{Glob},\texttt{Grep},\texttt{TodoWrite}\} $$

其中：

1. `Read`、`Write`、`Edit` 负责读、写、改文件。
2. `Bash` 负责执行命令。
3. `Glob`、`Grep` 负责文件发现和文本搜索。
4. `TodoWrite` 负责记录计划和待办。

这个工具集合很像 Coding Agent 的最小工作台。MCE 会落到文件系统和工具调用上：Agent 可以读取历史 Skill，编辑上下文文件，运行命令验证结果，再把新的上下文函数保存下来。

Meta-Harness 再往下一层。它直接优化决定信息如何存储、检索和呈现给模型的代码，不停在上下文本身或上下文 Skill 上。“Meta” 的意思就是用一个 harness 去优化另一个 harness。

Meta-Harness 外部 loop 优化算法：

![Meta-Harness 外层 loop 算法](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-05.png>)

算法输入任务集合 $\mathcal{X}$、LLM $M$、提案生成器 $P$ 和迭代次数 $N$。先初始化 harness 种群 $\mathcal{H}$ 和文件系统 $\mathcal{D}$。对每个初始 harness $H$，执行 `Evaluate(H, M, X)` 并把结果写入文件系统。之后每轮提案生成器查询文件系统 $\mathcal{D}$，查看过去 harness 和分数，再提出 $k$ 个新 harness。只有通过接口验证的候选会被评估并保存。最后返回文件系统中帕累托前沿（Pareto frontier）上的 harness。

算法主要靠 3 个设计点支撑：

1. 全部执行历史可通过文件系统访问，Coding Agent 用 `grep`、`cat` 等命令阅读历史，避免把所有内容塞进单个 prompt。
2. 每个候选 harness 都是文件系统里的一个字典，包含源代码、分数、运行轨迹和状态更新。
3. Meta-harness loop 会迭代创建新 harness，只保留合格候选。

![Meta-Harness 性能结果](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-06.png>)

左图展示文本分类任务上的搜索进展：Meta-Harness 在较少 harness 评估次数内快速提升，并超过 TTT-Discover、OpenEvolve、ACE、few-shot、zero-shot 等对照。
右图展示 TerminalBench-2 上的 harness 性能：Meta-Harness 达到 37.6%，高于 Goose、Terminus-KIRA、Mini-SWE-Agent、Terminus-2、Claude Code。需要注意的是，TerminalBench-2 的搜索从 Terminus-KIRA 和 Terminus-2 这类很强的 harness 初始化。

这条经验最值得记住：**只要 harness 设计被表示成可执行搜索空间，强大的 Coding Agent 就能探索人类工程师也会探索的设计空间。**

## 工作流设计（Workflow Design）

工作流设计可以由领域专家手工设计。自动化研究是典型案例：AI Scientist 系统会提出研究想法、写代码、跑实验、分析结果、写论文、做审查。ScientistOne 则把可验证性放在中心位置，要求引用、数值、方法、结论等每个声明都能追溯到证据来源，并通过证据链检查审计。

![AI Scientist 流程](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-07.png>)

图中流程分成想法构思、实验和写作：
- 想法构思里，LLM 提出的研究想法先经过新颖性检查，再评分和归档。
- 实验里依次进行初步调查、超参数调优、研究执行、消融研究，每一步都会写入日志，并把最佳结果传递给后续步骤。
- 写作里，系统进行画图和反馈、论文模板填充、论文写作，最后进行 AI 论文审查。

这个流程把科研拆成可执行阶段：产生想法、查新、归档评分、初步实验、调参、主实验、消融、画图、写稿、审稿。每个阶段都有日志和最佳结果，避免研究轨迹只存在模型上下文里。

Autodata Agent 被设计成数据科学家，用来生成训练和评估数据。主 Agent 管理挑战者大语言模型、弱求解器、强求解器和验证器/裁判模型，这些组件共同作用，旨在生成难度“恰到好处”的数据，也就是说，强解算器能够成功解决问题，而弱解算器则无法解决这些问题。

![Autodata 工作流](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-08.png>)

图中基础数据进入主 Agent。主 Agent 给挑战者大语言模型写 prompt，挑战者返回样例；主 Agent 同时把任务交给强求解器、弱求解器和验证器/裁判模型。验证器/裁判模型充当奖励模型，帮助主 Agent 判断生成数据的质量。挑战者 prompt 会根据求解器和验证器的反馈迭代更新。

Autodata 的限制也很明确：合成任务被用于微调弱求解器，没有用于微调强求解器。如果 loop 不能持续改善强模型，它更像是在生成 prompt 分布上做间接蒸馏，RSI 味道会弱一些。

工作流的设计空间巨大，因此可以把工作流设计看成搜索问题。自动化 Agent 系统设计（Automated Design of Agentic Systems, ADAS）把 Agent 设计本身定义为优化问题，也就是元 Agent 搜索。


![自动化 Agent 系统设计](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-09.png>)

图中元 Agent 读取 Agent 档案，生成新 Agent。新 Agent 包含总结与动机、名称和代码。经过任务测试后，如果表现好，就加入档案，成为下一轮生成的参考。下方展示了被发现的 Agent 示例：多步同行评审 Agent、已验证多模态 Agent、分治 Agent。

ADAS 的流程是：

1. 用 CoT、self-refine 等简单 Agent 初始化 Agent 式工作流档案。
2. 让元 Agent 参考档案中已有方案，用代码编写新 Agent。
   1. 先生成新工作流的高层描述。
   2. 再把它实现成代码。
   3. 草稿程序经过两轮自我改进，用模型给反馈，再根据反馈修改，用来检查新颖性。
3. 评估每个新候选，把成功方案加入档案。
4. 重复 2-3，直到达到最大迭代次数。

AFlow 把 Agent 式工作流表示成图：节点是调用 LLM 的动作，边是由代码实现的逻辑操作。优化使用蒙特卡洛树搜索（Monte Carlo Tree Search, MCTS）。


![AFlow 工作流优化](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-10.png>)

图左是搜索空间。节点中有固定参数，例如模型、温度、输出格式；也有可变 prompt 参数，例如 `PROMPT = """ Think step by step and then answer the question. Question: {question} """`。算子包括测试、格式化、程序员、集成、代码生成、审查与修改、上下文生成。边用代码表示，例如 `_call_(self, ...)`，根据响应的真假分支走不同路径。

图中间是通过 AFlow 搜索。系统用软混合概率选择候选工作流，用基于 LLM 的扩展生成新方案，再执行评估，并通过经验反向传播更新全局表现。

图右是搜索结果，展示数学工作流、问答工作流和代码生成工作流。

AFlow 的步骤是：

1. 用模板初始化起始工作流 $W_0$。
2. 使用分数和均匀探索的软混合策略选择工作流节点。
3. 让 LLM 根据该工作流的评估表现生成修改版本。
4. 执行并评估新工作流。
5. 如果新工作流在 $N$ 轮预算内表现提升，就加入搜索树。
6. 重复 2-5，直到 top-$k$ 平均分停滞或达到预算。


在 QA、代码和数学任务中的实验表明，AFlow 相比手工设计的工作流程和 ADAS 有着显著的改进。

![AFlow 实验结果](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-11.png>)

实验表比较了 HotpotQA、DROP、HumanEval、MBPP、GSM8K、MATH 六个基准测试。AFlow 平均分 80.3，高于 IO、CoT、CoT SC、MedPrompt、MultiPersona、Self Refine 和 ADAS。它在 HumanEval、MBPP、GSM8K 等任务上表现尤其突出，说明自动搜索出来的工作流可以超过一批手工方法。

## 自我改进 Harness（Self-Improving Harness）

上下文工程和工作流设计都只是 harness 的一部分。完整 harness 还包括上下文管理逻辑、工作流、权限、工具、子 Agent、控制流、记忆和评估。由于代码是定义程序和系统的通用语言，harness 可以被表示成代码。一个 LLM 如果能优化执行 Agent 的代码，就能进入比手写 prompt 大得多的设计空间。

自学优化器（Self-Taught Optimizer, STOP）是早期递归脚手架改进例子。递归过程在 $t=0$ 时从种子改进器 $I_0$ 开始。一般情况下，改进器 $I$ 接收初始解 $s$、效用函数 $u$ 和黑盒语言模型 $M$，返回改进后的解 $s'$：

$$s’ = I(u, s; M)$$

其中：

1. $s$ 是当前已有的解，可以是某段代码、某个 prompt、某个脚手架程序或某个任务方案。
2. $u$ 是效用函数，用来评价一个解好不好。
3. $M$ 是可调用的语言模型。
4. $I$ 是改进器函数，它利用 $u$、$s$ 和 $M$ 生成更好的解。
5. $s'$ 是改进后的解。

关键点在于：$I$ 是一个使用模型来改进解的程序或脚手架，模型 $M$ 只是它调用的能力来源。

STOP 的目标是改进改进器 $I$ 自身。为了衡量一个改进器的质量，其元效用被定义为改进器函数 $I$ 在一组下游任务 $\mathcal{D}$ 上的平均效用：

$$ \hat{u}(I) \triangleq \frac{1}{\vert\mathcal{D}\vert}\mathbb{E}_{(u,s)\sim \mathcal{D}}[u(I(u,s; M))] $$

其中：

1. $\hat{u}(I)$ 是改进器 $I$ 的元效用，也就是评价改进器本身的分数。
2. $\mathcal{D}$ 是下游任务集合，每个任务可以看成一对 $(u,s)$：一个效用函数和一个初始解。
3. $|\mathcal{D}|$ 是任务数量，用来做平均。
4. $(u,s)\sim \mathcal{D}$ 表示从任务集合中抽取任务。
5. $I(u,s;M)$ 表示改进器 $I$ 在语言模型 $M$ 的帮助下，把初始解 $s$ 改成新解。
6. $u(I(u,s;M))$ 表示用效用函数 $u$ 给新解打分。
7. $\mathbb{E}$ 表示对任务集合上的表现取期望。

它的含义是：如果一个改进器在很多不同任务上都能把初始解改好，那么这个改进器本身就更好。STOP 把优化对象从“某个任务的答案”上移到了“能改进很多任务答案的改进过程”。

递归脚手架改进的关键，是让上一轮改进器 $I_{t-1}$ 以元效用函数和自身为输入，再调用模型 $M$ 生成第 $t$ 轮的新改进器：

$$ I_t=I_{t-1}(\hat{u},I_{t-1};M) $$

其中：

1. $I_{t-1}$ 是上一轮改进器。
2. $\hat{u}$ 是评价改进器好坏的元效用函数。
3. $I_{t-1}(\hat{u},I_{t-1};M)$ 表示上一轮改进器把“元效用函数”和“自己”当作输入，再调用模型 $M$，生成一个新改进器。
4. $I_t$ 是第 $t$ 轮得到的新改进器。

这和前一个公式的区别很重要：前一个公式是“改进器改进解”，这一行是“改进器改进自己”。因此 STOP 会把改进过程本身放进优化 loop，已经超出普通任务优化。

![自学优化器算法](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-12.png>)

算法输入种子改进器 $I_0$、语言模型 $L$、递归深度 $T$ 和下游任务 $D$，输出改进后的改进器 $I_T$。每轮执行 $I_t <- I_{t-1}(\hat{u}, I_{t-1}, L)$，也就是用旧改进器根据元效用更新自己。函数 $\hat{u}(I)$ 会遍历下游任务，把改进器作用到初始解 $S$ 上得到 $S'$，再累加效用，最后返回平均效用。

这张算法图可以和上面的 3 个公式对应起来：

1. $I(u, s; M)$ 对应单个任务上的改进动作。
2. $\hat{u}(I)$ 对应跨任务评估改进器的质量。
3. $I_t=I_{t-1}(\hat{u},I_{t-1};M)$ 对应递归更新改进器本身。

从 harness engineering 的角度看，STOP 没有证明系统已经能无限自我改进。它给出的是一种形式化思路：**只要 harness 可以被表示成可修改程序，并且有可靠评估函数，就可以把 harness 自身放进优化 loop。**

![STOP 发现的策略](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-13.png>)

图中展示 STOP 发现的策略，包括遗传算法、拆分并改进组件、多臂 prompt bandit、通过改变温度探索、基于模拟退火的搜索、束搜索/树搜索。它们都可以理解为搜索或改进策略。harness 工作流也可以被当作优化对象，模型可能发现人类工程师也会使用的策略。

STOP 的警示结果也很重要：GPT-4 上平均下游表现会随迭代提升，但 GPT-3.5 和 Mixtral 这类弱模型可能退化。递归结构本身并不保证改进。**基础模型必须足够强，才能改进 harness**。Harness 改进能让模型部署得更好，但智能仍然是核心。

Lin et al. 把 harness 演化对模型能力的依赖拆成 2 个轴：

1. Harness 更新能力：产生有用 harness 编辑的能力。
2. Harness 受益能力：利用更新后 harness 解决任务的能力。

![Harness 更新能力与受益能力](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-14.png>)

图左显示 harness 更新能力在不同基础能力上相对平坦。Qwen3-32B、Qwen3-235B、GPT-OSS-120B、Sonnet 4.6、Haiku 4.5、Opus 4.6 之间差异不大，说明较小模型也可能写出和强模型程序同构的 Skill。

图右显示 harness 受益能力是非单调的。中等模型受益最多，例如 GPT-OSS-120B 和 Haiku 4.5；弱模型有两种失败模式：harness 激活失败，也就是 harness 没有加载；harness 遵循失败，也就是 harness 加载了但执行不正确。强模型则可能接近性能上限，因此新增 harness 带来的边际收益变小。

Self-Harness 用 LLM agents 通过“提出-评估-接受” loop 改进自己的 harness。

![Self-Harness loop](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-15.png>)

图中左侧是当前 harness $h_t$，包含 prompts、tools、memory、policies，并运行在固定模型 $M$ 上。上方蓝色区域是弱点挖掘：让 $h_t$ 执行任务，收集执行轨迹，再把失败聚类成失败模式，例如缺少验证、工具结果损坏。中间绿色区域是 harness 提案：用 $h_t$ 作为提案生成器，根据选中的失败模式提出候选 harness 编辑，例如先验证再下结论、loop 中断中间件、工具策略更新。下方橙色区域是提案验证：通过回归测试和晋级决策决定接受或拒绝。右侧是更新后的 harness，只有被接受的编辑才会进入下一轮。

Self-Harness 的 loop 有 3 个阶段：

1. 弱点挖掘：把失败聚类成由验证器支撑的失败模式。当前 harness $h_t$ 在任务上运行，收集执行轨迹。表面上两个运行可能都有相同验证器结果，例如超时或缺少产物，但背后因果机制不同，所以失败记录要包含验证器层面的原因、相关 Agent 行为的因果状态，以及轨迹暴露出的抽象 Agent 机制。
2. Harness 提案：基于失败模式提出有边界的 harness 编辑。同一个模型在 $h_t$ 下被当作提案生成器。它拿到的提案上下文有边界，包括当前 harness 的可编辑表面、评估系统发现的失败模式、需要保留的已通过行为，以及过去尝试过的编辑摘要。Harness 编辑应优先处理反复出现、可定位、可通过小改动解决的问题，并保持候选方案差异清楚且多样。
3. 提案验证：验证并合并合格编辑，生成 $h_{t+1}$。候选编辑会在内部保留集 $D_\text{in}$ 和外部保留集 $D_\text{out}$ 上做回归测试。内部保留集检查目标弱点是否被修复；外部保留集检查是否引入未知问题。只有两边都没有回归问题的候选才会被接受，被拒绝的候选会被记录，但不改变当前生效的 harness。

Self-Harness 在 Terminal-Bench-2 上运行 `MiniMax M2.5`、`Qwen3.5-35B-A3B` 和 `GLM-5` 时，能学到针对不同基础模型弱点的模型专属 harness 指令，并提升外部保留集通过率。

但 self-harness 也带来安全担忧。如果一个程序被允许编辑操作系统，抽象边界会被破坏。可编辑表面必须被严格设计，权限控制和安全层必须放在这个 loop 外面。奖励投机的问题仍然存在。

智能体 Harness 工程（Agentic Harness Engineering, AHE）把 harness 演化的瓶颈定位为可观察性。失败时必须知道哪个组件负责，每次编辑都要有证据支撑。

AHE 的闭环有 3 个可观察性支柱：

1. 组件可观察性：每个可编辑 harness 组件在文件系统中都有表示，所以行动空间显式且可追踪。Harness 包含 7 个组件：系统 prompt、工具描述、工具实现、中间件、Skill、子 Agent 配置、长期记忆。每个失败模式会映射到一个组件，编辑更有针对性。
2. 经验可观察性：把大量原始轨迹分析并总结成证据和失败模式的层级结构。每个 harness 生成 $k$ 条轨迹。Agent 调试器分析每条文件中的轨迹，生成单任务分析报告，说明失败或成功的根因。所有单任务报告聚合成基准测试总览，下一步需要时也能回看原始轨迹。这种分层访问结构更节省 token。
3. 决策可观察性：每次编辑都配有下一轮可验证预测。演化 Agent 读取仓库，决定编辑哪个组件，产出编辑和推理依据。每次编辑都是文件级、可证伪的声明，并在下一轮验证。

AHE 有 2 个重要约束：

1. 编辑只应用到 harness 工作区。运行目录、轨迹记录器、验证器和 LLM 配置只读。这样可以禁掉一批奖励投机，例如关闭验证器、替换模型、提高推理预算，从而让记录到的提升只能归因于 harness 编辑。
2. 编辑由证据驱动。清单条目要包含失败证据的名称、推断根因、针对性修复，以及预测影响，包括预期修复点和有风险的回归问题。

在 Terminal-Bench-2 上，AHE 除 Hard tier 和少数自我演化基线外，超过 OpenCode、Terminus-2、Codex 等人类设计的 harness。同一个冻结 harness 不继续演化，也能迁移到 SWE-bench-verified，说明演化出的 harness 把工程经验编码进了 harness 组件，没有只做面向特定基准测试的优化。

## 进化搜索（Evolutionary Search）

进化搜索（Evolutionary Search）模仿自然选择。它维护一个候选解种群，通过变异产生新候选，只保留适应度高的候选。它适合 2 种情况：

1. 搜索空间很大或形状很奇怪。
2. 很难用梯度直接优化，但候选方案容易评估。

Harness 搜索很符合这个条件。Harness 可以有很多工具、prompt、工作流、权限、记忆、子 Agent 配置组合，空间巨大；但如果有基准测试或验证器，就能对候选 harness 打分。

Prompt engineering 里已经用过进化搜索。Promptbreeder 优化面向特定任务的 prompt，而且变异 prompt 本身也通过演化改进。GEPA 把基于反思的 prompting 和进化搜索结合，用自然语言反思总结试错轨迹，再提出 prompt 更新。

AlphaEvolve 是 Coding Agent 进化搜索系统。它存储候选程序池，让冻结的大语言模型生成 diff 来改进程序。系统不断评估子程序，保留成功候选，随着时间发现更好的解。

![AlphaEvolve 系统](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-16.png>)

图中最上方是科学家/工程师，提供 prompt 模板和配置、选择现有或自定义 LLM、提供评估代码、提供带有可演化组件的初始程序。中间是 AlphaEvolve 系统组件：prompt 采样器、LLM 集成、评估器池、程序数据库。底部是分布式控制器 loop：从数据库抽样父程序和灵感材料，构造 prompt，让 LLM 生成 diff，把 diff 应用到父程序得到子程序，运行评估器，再把子程序和结果写回数据库。最终输出最佳程序。

AlphaEvolve 有几个关键设计：

1. Prompt 包含父程序、结果、指令，有时还包含元信息。
2. Coding Agent 可以访问完整仓库。
3. 可改进代码区域用 `# EVOLVE-BLOCK-START` 和 `# EVOLVE-BLOCK-END` 明确标记。
4. Meta-prompt 会和指令、上下文一起共同演化。

![AlphaEvolve 消融实验](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-17.png>)

消融图包含两个任务：矩阵乘法张量分解和 kissing number problem。完整方法表现最好。去掉 meta-prompt 演化、只用小基础模型、prompt 中不放上下文、不做完整文件演化、不做演化，都会降低效果。消融结果显示，演化过程、prompt 中的上下文、meta-prompt、完整文件演化和强 LLM 都是有效设计。

后续变体包括 ThetaEvolve、DemoEvolve 和 ShinkaEvolve。ThetaEvolve 结合进化搜索、RL 和上下文内学习。DemoEvolve 用人类专家示范扩充自我运行档案，作为 harness 层诊断和编辑的参考经验。ShinkaEvolve 加入 3 个组件提升 LLM 采样效率：

1. 更高效探索：父代采样平衡性能排名和子代数量。
2. 代码新颖性拒绝采样：丢弃和现有种群过于相似的候选，使用嵌入余弦相似度判断相似度。
3. 元草稿板：记录成功方案中的好模式，用来指导未来变异。

Darwin Gödel Machine（DGM）不同于只优化解的方法，它直接针对可编辑的 harness 代码仓库做演化。也就是说，一个基于 LLM 的 Coding Agent 被允许修改自己的 harness。

DGM 的流程是：

1. 从一个 Coding Agent 池开始。
2. 每轮选择一个父代，选择概率与性能成正比，并与它已有子代数量成反比。
3. 被选中的父 Agent 查看自己的基准测试评估日志，提出对自身 harness 代码库的改进，生成新版本 Coding Agent。代码编辑用两个基本工具：`bash`，参数是 `<bash_command>`；`editor`，参数是 `view/create/edit <file_path>`。
4. 新 Coding Agent 被评估，只有性能足够高的候选才加入池中。
5. 重复 2-4，直到达到停止条件。

DGM 是固定模型下的 harness 演化。使用 `Claude 3.5 Sonnet` 作为基础 LLM 和简单初始 harness 配置时，DGM 发现的 Agent 在 SWE-bench Verified 上从 20% 提升到 50%，在 Polyglot 上从 14.2% 提升到 30.7%，达到或超过手写 Agent。

这类方法适合候选方案可自动评估、适应度容易量化的场景，例如矩阵乘法、GPU kernel 优化、算法竞赛、数据中心调度。它不适合评估慢、模糊或主要依赖启发式判断的领域。计算效率和进化效果也是现实问题。

## 与模型权重联合优化（Joint Optimization with Model Weights）

Harness 演化改的是模型周围的非参数系统。要实现完整自我改进，也可以允许模型同时更新自身权重。权重更新可以通过训练 pipeline 改进，也可以通过测试时持续学习实现。

SIA 是早期尝试，把 harness 改进和模型参数更新放进同一个优化 loop。它有 3 个组件：

1. 元 Agent：提出初始 harness。
2. 任务专用 Agent：执行任务。
3. 反馈 Agent：根据近期轨迹决定更新 harness 还是更新模型权重。

![SIA 反馈 Agent loop](<images/Harness-12.-Lilian-Weng-Harness-Engineering-for-Self-Improvement-18.png>)

图左是“两个杠杆，一个 loop”。反馈 Agent 可以选择 harness 更新，也可以选择权重更新。Harness 脚手架包含 prompts、tools、重试和解析，由反馈 Agent 编辑；权重 $\theta$ 是基础 LLM 上的 LoRA 适配器，通过 RL 更新。图右是一个交错步骤序列：任务 Agent $A_1, A_2, A_3, A_4$ 和权重 $\theta_1, \theta_2, \theta_3$ 交替出现，反馈 Agent 决定每步是 H（harness 更新）还是 W（权重更新）。下方曲线表示两类更新交错后指标随步骤提升。

SIA 的方向有价值，但实验证据还不够干净。一个混杂因素是任务专用 Agent 明显弱于元 Agent 和反馈 Agent 使用的模型，例如 `gpt-oss-120b` 对比 `Claude Sonnet 4.6`；另一个问题是基线太弱，不容易和相关方法交叉验证。训练稳定性和古德哈特效应也仍然开放。

Continual Harness 在长周期游戏环境中测试 harness 更新，并通过蒸馏强教师模型在低奖励轨迹上的标签，共同训练策略模型。

# 未来挑战（Future Challenges）

AI Scientist 这条工作线证明，专家设计的 harness 可以协调自动化研究 loop 的很大一部分，尤其是写研究论文这种形式。但论文生产不等于科学发现。系统可以写出看起来合理的手稿，同时仍然有伪造引用、实现漂移或薄弱实验结果。

Trehan & Chopra 测试了 LLM 是否能用很少脚手架和基础工具，从研究想法走到论文。基础工具包括 `read_file`、`write_file`、`llm_search`、`list_files`。每个想法有独立工作区，Agent 可以生成和读取资料作为上下文。实验覆盖 3 个领域：世界模型、多 Agent RL、AI 安全与对齐。每个领域有 45-50 篇高质量种子资料来启发新想法。

最后只有 4 个想法被人类专家选中进入完整流程，只有 1 个真正执行成论文。实验观察到 6 个反复出现的失败模式：

1. 偏向训练数据默认值：使用旧库、过期命令、标准格式，或者使用没有被当前仓库和数据集支撑的假设。
2. 执行压力下的实现漂移：实现变复杂时，模型会退回常见简单方案，放弃原本提出的方法。
3. 记忆和上下文退化：长周期项目如果不把日志写成持久产物，就会丢失关键细节。
4. 过度乐观：实验有噪声或失败时，模型仍然宣布成功。类似 “p-hacking and eureka-ing” 模式，模型可能贴上数值胶带，然后把噪声当突破。
5. 领域智能不足：缺少隐性专业经验，例如预判实现复杂度、判断实验结果是否合理、知道哪些基线重要。
6. 科学品味弱：实验可以运行，但没有回答真正重要的问题。

面向完整 RSI，已经有真实进展，但仍然有 7 个瓶颈：

1. 评估器弱且模糊。很多研究主张没有快速、精确的验证器，许多真实任务也是如此。当前自我改进 loop 最适合指标可测、客观的任务，类似 RL 中奖励明确的场景。研究品味、新颖性、长期科学价值很难衡量。研究品味往往混合问题 framing、实验设计，以及判断哪些惊讶结果值得追、哪些失败案例值得重试。
2. 上下文和记忆生命周期。AI Agent 越自主，记忆越大。好的 harness 需要管理上下文和记忆，弥补长上下文生成的限制，同时最大化长周期任务成功率。人类能用一生维护记忆，这也提示上下文工程可能会成为智能的核心组成，不能只停留在软件系统层。
3. 负结果。研究生态偏向发表成功结果，文献天然有成功偏差。LLM 训练数据大多来自人类产出，也会受到这种偏差影响，因此可能不擅长判断何时放弃假设、报告负结果或承认失败。研究 harness 应该让失败尝试容易保留，因为从失败中学习是缩小任务搜索空间的最好方式。
4. 多样性坍缩。进化和 RL loop 容易利用已知高奖励模式，导致候选群体坍缩成同一类方案的变体。开放式研究尤其需要避免这种坍缩，因为最好的路径一开始可能在当前评估器下看起来并不好。
5. 奖励投机。自我改进 loop 会优化给定信号。如果奖励来自单元测试，Agent 可能过拟合测试；如果来自裁判模型，Agent 可能学会针对裁判的技巧；如果来自基准测试分数，Agent 可能利用基准测试伪影。评估器和权限控制应该放在 harness 演化 loop 外面，并配合外部保留测试、轨迹审计和关键节点上的人类审查。
6. 长期成功。外部优化 loop 通常依赖沙盒中可模拟的奖励。Coding Agent 已经提高日常软件工程效率，但很多优化目标仍然太短期。Agent 可以完成眼前任务，却未必保护由数百或数千工程师共同维护的仓库的长期健康。标准的基于沙盒的 RLVR 训练很难捕捉可维护性、所有权边界、迁移成本、向后兼容性和未来调试负担。
7. 人类角色。人类应该上移到更高抽象层，不该从 loop 中消失。人类需要在正确时间、正确抽象层提供监督。系统设计要考虑什么时候设置人类介入点，以及如何规模化这些监督点。

这些挑战很多都需要人类反馈和方向控制。构建这类技术的目标是服务更好的人类未来，不能反过来让人被系统牵着走。

# 附录：一些有用的基准测试（Appendix: Some useful benchmarks）

PaperBench 评估 AI 从零复现 20 篇 ICML 2024 Spotlight 和 Oral 论文的能力。任务包括理解论文贡献、开发代码库和成功执行实验。每个复现任务被拆成更小、可单独评分的任务，总共有 8,316 个评分细则，并与论文作者共同开发。当时最佳模型 `Claude 3.5 Sonnet` 约 21%，仍然没有超过机器学习博士。它还包含 PaperBench Code-Dev 和 JudgeEval。

CORE-Bench 评估已发表研究的计算可复现性。它基于 90 篇科学论文构造 270 个任务，覆盖计算机科学、社会科学和医学。任务要求从给定代码和数据复现结果，并包含多个难度层级，以及纯语言任务和视觉语言任务。当时报告中最好的 Agent，`GPT-4o` 和 `GPT-4o-mini`，在最难任务上只有 21% 准确率。

ScienceAgentBench 评估 LLM Agent 的数据驱动科学发现能力。它从 44 篇同行评审论文中抽取 102 个任务，覆盖数学、化学、生物和地理。任务类型包括数据处理、模型开发、数据分析和信息可视化。

RE-Bench 评估前沿 AI Agent 在真实机器学习研究工程环境中相对人类专家的表现。它包含 7 个开放式机器学习研究工程环境。每个环境由评分函数、起始解和参考解构成，并且能在 8 个或更少 H100 GPU 上运行。任务例子包括优化 kernel、运行 scaling-law 实验、修复 embedding、微调 GPT-2 做问答等。它还包含 61 位人类专家的 71 次 8 小时尝试。人类专家 82% 的 8 小时尝试拿到非零分，24% 匹配或超过强参考解。最佳 AI Agent 在 2 小时预算下比人类高 4 倍，但人类在更长预算下收益更好，并在 8 小时和 32 小时设置中超过 Agent。

MLE-bench 评估机器学习工程 Agent 在离线 Kaggle 比赛上的能力。它包含 75 个 Kaggle 机器学习工程比赛，测试训练模型、准备数据集、运行实验和提交预测到评分脚本的能力，并用 Kaggle 公开排行榜作为人类基线。论文中的最佳设置 `o1-preview` + AIDE 脚手架在 16.9% 的比赛中至少达到 Kaggle 铜牌水平，并包含资源扩展和污染分析。

KernelBench 评估 LLM 能否写出快速且正确的 GPU kernels。它包含 250 个 PyTorch 任务，核心指标 `fast_p` 表示生成 kernel 中既正确又快于基线的比例。这个基准测试特别适合进化搜索，因为候选 kernel 可以自动运行、验证正确性并测量速度。
