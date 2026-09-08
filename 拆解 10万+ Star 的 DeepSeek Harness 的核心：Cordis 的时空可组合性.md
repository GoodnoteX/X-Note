> 大家好，这里是 Goodnote（好评笔记）。本文详细介绍我们对论文《A Programming Paradigm for Spatiotemporal Composability》的理解。这篇论文解释了 DeepSeek Harness（DeepSeek 的 Agent 运行框架）底层元框架 Cordis 的设计。DeepSeek Harness 上线后很快接近 10 万个 GitHub Star（收藏数）。
![](<images/Harness-13.-Spatiotemporal-Composability-Cordis-01.png>)
> 北京大学与 DeepSeek-AI 团队在 2026 年提出一种面向**时空可组合性（spatiotemporal composability）** 的编程范式，直接回应动态插件系统和自我演化 Agent Harness 中一个很现实的问题：**组件可以随时加入、退出或替换，但它留下的副作用能不能完整撤销？依赖它的其他组件能不能自动停下、重连并恢复运行？** 这里把传统编程语言理论中的**效应（effect）** 和 **共效应（coeffect）** 变成运行时机制，并实现为**框架 Cordis**。它给出了一套完整方案，用数学定义、生命周期演算和工程实现解释运行中的系统怎样安全重组自己。

## 核心问题

问题可以直接落到一句话上：**如果一个系统允许组件在运行时持续加载、卸载和替换，怎样保证旧组件留下的修改被完整收回，同时让依赖关系随着组件变化自动重新组织**？

现有系统通常把问题推到进程或容器层：模块出问题就重启进程，服务变化就重新部署容器。这样确实能回收资源和重建依赖，但粒度太粗。一次插件更新可能让整个进程丢掉缓存、连接和进行中的任务；进程内组件之间的依赖也无法交给容器编排器处理。

解决方案把问题拆成两个互相正交的维度：

- **时间可组合性（temporal composability）**：组件退出时，它对共享环境造成的修改能够被完整、安全地撤销；
- **空间可组合性（spatial composability）**：组件能够声明自己依赖什么，运行时能够在依赖出现、消失或更换提供者时自动调整组件生命周期。

前者由**可回退效应（revertible effects）** 处理，后者由**响应式共效应（reactive coeffects）** 处理。二者被统一进同一个上下文，再通过组件生命周期演算推广到整个动态系统。

## 摘要

现代软件越来越依赖动态组合，从插件系统到能够修改自身组件的 Agent Harness，系统都需要在运行过程中加入、删除和重新配置功能。这里把动态组合拆成时间和空间两个维度，并把经典效应（Effect）、共效应（Coeffect）概念改造成运行时可以直接操作的机制。

- 可回退效应要求每次上下文变换都同时返回一个逆操作，运行时负责追踪和组合这些逆操作。
- 响应式共效应则要求组件声明依赖，每次上下文变化都按照这份声明判断组件应该激活、停用还是保持不变。

效应上下文与共效应上下文随后被统一为一种上下文类型，并进一步组成组件和动态组合演算。

这些机制最终实现为 Cordis：底层核心库负责效应追踪和共效应解析，上层组件加载器负责声明式配置协调与热模块替换。开源聊天机器人框架 Koishi 的生态中已有 4000 多个社区插件，为这套设计提供了生产系统中的使用案例。

# 1 引言

软件工程一直依赖“用简单部分组合复杂系统”。传统组合大多是静态的：函数调用、模块导入和类继承在编译或启动时确定，之后基本不变。插件架构和自我演化 Agent Harness 则要求**动态组合**，组件要在运行期间加入、退出和重新配置。

现有实践能够做到“动态加载”，却很难做到“安全卸载”和“依赖自动重连”。这两个问题分别对应时间和空间。

## 1.1 可组合性的两个维度

**时间可组合性关注组件在时间上的退出**。组件运行期间可能分配资源、注册事件、修改共享状态。卸载时必须把这些修改按正确顺序收回，让共享环境回到组件加入前的状态。

**空间可组合性关注组件在系统中的相互位置**。组件需要声明、发现和解析依赖；当依赖提供者出现、离开或换成另一个组件时，运行时还要协调所有相关组件的生命周期。

静态程序里，这两个问题相对简单：

- 时间维度通常退化为词法作用域中的资源管理，例如**资源获取即初始化（Resource Acquisition Is Initialization, RAII）** 或**括号式资源管理（bracket pattern）**；
- 空间维度通常退化为编译时模块导入解析。

> **词法作用域（lexical scope）** 指变量、资源或代码的有效范围由源代码的嵌套结构决定。例如进入一个函数或代码块时打开文件，离开这个函数或代码块时自动关闭文件。RAII 和 `bracket` 都利用这种明确的进入、退出边界安排资源清理。

动态系统没有固定的词法边界，也无法在编译时预知未来会出现哪些组件。一个部署之后才加载的插件，不能由原程序的静态作用域决定何时清理；运行时配置产生的新依赖，也不可能提前全部写进模块图。

## 1.2 为什么现有插件系统和 Agent Harness 不够

### 1.2.1 插件系统：能装进来，却很难单独拿出去

VSCode 被用作代表性例子。扩展都运行在共享的**扩展宿主进程（extension host）** 里，这个进程专门负责加载和执行扩展代码。扩展可以动态安装，但执行过激活函数 `activate` 的代码扩展无法在运行时单独卸载；禁用或删除它通常需要重启整个扩展宿主，其他扩展也会受到影响。

这里有两个具体限制：

- **运行时资源难以完整清理**：`activate` 会注册命令、事件监听器和后台任务，也可能打开文件或网络连接；`deactivate` 负责在退出时清理这些资源。但 `deactivate` 只是在扩展宿主进程终止时调用的退出钩子，不能支持单个扩展在线移除。资源的创建与清理又分散在两个函数中，很难确认是否全部一一对应。
- **组件依赖缺少结构约束**：VSCode 提供扩展依赖配置字段 `extensionDependencies`，用来在扩展清单中声明“当前扩展依赖哪些其他扩展”，但实际使用较少，扩展更常通过固定扩展点接入宿主。扩展之间也可以通过导出对象 `exports` 交互：一个扩展把函数或对象导出，另一个扩展再读取并调用。问题是 `exports` 默认返回任意类型 `any`，依赖方无法依靠经过检查的接口确定其中有哪些方法和字段。

VSCode 只是一个例子。多数插件系统都擅长“让插件接到宿主上”，却没有把插件之间的依赖和卸载后的完整恢复变成系统级保证。

### 1.2.2 自我演化 Agent Harness：修改自己之后还必须能救回自己

Agent Harness 需要组合工具、执行环境、权限与沙箱、会话状态、记忆、子 Agent 和自动化接口。未来的 Harness 还可能一边服务请求，一边由 Agent 自动生成并替换自己的组件。

这种场景比普通插件系统更依赖动态可组合性：

- 缺少时间可组合性，每次修改都要重启，进程内状态和进行中的任务会反复丢失；更严重的是，一次有缺陷的自我修改可能破坏负责恢复系统的进程本身。
- 缺少空间可组合性，每个模块都要自己监听依赖变化；直接替换代码还可能悄悄破坏下游组件，或者在重载时才暴露循环依赖。

因此，**自我演化不能只解决“怎样生成新代码”，还要解决“怎样把新组件安全接入运行中的系统，并在失败时撤回”**。

### 1.2.3 进程和容器是可用的替代方案，但粒度太粗

操作系统能够在进程粒度回收资源，K8S这类容器编排器能够在服务粒度管理依赖。多数系统正是依靠重启进程、替换容器来绕过细粒度动态组合问题。

代价同样直接：

- 重启会丢弃缓存、连接和中间计算，恢复可能需要数秒甚至数分钟；
- 为维持可用性还要准备冗余副本；
- 容器无法表达同一地址空间内组件之间的依赖；
- 原本可以是本地函数调用的交互，被提升到服务层后还会增加网络成本。

问题在于，系统缺少一种**与组件处于同一粒度的组合抽象**。

## 1.3 核心贡献

整套方案沿着五步推进：

1. **可回退效应**：每个上下文变换都返回显式逆操作，运行时追踪并组合这些逆操作，建立局部时间可组合性；
2. **响应式共效应**：组件声明依赖，上下文每次变化都被判定为激活、停用或中性变化，建立局部空间可组合性；
3. **统一上下文**：效应状态、逆操作累加器与依赖上下文被放进同一种递归上下文类型；
4. **动态组合演算**：把依赖声明、对外提供和效应函数组成组件，给出组件生命周期的操作语义与元理论；
5. **Cordis 实现**：核心库实现效应追踪和依赖解析，组件加载器实现配置协调与热模块替换。

# 2 预备知识

## 2.1 效应：程序对环境做了什么

**简单类型 λ 演算（Simply Typed Lambda Calculus）** 用一个判断式描述表达式的类型：在类型环境 $\Gamma$ 中，表达式 $t$ 的类型是 $T$。效应系统在这个判断式上继续标出执行 $t$ 时可能产生哪些副作用：

$$
\Gamma \vdash t : T^{\text{effect}}
$$

- $\Gamma$ 是类型环境，记录当前可用变量及其类型；
- $\vdash$ 是类型系统中的**推导符号**，表示“根据左侧环境，可以判断或推出右侧结论”；
- $t$ 是待判断的程序表达式；
- 冒号表示“具有……类型”，$T$ 是表达式的返回值类型；
- $\text{effect}$ 描述计算可能读取状态、写入状态、抛出异常或进行外部交互等副作用。

整条公式读作：**在类型环境 $\Gamma$ 中，可以判断表达式 $t$ 的返回类型为 $T$，并且执行它可能产生 $\text{effect}$ 所标记的副作用**。

这里介绍了两条经典路线：

- **单子效应（monadic effects）** 把带副作用计算包装成 $T(A)$。$\eta:A\rightarrow T(A)$ 把纯值放进计算上下文，$\mu:T(T(A))\rightarrow T(A)$ 负责把嵌套计算接起来。可选值 `Maybe`、状态计算 `State` 和输入输出计算 `IO` 都是典型例子。
- **代数效应（algebraic effects）** 把“能执行哪些操作”和“这些操作如何解释”分开。程序可以调用读取状态 `get`、写入状态 `put` 等效应操作，再由处理器接收操作参数和受限续体，决定继续、终止或重复执行。

两条路线都在回答同一个问题：**一段计算会怎样改变它周围的世界**。

## 2.2 共效应：程序需要环境提供什么

**共效应与效应方向相反。它不在结果类型上标记“会做什么”，而在上下文上标记“需要什么”：**

$$
\Gamma^{\text{coeffect}} \vdash t : T
$$

共效应可以表示所需资源、访问权限、环境信息或依赖服务。

- **余单子共效应（comonadic coeffects）** 描述依赖上下文的计算，例如固定环境或时间流；
- **分级共效应（graded coeffects）** 用半环中的值标记变量使用方式，例如 0 表示不用、1 表示线性使用、$n$ 表示有限次数、$\infty$ 表示不受限使用。

可以先记住一句话：**效应描述程序怎样改变环境，共效应描述程序运行前要求环境具备什么**。

## 2.3 效应与共效应如何对应动态可组合性

动态可组合性包含两个方向：组件对环境做了哪些修改，以及组件需要环境提供哪些条件。它们分别对应效应与共效应。

1. **时间可组合性，用可回退效应记录“组件改了什么”**：组件分配资源、注册事件或修改共享状态时，每次环境变换都要同时生成逆操作。组件卸载后，运行时按相反顺序执行这些逆操作，撤销它留下的修改。
2. **空间可组合性，用响应式共效应声明“组件需要什么”**：组件显式声明所需依赖，运行时负责寻找依赖提供者。依赖出现、消失或更换提供者时，系统重新判断组件应该激活、停用还是保持当前状态，并据此协调生命周期。

经典效应与共效应理论主要服务于编译期分析，作用范围也由固定的词法结构决定。动态组件却会在运行时到达和离开。这里把相关信息变成运行时对象：逆操作由运行时追踪，依赖声明进入依赖表，组件当前处于加载、活动还是卸载状态也被明确记录。这样，原本停留在类型系统中的概念才可以直接管理运行中的组件。

# 3 可回退效应与响应式共效应

这里要解决两个很实际的问题：**组件加载时改过的东西，卸载时怎样自动撤销；组件依赖的服务出现或消失时，系统怎样自动启停组件**。

- **可回退效应**负责前一个问题。每次修改环境时，同时记下“怎样撤销这次修改”；
- **响应式共效应**负责后一个问题。组件提前声明自己需要什么依赖，运行时持续检查这些依赖是否齐全；
- **统一上下文**把当前状态、撤销记录和依赖信息放在同一个运行时对象里。

形式化地说，这里把原本只用于类型检查的“上下文”变成程序运行时真正可以读取和修改的数据类型。

## 3.1 可回退效应

### 3.1.1 效应上下文：当前状态旁边始终带着恢复函数

**不纯函数（impure function）** 除了根据输入计算返回值，还会读取或修改函数外部的状态，例如修改全局变量、写入文件、更新数据库或注册事件监听器。只看 $f_{\text{impure}}:X\rightarrow Y$，只能看到输入 $X$ 和输出 $Y$，函数执行期间改了什么并没有写出来。

这里把所有可能被函数读取或修改的外部状态统一放进上下文 $\Gamma$，再把它显式地传给函数。函数接收“原来的上下文和输入”，返回“修改后的上下文和计算结果”：

$$
f:\Gamma\times X\rightarrow \Gamma\times Y
$$

其中：

- $X$ 是函数原本接收的输入类型；
- $Y$ 是函数原本返回的结果类型；
- $\Gamma$ 是函数可能读取或修改的完整外部状态；
- $\Gamma\times X$ 表示输入由“旧上下文和普通参数”组成；
- $\Gamma\times Y$ 表示输出由“新上下文和计算结果”组成；

例如，一个原本会直接修改全局配置的函数，现在可以写成“接收旧配置和目标值，返回新配置和执行结果”。这样，原来隐藏在函数内部的状态修改就被显式表示出来。固定普通输入 $x$ 后，只看上下文的变化，这个函数就相当于把一个旧状态变成一个新状态，即 $\Gamma\rightarrow\Gamma$。

为了让修改可以撤销，需要为正向操作 $f$ 配一个逆操作 $g$。如果满足 $g\circ f=\operatorname{id}_{\Gamma}$，意思就是：**先执行 $f$ 修改状态，再执行 $g$，结果等于什么也没有发生**。这里把 $g$ 称为 $f$ 的左逆。

连续执行多个修改时，撤销顺序必须反过来。例如先创建文件、再注册文件路径，清理时就要先取消注册、再删除文件。对应的组合写成：

$$
(f_1,g_1)\circ(f_2,g_2)
=
(f_1\circ f_2,\ g_2\circ g_1)
\tag{4}
$$

其中，$(f_1,g_1)$ 和 $(f_2,g_2)$ 分别表示两组“正向操作与对应逆操作”。公式复用了 $\circ$：它作用在函数上时表示普通函数组合，作用在两个“正向操作—逆操作”二元组上时表示扭曲组合。$f_1\circ f_2$ 表示先执行 $f_2$、再执行 $f_1$；$g_2\circ g_1$ 表示撤销时先执行 $g_1$、再执行 $g_2$。

公式中的函数组合从右向左执行，所以 $f_2$ 先执行、$f_1$ 后执行；撤销时则由 $g_1$ 先撤销 $f_1$，再由 $g_2$ 撤销 $f_2$。这就是常见的**后进先出**：最后做的修改最先撤销。这种“正向按一个顺序组合、逆操作按相反顺序组合”的方式称为**扭曲组合（twisted composition）**。

效应上下文被定义为：

$$
\partial\Gamma\coloneqq \Gamma\times(\Gamma\rightarrow\Gamma)
\tag{5}
$$

这里的 $\coloneqq$ 表示“定义为”，$\partial\Gamma$ 表示在普通上下文 $\Gamma$ 外再增加一层效应追踪能力。右侧第一项 $\Gamma$ 保存当前状态，第二项 $\Gamma\rightarrow\Gamma$ 是一个能把当前状态变回先前状态的函数。

一个效应上下文写成 $(\gamma,\varphi)$，也就是“**当前状态 + 一键恢复函数**”：

- $\gamma$ 是当前上下文状态；
- $\varphi$ 是逆操作累加器，把目前记录的所有撤销操作按正确顺序组合成一个函数；
- 初始状态是 $(\gamma_0,\operatorname{id}_{\Gamma})$。

追踪一个正向函数 $f$ 和逆函数 $g$ 时：

$$
\operatorname{track}_{\Gamma}(f,g)(\gamma,\varphi)
=
(f(\gamma),\ \varphi\circ g)
\tag{6}
$$

其中，$\operatorname{track}_{\Gamma}$ 是追踪操作，$f$ 是这次状态修改，$g$ 是对应的逆操作，$\gamma$ 是修改前的当前状态，$\varphi$ 是此前已经积累的恢复函数。输出中的 $f(\gamma)$ 是修改后的状态，$\varphi\circ g$ 是更新后的恢复链。

计算时先用 $f$ 更新 $\gamma$，再把 $g$ 放到旧恢复函数 $\varphi$ 前面。以后调用恢复链时，函数组合会先执行最新加入的 $g$，再继续执行此前保存的逆操作。此后只要调用下面的恢复函数，就能执行整条撤销链：

追踪操作不会改变正向操作本身的效果。把追踪结果中的当前状态取出来，等同于直接在原状态上运行 $f$：

$$
\operatorname{pr}_1\circ
\operatorname{track}_{\Gamma}(f,g)
=
f\circ\operatorname{pr}_1
\tag{7}
$$

- $\operatorname{pr}_1$ 表示从二元组中取出第一项，也就是当前状态；
- 左侧先运行 `track`，再从结果中取出状态；
- 右侧先从效应上下文中取出原状态，再直接运行 $f$；
- 两边相等，说明加入逆操作追踪不会偷偷改变 $f$ 的正向行为。

`track` 还会保持前面的扭曲组合结构：

$$
\operatorname{track}_{\Gamma}
\bigl((f_1,g_1)\circ(f_2,g_2)\bigr)
=
\operatorname{track}_{\Gamma}(f_1,g_1)
\circ
\operatorname{track}_{\Gamma}(f_2,g_2)
\tag{8}
$$

先组合两组“正向操作—逆操作”再追踪，与分别追踪后再组合，结果完全相同。`track` 因而可以直接处理连续效应，多步操作不需要另一套追踪规则。

$$
\operatorname{recover}_{\Gamma}(\gamma,\varphi)
=
(\varphi(\gamma),\operatorname{id}_{\Gamma})
\tag{9}
$$

其中，$\varphi(\gamma)$ 表示把积累的逆操作应用到当前状态，$\operatorname{id}_{\Gamma}$ 是恒等函数，满足 $\operatorname{id}_{\Gamma}(\gamma)=\gamma$。恢复完成后，累加器被重置为恒等函数，表示当前已经没有等待执行的撤销操作。

只要这次操作满足 $g(f(\gamma))=\gamma$，追踪这次修改不会改变最终恢复目标：

$$
\operatorname{recover}_{\Gamma}
\bigl(\operatorname{track}_{\Gamma}(f,g)(\gamma,\varphi)\bigr)
=
\operatorname{recover}_{\Gamma}(\gamma,\varphi)
\tag{10}
$$

左侧表示“执行并记录 $f$，然后恢复”，右侧表示“什么新操作都不做，直接恢复”。两边相等，说明只要 $g$ 能撤销这一次的 $f$，加入这次效应前后，系统最终都会回到同一个起点。

多个效应连续执行时，这个结论仍然成立：

$$
\operatorname{recover}_{\Gamma}
\Bigl(
\bigl(
\operatorname{track}_{\Gamma}(f_n,g_n)
\circ\cdots\circ
\operatorname{track}_{\Gamma}(f_1,g_1)
\bigr)(\gamma,\varphi)
\Bigr)
=
\operatorname{recover}_{\Gamma}(\gamma,\varphi)
\tag{11}
$$

这里 $f_1$ 到 $f_n$ 按顺序执行，每一步都满足自己的逆操作能撤销当前结果。累加器把 $g_1$ 到 $g_n$ 按相反顺序组织起来，最后调用一次 `recover`，整段操作序列就会被撤销。

### 3.1.2 可回退效应函数：逆操作要在执行时产生

上面的模型假设逆函数可以提前确定，但现实中，“怎样撤销”往往要等操作真正执行时才知道。例如把配置从 3 改成 7，撤销操作必须先记住旧值是 3；如果旧值是 5，撤销方式也要随之改变。

效应函数需要同时返回新状态和这一次操作对应的逆函数：

$$
\mathcal{E}_{\Gamma}\coloneqq
\Gamma\rightarrow\Gamma\times(\Gamma\rightarrow\Gamma)
$$

这里的 $\mathcal{E}_{\Gamma}$ 表示定义在上下文 $\Gamma$ 上的全部效应函数。它接收一个当前状态，返回两项：修改后的新状态，以及一个用于撤销本次修改的函数。

$\mathcal{E}^{*}_{\Gamma}$ 表示已经带有可回退保证的效应函数：

$$
\begin{aligned}
\mathcal{E}^{*}_{\Gamma}
\coloneqq{}&
\bigl(e:\Gamma\rightarrow
\Gamma\times(\Gamma\rightarrow\Gamma)\bigr)
\\
&\times
\Bigl(
(\gamma:\Gamma)\rightarrow
\bigl(
(\delta:\Gamma)\times
(g:\Gamma\rightarrow\Gamma)
\times
(((\delta,g)=e(\gamma))\rightarrow g(\delta)=\gamma)
\bigr)
\Bigr)
\end{aligned}
\tag{12}
$$

定义看起来很长，实际只有两层要求：

- 第一部分给出效应函数 $e$，它接收旧状态并返回新状态与逆函数；
- 第二部分要求对任意旧状态 $\gamma$，只要 $e(\gamma)$ 返回 $(\delta,g)$，就必须满足 $g(\delta)=\gamma$。

星号 $*$ 表示这个效应函数附带了一项证明条件：返回的逆函数确实能够撤销本次修改。它不会增加新的执行步骤。

$$
e(\gamma)=(\delta,g),\qquad g(\delta)=\gamma
$$

- $e$ 是效应函数；
- $\gamma$ 是应用前状态；
- $\delta$ 是应用后状态；
- $g$ 是在这次应用现场生成的逆函数；
- 约束 $g(\delta)=\gamma$ 表示把逆函数 $g$ 用在操作后的状态 $\delta$ 上，确实能够回到操作前的状态 $\gamma$。这个可检查的保证称为“见证”。

两个效应函数的组合写成：

$$
(f\diamond g)(\gamma)
=
\mathbf{let}\ (\delta,s)=g(\gamma)\ \mathbf{in}
\mathbf{let}\ (\varepsilon,t)=f(\delta)\ \mathbf{in}
(\varepsilon,s\circ t)
\tag{13}
$$

这里的 $\diamond$ 表示效应函数的顺序组合，`let` 表示先计算右侧，再把结果拆给括号里的变量。需要注意，公式中的 $f$ 和 $g$ 都是**效应函数**，而 $s$ 和 $t$ 才是它们在这次执行中返回的逆函数。

计算顺序如下：

1. 先在 $\gamma$ 上执行效应函数 $g$，得到中间状态 $\delta$，同时得到撤销它的函数 $s$；
2. 再在 $\delta$ 上执行效应函数 $f$，得到最终状态 $\varepsilon$，同时得到撤销它的函数 $t$；
3. 最终返回状态 $\varepsilon$；
4. 把两个逆函数组合成 $s\circ t$。真正撤销时，先运行 $t$ 撤销后执行的 $f$，再运行 $s$ 撤销先执行的 $g$。

为了让某一个效应能够被单独撤销，还需要把普通上下文上的效应函数提升到效应上下文 $\partial\Gamma$ 上：

$$
\begin{aligned}
\operatorname{effect}_{\Gamma}(e)(\gamma,\varphi)
\coloneqq{}&
\mathbf{let}\ (\delta,g)=e(\gamma)\ \mathbf{in}
\\
&\Bigl(
(\delta,\varphi\circ g),
\operatorname{track}_{\Gamma}
\bigl(g,\operatorname{pr}_1\circ e\bigr)
\Bigr)
\end{aligned}
\tag{14}
$$

返回值仍然是“新状态 + 逆操作”，但操作对象已经提升到效应上下文：

- 先运行 $e(\gamma)$，得到新状态 $\delta$ 和逆函数 $g$；
- $(\delta,\varphi\circ g)$ 是更新后的效应上下文；
- $\operatorname{pr}_1\circ e$ 是 $e$ 的正向状态变换，也就是只取 $e$ 返回的新状态；
- $\operatorname{track}_{\Gamma}(g,\operatorname{pr}_1\circ e)$ 把“撤销本次效应”本身也包装成一个可追踪效应。

组件由此可以单独保存自己的逆操作，卸载时只运行这一部分。提升操作也能和前面的效应组合配合：

$$
\operatorname{effect}_{\Gamma}(f)
\diamond
\operatorname{effect}_{\Gamma}(g)
=
\operatorname{effect}_{\Gamma}(f\diamond g)
\tag{15}
$$

左侧先分别提升 $f$、$g$ 再组合，右侧先组合 $f$、$g$ 再整体提升，两种方式相同。这保证单个效应和复合效应可以使用同一套提升规则。

不过，提升后的逆操作能恢复什么，需要区分“当前状态”和“逆操作累加器”。设 $e(\gamma)=(\delta,g)$，提升后的新状态为 $\Delta$、逆操作为 $g'$，那么：

$$
g'(\Delta)
=
(\gamma,\ \varphi\circ g\circ f)
\tag{16}
$$

其中 $f=\operatorname{pr}_1\circ e$ 是 $e$ 的正向状态变换。这个结果说明：

- 第一项一定回到原状态 $\gamma$；
- 第二项只有在 $g\circ f=\operatorname{id}_{\Gamma}$ 时才会完全回到原累加器 $\varphi$；
- 即使累加器的函数表示没有逐字恢复，它在当前应用点上仍然保持同一个最终恢复目标。

这套提升让**选择性撤销**成为可能。逆操作离开原来的执行位置后能否安全运行，还要看它与其他组件的修改是否独立，下一小节专门处理这个条件。

### 3.1.3 效应独立性：为什么可以只卸载中间某个组件

按后进先出顺序撤销全部效应相对容易。难点出现在选择性卸载：组件 A、B、C 的操作彼此穿插，系统现在只想卸载 B，同时保留 A 和 C。

这里的“独立”专指一件事：**撤销其中一个组件的修改，不会破坏另一个组件已经完成的修改**。形式上需要满足两个条件：

1. 一个效应的正向操作和逆操作，要与另一个效应的所有正向操作和逆操作可交换；
2. 一个效应改变状态后，不能改变另一个效应本来会返回哪个逆函数。

为了同时覆盖一个效应的正向函数和它可能在不同状态下生成的所有逆函数，先定义效应 $e$ 的**变换幺半群**：

$$
\mathfrak{M}(e)
\coloneqq
\left\langle
\{\operatorname{pr}_1\circ e\}
\cup
\{\operatorname{pr}_2(e(\gamma))\mid\gamma\in\Gamma\}
\right\rangle
\tag{17}
$$

- $\operatorname{pr}_1\circ e$ 是效应 $e$ 的正向状态变换；
- $\operatorname{pr}_2(e(\gamma))$ 是 $e$ 在状态 $\gamma$ 上生成的逆函数；
- $\mid$ 读作“满足……条件的”，这里表示收集所有 $\gamma\in\Gamma$ 时可能产生的逆函数；
- $\cup$ 表示把正向函数和所有逆函数放到同一集合；
- 尖括号 $\langle\cdot\rangle$ 表示再加入这些函数的有限次组合以及恒等函数。

$\mathfrak{M}(e)$ 收集了效应 $e$ 在运行中可能施加到上下文上的全部状态变换。两个效应要保持独立，这些变换必须能够交换顺序：

$$
\forall f\in\mathfrak{M}(e_1),\
g\in\mathfrak{M}(e_2),\qquad
f\circ g=g\circ f
\tag{18}
$$

$\forall$ 表示对两个集合中的任意 $f$ 和 $g$ 都成立。这个条件不只比较两个正向操作，也比较正向操作与逆操作、两个逆操作之间的顺序。无论先运行 $f$ 还是先运行 $g$，最终状态都必须相同。

第二个条件要求，另一个效应改变状态后，当前效应仍然生成同一个逆函数：

$$
\forall g\in\mathfrak{M}(e_2),\
\gamma\in\Gamma,\qquad
\operatorname{pr}_2\bigl(e_1(g(\gamma))\bigr)
=
\operatorname{pr}_2\bigl(e_1(\gamma)\bigr)
\tag{19}
$$

左侧先让 $e_2$ 的某个变换 $g$ 改变状态，再询问 $e_1$ 会生成什么逆函数；右侧直接在原状态上询问 $e_1$。两边相等，表示其他效应不会改变 $e_1$ 的撤销方式。还要把 $e_1$ 和 $e_2$ 对调后再满足同样条件，二者才是相互独立的。

例如，A 往路由表增加键 `a`，B 增加键 `b`，两者互不覆盖，执行和撤销顺序就可以交换。若 A、B 都修改同一条有顺序含义的中间件链，先后顺序会影响行为，也就不能把它们当成独立效应。

在两两独立的条件下，逆操作不必严格按照全局后进先出的顺序执行。系统可以先卸载 B，再卸载 A 或 C，最后仍能恢复到正确状态。**独立性把“只能把所有修改一起倒放”变成“可以单独撤销任意组件的修改”**。

## 3.2 响应式共效应

### 3.2.1 共效应上下文：把依赖变成带类型的运行时表

可回退效应解决“怎么撤销修改”，响应式共效应解决“组件需要的依赖现在能不能用”。运行时把依赖保存在一张带类型的表中：键表示依赖名称，值是对应的服务或资源。

这张表被形式化为从依赖键到对应类型值的有限偏函数：

$$
\Sigma\coloneqq(k:K)\rightharpoonup \mathcal{V}_k
\tag{20}
$$

其中，$\rightharpoonup$ 表示**偏函数**：它不要求每个键都有对应值。整个公式表示，给定一个键 $k$，如果该依赖当前存在，就能取得类型为 $\mathcal{V}_k$ 的值；如果不存在，这次读取就没有定义。

- $K$ 是所有依赖键的集合；
- $\mathcal{V}_k$ 是键 $k$ 对应的值类型；
- $\sigma:\Sigma$ 是当前依赖表；“偏函数”表示并非每个键现在都有值；
- $\operatorname{dom}(\sigma)$ 是已经存在的依赖键。

最基本的读取和写入操作是：

$$
\operatorname{get}(k)(\sigma)=\sigma(k)
$$

`get` 接收依赖键 $k$ 和当前依赖表 $\sigma$，返回表中绑定在 $k$ 上的值。它要求 $k\in\operatorname{dom}(\sigma)$，也就是该键当前确实存在。

$$
\operatorname{set}(k,v)(\sigma)
=
(\sigma[k\mapsto v],\ \lambda\sigma'.\sigma'\setminus k)
\tag{21}
$$

各部分含义如下：

- $v$ 是准备注册到键 $k$ 上的依赖值；
- $\sigma[k\mapsto v]$ 表示在依赖表中增加绑定 $k\mapsto v$；
- $\lambda\sigma'.\sigma'\setminus k$ 是这次注册对应的逆函数；
- $\lambda\sigma'.\ldots$ 表示构造一个以恢复时状态 $\sigma'$ 为输入的函数；
- $\sigma'\setminus k$ 表示从恢复时的依赖表中删除键 $k$；
- 整个输出是一对“注册后的依赖表 + 将来撤销注册的函数”。

这里的 `set` 只负责注册新依赖，不负责覆盖旧值。它要求键 $k$ 当前尚不存在；如果该键已经存在，操作会失败，状态保持不变。注册成功后，它会同时返回“删除这个键”的逆操作。**提供依赖本身也是一种可回退效应**：提供者组件退出时，运行时可以自动撤销它注册的服务。

每个键还携带三类信息：值类型 $\mathcal{V}_k$、判断两个值是否等价的关系 $\simeq_k$，以及组件可以对该值执行的操作集合 $\mathcal{A}_k$。操作被限制在键对应的值上，提升到整个依赖表后也只读写这个键，不会碰其他键。

键 $k$ 上的一个操作 $a$ 被写成：

$$
a:
X_a\rightarrow
\mathcal{V}_k\rightharpoonup
\mathcal{V}_k
\times
(\mathcal{V}_k\rightharpoonup\mathcal{V}_k)
\times B_a
\tag{22}
$$

这条类型从左到右表示：

- $X_a$ 是操作 $a$ 接收的参数类型；
- 输入当前值后，操作可能成功，也可能因前提不满足而没有定义，所以使用 $\rightharpoonup$；
- 成功时返回新的依赖值 $\mathcal{V}_k$；
- 同时返回一个把新值恢复成旧值的逆操作；
- $B_a$ 是操作额外返回给调用者的结果类型，例如查询结果或新创建资源的句柄。

这个操作原本只作用于键 $k$ 对应的单个值。把它提升到整张依赖表后，公式变为：

$$
\begin{aligned}
a_{\Sigma}(x)(\sigma)
\coloneqq{}&
\mathbf{let}\ (v,g,b)=a(x)(\sigma(k))\ \mathbf{in}
\\
&\bigl(
\sigma[k\mapsto v],
\lambda\sigma'.\sigma'[k\mapsto g(\sigma'(k))],
b
\bigr)
\end{aligned}
\tag{23}
$$

计算过程是：

1. 从依赖表 $\sigma$ 读取键 $k$ 当前的值 $\sigma(k)$；
2. 用参数 $x$ 执行操作 $a$，得到新值 $v$、逆函数 $g$ 和结果 $b$；
3. 只把表中键 $k$ 的值替换为 $v$，其他键保持不变；
4. 生成一个表级逆函数，它在恢复时读取 $k$ 的当前值，再用 $g$ 恢复该值；
5. 把结果 $b$ 原样返回给调用者。

这条提升公式说明，键上的操作进入整个上下文后仍然只读写自己的键。后面证明“不同键上的操作彼此独立”，依靠的就是这个限制。

键 $k$ 上的每个操作还必须尊重该键的等价关系 $\simeq_k$：如果两个输入值被视为等价，那么操作要么在两边都能执行、要么在两边都不能执行；执行后得到的新值和逆操作仍应等价，返回给调用者的结果也必须相同。否则，后面的观察等价会被这个操作直接破坏。

### 3.2.2 声明与通知：依赖变化直接驱动组件启停

组件会提前列出自己运行所需的全部依赖，这个键集合记为 $d$。只有 $d$ 中的每个键都已经出现在依赖表里，组件的依赖才算满足：

$$
\sigma\vDash d
\quad\coloneqq\quad
\forall k\in d,\ k\in\operatorname{dom}(\sigma)
\tag{24}
$$

其中：

- $d$ 是组件声明的依赖键集合；
- $\sigma\vDash d$ 读作“依赖表 $\sigma$ 满足声明 $d$”；
- $\vDash$ 表示满足关系；
- $\forall k\in d$ 表示“对于 $d$ 中的每一个键 $k$”；
- $\operatorname{dom}(\sigma)$ 是依赖表当前已经绑定的全部键；
- 因此，右侧表示 $d$ 中不能缺少任何一个依赖。

依赖声明本身被定义为键集合：

$$
\mathcal{D}_{\Sigma}
\coloneqq
\operatorname{Set}(K)
\tag{25}
$$

$\operatorname{Set}(K)$ 表示由 $K$ 中元素组成的所有集合，所以一个 $d\in\mathcal{D}_{\Sigma}$ 就是组件需要的若干依赖键。它只声明“需要哪些键”，不直接保存这些键的值。

每次依赖表从 $\sigma$ 变为 $\sigma'$，运行时都重新检查一次“依赖是否齐全”，然后把这次变化分成三类：

$$
\operatorname{notify}_d(\sigma,\sigma')=
\begin{cases}
\text{activating}, & \sigma\nvDash d\land\sigma'\vDash d\\
\text{deactivating}, & \sigma\vDash d\land\sigma'\nvDash d\\
\text{neutral}, & \text{其他情况}
\end{cases}
\tag{26}
$$

这里，$\operatorname{notify}_d$ 根据依赖声明 $d$ 检查一次状态变化；$\sigma$ 是变化前的依赖表，$\sigma'$ 是变化后的依赖表；$\nvDash$ 表示“不满足”，$\land$ 表示左右条件必须同时成立。公式先分别判断变化前后是否满足 $d$，再根据结果决定组件是否切换生命周期：

- **激活**：原来缺依赖，现在全部具备，开始执行组件效应；
- **停用**：原来依赖齐全，现在至少一个消失，运行逆操作撤销组件效应；
- **中性变化**：依赖满足性没有改变，不需要切换生命周期。

这样一来，依赖数据库的组件不会在数据库服务出现之前启动。不过，关闭顺序比启动顺序更麻烦：提供者不能一退出就立刻删除依赖，因为消费者的清理代码可能还要最后使用一次它。系统必须先停用消费者，再撤销提供者，这个完整顺序会在第 4 节的生命周期演算中解决。

### 3.2.3 隔离与拦截：同一个依赖键可以按上下文得到不同结果

简单依赖表默认“同一个键在整个系统里只有一个结果”，但现实中还要解决两个问题：同一个依赖在不同局部环境里可能指向不同服务；访问依赖时也可能需要附加权限、日志或其他限制。

**共效应隔离（coeffect isolation）** 通过“键到隔离域、隔离域到值”的两级映射，让同一个逻辑键在不同上下文中解析到不同值：

$$
\Sigma_{\text{iso}}
\coloneqq
(K\rightharpoonup R)\times((r:R)\rightharpoonup\mathcal{V}_r)
\tag{27}
$$

其中：

- $\Sigma_{\text{iso}}$ 是支持隔离的共效应上下文；
- $R$ 是所有隔离域标识组成的集合；
- 第一张表 $K\rightharpoonup R$ 记作 $\rho$，负责把逻辑依赖键 $k$ 映射到隔离域 $r$；
- 第二张表 $(r:R)\rightharpoonup\mathcal{V}_r$ 记作 $\sigma$，负责把隔离域 $r$ 映射到该域里的真实依赖值；
- $\times$ 表示这个上下文同时包含这两张表。

隔离上下文中的读取、注册和隔离操作分别写成：

$$
\operatorname{get}(k)(\rho,\sigma)
=
\sigma(\rho(k))
$$

$$
\begin{aligned}
\operatorname{set}(k,v)(\rho,\sigma)
=
\Bigl(
&(\rho,\sigma[\rho(k)\mapsto v]),
\\
&\lambda(\rho',\sigma').
(\rho',\sigma'\setminus\rho'(k))
\Bigr)
\end{aligned}
$$

$$
\operatorname{isolate}(k,r)(\rho,\sigma)
=
(\rho[k\mapsto r],\sigma)
\tag{28}
$$

三条公式分别完成：

- `get`：先用 $\rho(k)$ 找到键 $k$ 在当前上下文对应的隔离域，再从 $\sigma$ 读取该域中的值；
- `set`：把新依赖写到 $\rho(k)$ 指向的隔离域，并返回删除该域绑定的逆操作；
- `isolate`：只修改键到隔离域的映射，把 $k$ 重新指向 $r$，不修改底层依赖表 $\sigma$。

`get` 要求 $\rho(k)$ 指向的隔离域已经存在依赖值，`set` 则要求该隔离域当前还没有值。如果一个键没有显式隔离映射，就约定它解析到自己的默认域，可以理解为 $\rho(k)=k$。例如，两个测试用例都请求 `database`，但各自把它映射到不同隔离域，因此可以拿到两套互不干扰的测试数据库。

**共效应拦截（coeffect interception）** 保留原来的依赖，在访问时附加元数据。依赖表保存的是“接收元数据后生成值”的提供者函数：

$$
\Sigma_{\text{inter}}
\coloneqq
\bigl((k:K)\rightarrow\mathcal{M}_k\bigr)
\times
\bigl((k:K)\rightharpoonup(\mathcal{M}_k\rightarrow\mathcal{V}_k)\bigr)
$$

$$
\mathcal{D}_{\text{inter}}
\coloneqq
(k:K)\rightharpoonup\mathcal{M}_k
\tag{29}
$$

其中：

- $\mathcal{M}_k$ 是访问键 $k$ 时可以携带的元数据类型；
- 上下文 $\Sigma_{\text{inter}}$ 的第一项记作 $\iota$，保存外层上下文附加的元数据；
- 第二项记作 $\sigma$，它把键 $k$ 映射到提供者函数 $\mathcal{M}_k\rightarrow\mathcal{V}_k$；
- $\mathcal{D}_{\text{inter}}$ 是组件自己的依赖声明，它除了声明键，还能为每个键附带元数据；
- 每个键的元数据提供合并运算 $\oplus_k$ 和空元数据 $\epsilon_k$。

读取、注册和拦截操作写成：

$$
\operatorname{get}(k,\mu)(\iota,\sigma)
=
\sigma(k)\bigl(\mu\oplus_k\iota(k)\bigr)
$$

$$
\begin{aligned}
\operatorname{set}(k,\psi)(\iota,\sigma)
=
\Bigl(
&(\iota,\sigma[k\mapsto\psi]),
\\
&\lambda(\iota',\sigma').
(\iota',\sigma'\setminus k)
\Bigr)
\end{aligned}
$$

$$
\operatorname{intercept}(k,\nu)(\iota,\sigma)
=
(\iota[k\mapsto\iota(k)\oplus_k\nu],\sigma)
\tag{30}
$$

- `get` 先把组件声明的元数据 $\mu$ 与上下文元数据 $\iota(k)$ 合并，再交给提供者函数 $\sigma(k)$ 生成最终依赖值；
- `set` 注册提供者函数 $\psi$，由函数根据元数据生成依赖值，逆操作仍然是删除这个键；
- `intercept` 把新的上下文限制 $\nu$ 合并进键 $k$ 的元数据，不改动提供者表；
- 元数据采用右侧优先的合并方式，因此外层上下文可以覆盖组件自己的声明。

例如，组件请求文件系统并声明目标路径，外层上下文可以额外加上“只读”限制。二者合并后再交给文件系统提供者，无须修改组件代码或文件系统实现。

隔离和拦截都不会直接修改所有组件共享的依赖表。它们从原上下文派生一个只在当前局部范围生效的新上下文。局部上下文退出后直接丢弃，不需要额外记录逆操作。

隔离改变的是**拿到哪个依赖**，拦截改变的是**怎样使用这个依赖**。

## 3.3 上下文范式

### 3.3.1 统一上下文：效应、恢复链和依赖表放在一起

前面分别构造了“带恢复链的状态”和“带依赖信息的状态”。这里把它们装进同一种上下文类型，让运行时只管理一个对象：

$$
\Gamma_{\infty}
\coloneqq
\mu\Gamma.\ \Gamma\times(\Gamma\rightarrow\Gamma)\times\Sigma
\tag{31}
$$

其中：

- $\Gamma_{\infty}$ 是最终统一后的上下文类型；
- $\mu\Gamma$ 表示取这个递归类型的固定点，可以直观理解为“上下文里面还可以继续放同样结构的上下文”；
- 第一项 $\Gamma$ 是当前层的状态；
- 第二项 $\Gamma\rightarrow\Gamma$ 是当前层积累的恢复函数；
- 第三项 $\Sigma$ 是当前层的依赖表；
- 两个 $\times$ 表示三部分被共同保存在一个上下文对象中。

递归类型允许上下文内部继续包含同样结构的子上下文，父上下文也就能管理多个子组件。加载组件时把它插入上下文并执行修改；卸载时运行该层保存的逆操作，把组件及其修改一起移除。

### 3.3.2 观察等价：恢复只需行为一致，无须逐字节还原

现实中的恢复很少能把物理状态还原得一模一样。`free` 释放内存后，堆布局未必与 `malloc` 前相同；删除一个自动生成的名称后，再创建时也可能得到新名称。

这里的“恢复”允许底层字节发生变化，只要求外部通过公开操作看不出差别。这叫作**观察等价（observational equivalence）**。例如，释放内存后地址布局可能变化；只要程序公开的操作观察不到这种变化，就可以认为恢复成功。

两个依赖上下文观察等价，需要满足两个条件：它们拥有相同的依赖键，而且每个键上的值在该依赖允许的操作下都表现一致：

$$
\sigma\simeq\sigma'
\quad\coloneqq\quad
\operatorname{dom}(\sigma)=\operatorname{dom}(\sigma')
\land
\forall k\in\operatorname{dom}(\sigma),\ \sigma(k)\simeq_k\sigma'(k)
$$

统一上下文的观察等价则由它的共效应投影决定：

$$
\gamma\simeq\gamma'
\quad\coloneqq\quad
\sigma_{\gamma}\simeq\sigma_{\gamma'}
\tag{32}
$$

其中：

- $\sigma$ 和 $\sigma'$ 是需要比较的两个依赖上下文；
- $\simeq$ 表示整体上的观察等价；
- $\operatorname{dom}(\sigma)=\operatorname{dom}(\sigma')$ 要求二者包含完全相同的依赖键；
- $\land$ 表示前后两个条件必须同时成立；
- $\simeq_k$ 是键 $k$ 自己定义的值等价关系；
- $\sigma(k)\simeq_k\sigma'(k)$ 表示两个上下文中键 $k$ 对应的值，虽然底层表示可能不同，但通过这个键公开的操作无法区分。
- $\sigma_{\gamma}$ 表示从完整上下文状态 $\gamma$ 中取出的共效应部分。只要 $\gamma$ 和 $\gamma'$ 的依赖表观察等价，这两个完整状态在当前体系中就被视为等价。

计算时先比较两张表的键集合，再逐个比较相同键上的值。只有所有键都通过各自的等价判断，两张依赖表才算观察等价。

这里的 $\simeq_k$ 由键 $k$ 对外提供的操作决定。如果外部操作从来不会比较内存句柄的具体编号，那么句柄从 12 变成 19 也可以视为等价；如果接口允许直接比较地址，编号变化就能被观察到，两个状态也就不再等价。

更严格地说，两个值是否“不可区分”，要用该键公开的操作进行任意有限次测试。所有测试在两个值上都同时可执行或同时不可执行，并且每一步的可见结果都相同，关系 $\simeq_k$ 才能把它们视为等价。判断依据是接口能够观察到的行为，底层数据结构可以不同。

要让观察等价真正用于效应恢复，还需要规定函数也必须尊重这种等价关系：

$$
\forall\gamma,\gamma'\in\Gamma,\qquad
\gamma\simeq\gamma'
\Longrightarrow
f(\gamma)\simeq f(\gamma')
\tag{33}
$$

这表示，如果两个输入在公开操作下无法区分，那么经过 $f$ 后得到的两个输出也必须无法区分。否则，$f$ 就会把原本不可见的底层差异暴露出来，观察等价也无法继续成立。

等价关系还要扩展到函数，以及效应函数返回的“状态—逆函数”二元组：

$$
f\simeq g
\quad\coloneqq\quad
\forall\gamma\in\Gamma,\ f(\gamma)\simeq g(\gamma)
$$

$$
(\delta,g)\simeq(\delta',g')
\quad\coloneqq\quad
\delta\simeq\delta'\land g\simeq g'
\tag{34}
$$

- $f\simeq g$ 不要求两个函数的代码相同，只要求它们在每个输入上都产生观察等价的结果；
- 两个效应结果等价，需要新状态 $\delta$ 与 $\delta'$ 等价，返回的逆函数 $g$ 与 $g'$ 也要等价；
- 这项要求不能省。效应函数会同时返回当前状态和逆函数，逆函数后面还要参与恢复过程。

观察等价让“两个操作是否相互独立”可以按对外行为判断，而不用要求底层表示完全一致：

- 不同键上的操作天然独立，因为各自只读写自己的键；
- 同一键上的操作是否独立，由该键公开的接口是否可交换决定；
- 组件之间共享的、顺序敏感的关系应通过显式依赖来排序，不能假装成独立效应。

对于两个共效应操作 $a$ 和 $a'$，除了状态变换能够交换，还要求其中一个操作不能改变另一个操作返回给调用者的结果：

$$
\forall x:X_a,\
g\in\mathfrak{M}(a'_{\Sigma}),\
\sigma\in\Sigma,\qquad
\operatorname{pr}_3\bigl(a_{\Sigma}(x)(g(\sigma))\bigr)
=
\operatorname{pr}_3\bigl(a_{\Sigma}(x)(\sigma)\bigr)
\tag{35}
$$

其中，$\operatorname{pr}_3$ 表示从操作结果中取出第三项 $b$，也就是返回给调用者的结果。左侧先让另一个操作的某个变换 $g$ 修改依赖表，再执行 $a$；右侧直接执行 $a$。两边相等，表示另一个操作不会改变 $a$ 的可见输出。把 $a$ 与 $a'$ 对调后也要满足同样条件。

组件的一段完整效应通常包含多个操作，后续步骤还可能取决于前一步的结果。这种连续计算写成：

$$
\begin{aligned}
\sigma\mapsto{}&
\mathbf{let}\ (\delta,s,b)=a_{\Sigma}(x)(\sigma)\ \mathbf{in}
\\
&\mathbf{let}\ (\varepsilon,t)=e_b(\delta)\ \mathbf{in}
(\varepsilon,s\circ t)
\end{aligned}
\tag{36}
$$

计算过程是：

1. 在依赖表 $\sigma$ 上执行操作 $a_{\Sigma}(x)$，得到新状态 $\delta$、逆函数 $s$ 和结果 $b$；
2. 根据结果 $b$ 选择后续效应 $e_b$；
3. 在 $\delta$ 上执行 $e_b$，得到最终状态 $\varepsilon$ 和逆函数 $t$；
4. 把两步逆函数组合为 $s\circ t$，恢复时先撤销后一步，再撤销前一步。

这里把依赖操作的返回结果直接交给后续效应。只要两个组件共同使用的键支持可交换操作，它们各自包含多步操作时仍能保持独立。

由此可以分工：**执行顺序不会改变结果的操作交给效应系统，允许灵活组合和撤销；顺序会影响结果的操作交给共效应系统，通过组件依赖明确规定谁先谁后**。

### 3.3.3 上下文范式的定位：它处在函数式显式状态与命令式隐式修改之间

函数式编程显式传递状态，容易推理，但每层函数都要接收并返回状态；命令式与面向对象编程允许直接修改共享状态，写起来方便，却很难从函数签名看出它改变了什么、依赖了什么。

上下文范式取两者中间路线。它没有要求开发者在每层函数中手动传递全部状态，但也没有让修改和依赖彻底隐藏在全局环境中：

- 像函数式方案一样，所有效应和依赖都经由明确上下文，因此能够追踪；
- 像命令式方案一样，开发者只写当前原子操作及其逆操作，不必手工传递完整状态或编写整套卸载流程；
- 组合效应的逆操作由运行时自动组合，依赖的解析和重连也由运行时负责。

# 4 动态组合演算

第 3 节处理的是局部问题：单个效应能否撤销，单个组件能否只读取声明过的依赖。到了完整系统里，多个组件会交错加载、停用，还可能在执行过程中注册子组件。**局部保证必须落到一套可执行的生命周期规则上，才能覆盖整个系统**。

这里分三步完成这件事：先定义组件、纤程和注册表；再从一个原子、即时、不会失败的基础生命周期开始；最后加入撤回等待、分步执行、异步操作和失败处理，并证明整套规则仍然满足时间与空间可组合性。

## 4.1 组件、纤程与注册表

这里先区分三个概念：

- **组件（component）** 是一份可复用的定义，写明自己需要哪些依赖、能够提供哪些内容，以及加载时要执行什么操作；
- **纤程（fiber）** 不是我们操作系统中的进程或者线程之类的,也不是编程语言中的协程，她是组件创建出的一个运行实例，每个纤程都有独立的依赖解析结果、逆操作和生命周期状态；
- **注册表（registry）** 保存系统当前存在的全部纤程，运行时通过它查找父子关系、依赖提供者和各纤程的状态。

三者的关系可以概括为：**组件负责定义能力，纤程负责实际运行，注册表负责统一管理所有纤程**。组件被实例化后形成纤程，纤程还可以在运行过程中注册子纤程；这些实例都会以唯一名称进入注册表，系统再根据注册表中的状态和依赖关系决定谁先加载、谁需要卸载。

一个组件由依赖声明、提供声明和效应函数三部分组成：

$$
\mathcal{C}_{\Gamma}
\coloneqq
\mathcal{D}_{\Gamma}\times\mathcal{P}_{\Gamma}\times\mathcal{E}^{*}_{\Gamma}
$$

也就是 $(d,p,e)$：

- $d$ 声明组件需要哪些依赖；
- $p$ 声明组件可能向外提供哪些键；
- $e$ 是组件激活时执行的可回退效应函数。

这里可以把 $d$ 和 $p$ 看成同一个组件接口的两个方向：$d$ 说明组件要从环境中读取什么，$p$ 说明它可能向环境写入什么。效应函数只能写入 $p$ 声明的键。

同一个组件可以实例化多次。每个实例称为**纤程（fiber）**，指带有独立生命周期状态的组件实例，与协程无关。一个纤程完整地写成：

$$
\langle d,p,e,\pi,\sigma,\tau,\theta\rangle.
$$

- $d$、$p$、$e$ 来自创建它的组件；
- $\pi$ 是父纤程，顶层纤程的父节点记作 $\operatorname{root}$；
- $\sigma$ 是这个纤程自己的**共效应提供表**，保存它实际向外提供的键和值；
- $\tau\in\{\bot,\top\}$ 是退休标记，新纤程为 $\bot$，收到退休请求后变成 $\top$；
- $\theta$ 是生命周期状态。

基础模型里的生命周期状态为：

$$
\Theta_{\Gamma}
\coloneqq
\operatorname{Inactive}
\mid
\operatorname{Active}(g,\omega).
$$

$g$ 是停用时要执行的逆操作累加器，$\omega:d\rightarrow\mathcal{N}$ 是**已提交视图（committed view）**，记录本次激活时每个依赖键实际解析到了哪个提供者纤程。依赖值可能相等，但提供者身份发生变化时，组件仍然需要重新加载，所以这里保存的是提供者名称。

当前演算把所有组件放在同一个共享作用域里，没有引入第 3 节提到的**隔离域（realm）**。因此，不同纤程声明的提供集合必须互不重叠。**只要组件会提供至少一个键，它在同一注册表中就只能存在一个实例**；可以大量重复实例化的是不对外提供键、只消费依赖或注册子组件的组件。

最简单的生命周期只有非活动和活动两种状态：

![](<images/Harness-13.-Spatiotemporal-Composability-Cordis-02.png>)

> 图1：基础组件生命周期。L-Reload 将满足依赖的非活动组件加载为活动状态，L-Unload 在目标状态改变后卸载活动组件。

解释：组件处于非活动状态时不会对外提供内容。所需依赖全部满足后，运行效应并进入活动状态；依赖消失、提供者变化或组件收到退休请求时，再运行累积的逆操作回到非活动状态。

运行状态 $\gamma$ 中还有一个纤程注册表 $F_\gamma$：

$$
F_\gamma:\mathcal N\rightharpoonup\mathcal F_\Gamma.
$$

$\mathcal N$ 是纤程名称集合，$\mathcal F_\Gamma$ 是上下文 $\Gamma$ 上的纤程集合，$\rightharpoonup$ 表示**部分函数**：并非每个可能的名称都对应一个正在运行的纤程。$F_\gamma$ 是从当前已使用名称到纤程实例的有限映射，父指针共同组成一棵以 $\operatorname{root}$ 为根的树。

每个活动纤程都有自己的共效应提供表。系统当前可见的共效应上下文，是所有活动纤程提供表的并集：

$$
\sigma_\gamma
\coloneqq
\bigcup
\{\sigma_m\mid m\in\operatorname{dom}(F_\gamma),\ \theta_m=\operatorname{Active}(-,-)\}
$$

每个依赖键最多只有一个提供者，所以这个并集不会冲突。记作 $\operatorname{provider}_k(\gamma)$ 的函数，会返回当前提供键 $k$ 的纤程。

这里要区分两类状态变化：

- **会被其他组件声明和解析的共享内容**写入纤程自己的 $\sigma$，成为共效应上下文的一部分；
- **其他普通状态修改**仍由逆操作累加器 $g$ 追踪，但不会进入 $\sigma$，因此也不会产生依赖顺序。

## 4.2 基础演算：目标视图驱动生命周期

每个纤程都有一个**目标视图（target view）**，表示它现在是否应该运行，以及每个依赖键应该由哪个纤程提供：

$$
\operatorname{target}_n(\gamma)=
\begin{cases}
\bot, & \tau_n\lor\neg(\gamma\vDash d_n)\\
(k\in d_n)\mapsto\operatorname{provider}_k(\gamma), & \text{其他情况}
\end{cases}
$$

- $n$ 是当前纤程；
- $\tau_n$ 是退休标记；
- $d_n$ 是所需依赖；
- $\bot$ 表示当前不应运行；
- `provider` 返回每个依赖键当前对应的提供者纤程。

目标视图只由两件事决定：组件是否已经退休，以及当前依赖能否满足。它和纤程内部状态并不相同：$\omega_n$ 保存的是本次激活时确认的提供者，$\operatorname{target}_n(\gamma)$ 表示现在应该使用的提供者。两者不一致，说明组件需要停用后重新加载。

系统处于**静止状态（quiescent state）**，意味着每个纤程都已经到达自己的目标：

$$
\operatorname{quiet}(\gamma)
\coloneqq
\forall n\in\operatorname{dom}(F_\gamma).
\begin{cases}
\operatorname{target}_n(\gamma)=\bot,
& \theta_n=\operatorname{Inactive},\\
\operatorname{target}_n(\gamma)=\omega_n,
& \theta_n=\operatorname{Active}(-,\omega_n).
\end{cases}
$$

也就是说，不该运行的纤程已经停用；应该运行的纤程已经按当前目标视图完成加载。

基础演算包含两类规则：

- **O-Insert**：插入一个名称尚未使用的新纤程。父纤程必须已经存在，新的提供集合不能与已有纤程冲突；
- **O-Retire**：把退休标记设为 $\top$。这只是退出请求，不会直接改写生命周期状态；
- **O-Remove**：只有纤程已经退休、处于非活动状态，而且没有子纤程时，才从注册表中删除；
- **L-Reload**：纤程处于非活动状态且目标视图不为 $\bot$ 时，执行效应函数，保存逆操作与已提交视图，然后进入活动状态；
- **L-Unload**：活动纤程的已提交视图与当前目标不一致时，执行逆操作并回到非活动状态。

退休和删除被分开处理。退休只是提出退出请求；如果组件仍然活动，必须先运行逆操作，不能直接删除注册表条目，否则恢复函数也会丢失。

组件还可以在自己的效应中注册子组件。这个注册动作同样是一种可回退效应：正向操作插入子纤程，逆操作把子纤程标记为退休。父组件的逆操作执行后，会逐层触发子组件和更深层后代退出。

这里的父子关系和依赖关系要分开看。父组件负责创建、退休子组件，但父组件退出时不必等待所有子组件完成停用；真正要求“消费者先退出、提供者后退出”的，是共效应依赖关系，后面会由撤回守卫处理。

为了让这些规则覆盖全部状态修改，组件效应还要满足**作用范围约束（confinement）**：

- **写入范围**：涉及注册表时，一个纤程只能修改自己的共效应表；它仍可修改注册表之外的环境状态，但必须同时返回对应逆操作。注册子组件是唯一例外，可以新增子纤程条目，逆操作则把该子纤程标记为退休；
- **读取范围**：它可以读取自己的表、声明过的依赖以及不属于任何纤程表的环境状态，但不能读取未声明的依赖，也不能根据其他纤程的生命周期控制字段做判断。

组件如果能绕过 $d$ 偷读其他纤程，或者直接修改别人的表，后面的依赖顺序与恢复证明就失去了前提。

生命周期规则没有规定固定调度顺序。多个纤程同时满足规则时，运行时可以任选一个执行；后面的定理需要对所有这类交错顺序都成立。

## 4.3 进行中的状态转换

基础演算假设加载和卸载都是原子、即时且不会失败的操作。现实运行时还要处理未完成的转换、异步等待和失败，因此生命周期被扩展为四种状态：

$$
\Theta_{\Gamma}
\coloneqq
\operatorname{Inactive}(\zeta)
\mid
\operatorname{Reloading}(i,g,\omega)
\mid
\operatorname{Active}(g,\omega)
\mid
\operatorname{Unloading}(g,\omega,\zeta)
$$

- $i$ 是尚未执行完的效应迭代器；
- $g$ 是已经积累的逆操作；
- $\omega$ 是本次加载时确认的依赖提供者视图；
- $\zeta$ 是正常空结果 $\bot$ 或错误信息。

在这四个状态中，Reloading、Active 和 Unloading 都带有累加器和已提交视图，因此统称为**已安装（installed）**：

$$
\operatorname{installed}_n(\gamma)
\coloneqq
\theta_n\neq\operatorname{Inactive}(-).
$$

纤程停在 $\operatorname{Inactive}(\xi)$ 且 $\xi$ 是错误时，则称为**失败（failed）**：

$$
\operatorname{failed}_n(\gamma)
\coloneqq
\exists\xi\in\Xi.\
\theta_n=\operatorname{Inactive}(\xi).
$$

四状态模型也重新定义了静止条件：

$$
\operatorname{quiet}(\gamma)
\coloneqq
\forall n\in\operatorname{dom}(F_\gamma).\
\begin{cases}
\zeta\neq\bot\ \lor\ \operatorname{target}_n(\gamma)=\bot,
& \theta_n=\operatorname{Inactive}(\zeta),\\
\operatorname{target}_n(\gamma)=\omega_n,
& \theta_n=\operatorname{Active}(-,\omega_n),\\
\bot,
& \text{其他情况。}
\end{cases}
$$

正常停用的纤程只有在目标为 $\bot$ 时才算静止；带错误结果的 Inactive 纤程不会自动重试，因此也算静止；Active 纤程需要已提交视图与目标视图一致。Reloading 和 Unloading 仍在转换过程中，都不算静止。

只有 Active 纤程的 $\sigma$ 会进入系统当前的共效应上下文。Reloading 和 Unloading 都不会成为新消费者眼中的提供者，但它们仍然保留 $\omega$，可以在加载或清理过程中继续访问本次激活时确认的依赖。这样一来，尚未完成加载的内容不会过早暴露，正在退出的提供者也不会再被新的消费者选中。

![](<images/Harness-13.-Spatiotemporal-Composability-Cordis-03.png>)

> 图2：包含进行中转换的组件生命周期。Reloading 和 Unloading 是两个过渡状态；加载可以迭代、完成、转向卸载或因错误转向卸载，活动组件也可以先离开服务再完成卸载。

解释：L-Begin 开始加载；L-Iter 执行一个效应步骤；L-Finish 提交为活动状态。若加载期间目标依赖已经变化，L-Divert 会转向卸载；若步骤失败，L-Raise 同样先进入卸载，清理已完成的部分。活动组件收到退出信号后，L-Leave 先停止对外提供依赖，等消费者退出后，L-Unload 再运行逆操作。

### 4.3.1 撤回：消费者先清理，提供者再完成退出

这里的**撤回**，指提供者准备收回自己对外提供的能力或资源。它不能一上来就把资源拿走，因为已有消费者的清理过程可能还要用到它；所以要先停止接纳新消费者，等旧消费者退出后，再真正完成收回。

提供者退出时，消费者的清理代码可能仍要使用该依赖。例如关闭连接池时，消费者可能要把连接归还给提供者。

基础演算把“停止提供”和“运行逆操作”放在同一步里，消费者没有时间完成自己的清理。扩展后的卸载拆成两步：

1. **L-Leave**把提供者标为 Unloading，使新消费者不再把它视为可用提供者，同时保留自身提供表、累加器和已提交视图；
2. **L-Unload**等待所有仍依赖它的组件完成停用，然后才运行逆操作并删除提供内容。

“仍被依赖”可以正式写成：

$$
\operatorname{relied}_n(\gamma)
\coloneqq
\exists m\neq n,\ k\in d_m.
\operatorname{installed}_m(\gamma)
\land
\omega_m(k)=n.
$$

也就是存在另一个已安装纤程，它的已提交视图仍把某个依赖键解析到 $n$。只要这个条件成立，$n$ 的 L-Unload 就不能执行。

L-Leave 之后，$n$ 已经从新的目标视图中消失。依赖它的消费者会发现目标发生变化，继而进入自己的清理流程。**在依赖关系无环、迭代长度有界且纤程集合有限时，这条等待链最终会解除**。如果依赖关系本身成环，则不能直接声称守卫一定不会卡住。

守卫按实际绑定关系工作，而不是粗略地按整个纤程判断。没有把任何依赖解析到 $n$ 的组件不会阻塞它退出，父子关系也不会自动触发这条守卫。

### 4.3.2 迭代：长加载过程可以在步骤边界停止并局部回滚

这里的**迭代**，指把一次较长的加载过程拆成多个连续步骤。每完成一步，就保存这一步的结果、逆操作和下一步入口；中途发生变化时，系统只需要撤销已经完成的步骤，不必把整个加载过程当成无法拆开的黑盒。

组件激活可能包含多个效应步骤。效应迭代器每一步返回新状态、当前步骤的逆操作，以及是否还有下一步：

$$
\mathcal{E}^{\text{iter}}_{\Gamma}
\coloneqq
\mu\mathfrak{I}.\ 
\Gamma\rightarrow
\Gamma\times(\Gamma\rightarrow\Gamma)\times\operatorname{Maybe}(\mathfrak{I})
$$

$$
\begin{aligned}
\mathcal{E}^{\mathrm{iter}*}_{\Gamma}
\coloneqq
\mu\mathfrak{I}.\;&
\Bigl(
e:\Gamma\rightarrow
\Gamma\times(\Gamma\rightarrow\Gamma)\times\operatorname{Maybe}(\mathfrak{I})
\Bigr)\\
&\times
\Bigl(
(\gamma:\Gamma)\rightarrow
\bigl(
\mathbf{let}\ (\delta,g,o)=e(\gamma)\ \mathbf{in}\
g(\delta)\simeq\gamma
\bigr)
\Bigr).
\end{aligned}
$$

其中：

- $\delta$ 是当前步骤执行后的新上下文；
- $g$ 是当前步骤对应的逆操作；
- $\operatorname{Nothing}$ 表示执行结束；
- $\operatorname{Just}(i)$ 表示还有下一步迭代器 $i$；
- 上标 $*$ 表示除了迭代器本身，还带有证明每一步可以恢复的见证。

带见证的迭代器要求当前步骤能够恢复：若 $e(\gamma)=(\delta,g,o)$，则必须满足 $g(\delta)\simeq\gamma$。这里的 $\simeq$ 是观察等价，不要求恢复后的内部表示逐字节完全相同。每一步的逆操作会按执行顺序加入累加器：

$$
g_{\mathrm{acc}}
=
g_1\circ g_2\circ\cdots\circ g_k.
$$

真正执行恢复时，函数复合会先运行最后得到的 $g_k$，再依次向前，因此撤销顺序是**后进先出**。

把普通迭代器提升为可追踪效应时，使用下面的递归变换：

$$
\begin{aligned}
\operatorname{effect}^{\mathrm{iter}}_{\Gamma}
&:\mathcal E^{\mathrm{iter}}_{\Gamma}
\rightarrow\partial\Gamma\rightarrow\partial^2\Gamma,\\
\operatorname{effect}^{\mathrm{iter}}_{\Gamma}(i)(\gamma,\varphi)
&\coloneqq
\mathbf{let}\ (\delta,g,o)=i(\gamma)\ \mathbf{in}\\
&\quad\mathbf{let}\ t=
\operatorname{track}_{\Gamma}
\bigl(g,\operatorname{pr}_1\circ i\bigr)\ \mathbf{in}\\
&\quad\mathbf{match}\ o\ \mathbf{with}\\
&\qquad\operatorname{Nothing}
\Rightarrow
\bigl((\delta,\varphi\circ g),t\bigr),\\
&\qquad\operatorname{Just}(i')
\Rightarrow
\mathbf{let}\ (s,r)=
\operatorname{effect}^{\mathrm{iter}}_{\Gamma}
\bigl(i'\bigr)(\delta,\varphi\circ g)\ \mathbf{in}\
(s,t\circ r).
\end{aligned}
$$

这里的 $(\gamma,\varphi)$ 是当前上下文和已经积累的逆操作。当前迭代先得到 $\delta$、$g$ 与后续标记 $o$，再用 $\operatorname{track}_{\Gamma}$ 把本步撤销也包装成可追踪效应 $t$：

- 返回 $\operatorname{Nothing}$ 时，迭代结束，结果是更新后的上下文 $(\delta,\varphi\circ g)$ 和本步追踪操作 $t$；
- 返回 $\operatorname{Just}(i')$ 时，继续递归执行下一步，得到后续结果 $s$ 和恢复操作 $r$，再组合为 $t\circ r$。实际恢复时会先执行 $r$ 撤销后续步骤，再执行 $t$ 撤销当前步骤。

这条变换把整段迭代包装成与普通效应相同的接口，因此组件加载、嵌套组合和卸载可以继续使用前面定义的效应追踪机制。

加载过程对应四条规则：

- **L-Begin**：从 Inactive 进入 Reloading，保存目标视图，并把累加器初始化为恒等函数；
- **L-Iter**：执行一步，保存下一迭代器，并把本步逆操作加入累加器；
- **L-Finish**：最后一步返回 $\operatorname{Nothing}$，纤程进入 Active；
- **L-Divert**：加载尚未结束，目标视图已经改变，纤程转入 Unloading，只撤销目前已经完成的部分。

普通效应函数是只有一步的特殊情况：第一步就返回 $\operatorname{Nothing}$。它仍会经过 Reloading，但最终只会出现“全部安装”或“完全没有安装”两种结果。

### 4.3.3 异步：已经发出的操作必须落地，再决定是否回退

这里的**异步**，指某个效应步骤发出后不会立即返回结果，中间存在一段等待时间。等待期间，组件的依赖或目标视图可能发生变化。系统不假设已经发出的操作一定能够取消：如果操作完成时目标仍然有效，就保留结果并继续加载；如果目标已经改变，就先接收结果和对应逆操作，再进入卸载并撤销这次修改。

一个异步步骤提交后，外部状态可能在等待期间变化。系统不能假设能够取消已经发出的 I/O，所以采用**惯性（inertia）**：步骤一旦启动就允许它完成；若完成时目标已经改变，就立刻进入卸载并撤销它产生的结果。

异步部分没有增加新的生命周期状态或规则，只限制 L-Divert 的选择：尚未发出的迭代可以直接放弃；已经发出、仍在等待结果的迭代必须先落地，拿到它产生的逆操作后再转入 Unloading。

所以 Reloading 和 Unloading 必须互相衔接。组件不会在目标变化后短暂进入 Active，以免下游组件依赖一个已经确定要离开的提供者。反过来，组件在卸载期间如果目标重新变得可用，也要先完成当前卸载，回到 Inactive 后再开始一次新的加载。

### 4.3.4 失败：先恢复已完成的部分，再记录错误

这里的**失败**，指加载过程中的某一步没有正常返回结果，例如端口绑定、文件访问或远程请求出错。失败不会直接抹掉前面已经完成的操作，系统要先运行累积的逆操作，把这些修改恢复，再把错误保存在当前纤程上。

端口可能已被占用，文件可能不存在，远端服务可能没有响应。某个迭代步骤失败时，L-Raise 不会直接把错误抛到整个系统，而是让当前纤程进入 Unloading：

$$
\mathcal{E}^{\mathrm{fail}}_{\Gamma}
\coloneqq
\mu\mathfrak{I}.\
\Gamma\rightarrow
\operatorname{Either}
\bigl(\Xi,\Gamma\times(\Gamma\rightarrow\Gamma)\times\operatorname{Maybe}(\mathfrak{I})\bigr)
$$

$$
\begin{aligned}
\mathcal{E}^{\mathrm{fail}*}_{\Gamma}
\coloneqq
\mu\mathfrak{I}.\;&
\Bigl(
e:\Gamma\rightarrow
\operatorname{Either}
\bigl(\Xi,\Gamma\times(\Gamma\rightarrow\Gamma)\times\operatorname{Maybe}(\mathfrak{I})\bigr)
\Bigr)\\
&\times
\Bigl(
(\gamma:\Gamma)\rightarrow
\bigl(
\mathbf{let}\ \operatorname{Right}(\delta,g,o)=e(\gamma)\ \mathbf{in}\
g(\delta)\simeq\gamma
\bigr)
\Bigr).
\end{aligned}
$$

第一条公式定义了允许失败的效应迭代器，第二条带星号的公式在此基础上增加了**可恢复性见证**。其中：

- $\Gamma$ 是运行上下文，$\Xi$ 是错误集合；
- $\mu\mathfrak{I}$ 表示递归定义迭代器类型，$\mathfrak{I}$ 代表可能返回的下一步迭代器；
- $\operatorname{Right}(\delta,g,o)$ 表示当前步骤正常完成，返回新上下文 $\delta$、逆操作 $g$ 和后续迭代 $o$；
- $\operatorname{Left}(\xi)$ 表示当前步骤抛出错误 $\xi$；
- $\operatorname{Maybe}(\mathfrak{I})$ 表示可能存在下一步，也可能到此结束；
- $g(\delta)\simeq\gamma$ 要求成功步骤产生的逆操作能够把新上下文恢复到执行前的上下文。

恢复条件只约束 $\operatorname{Right}$ 分支。$\operatorname{Left}$ 表示当前步骤没有成功产出状态和逆操作，因此这一步本身没有新的内容可撤销；此前已经完成的步骤仍然要由累加器恢复。

- 运行失败前已经积累的逆操作；
- 清除当前组件造成的修改；
- 最终停在带错误信息的 Inactive 状态；
- 不自动重试，也不影响兄弟组件继续运行。

**错误是纤程级状态，而不是整个系统的失败状态**。

## 4.4 元理论：这套生命周期到底保证了什么

完整演算共有十条规则：三条编排规则，三条正常加载规则，加载提前结束时使用的 L-Divert 和 L-Raise，以及停用时使用的 L-Leave 和 L-Unload。

为了分析多个纤程交错执行的过程，用 $\gamma_t$ 表示执行前 $t$ 步后的状态，用 $\operatorname{step}_t=r(n)$ 表示第 $t$ 步在纤程 $n$ 上执行规则 $r$。一次**安装期（episode）**，是纤程连续保持已安装状态的最大区间。它通常从 L-Begin 开始、由 L-Unload 结束；如果执行序列结束时组件仍在运行，这次安装期也可以保持未闭合。

每一步真正作用于上下文的状态变换记作 $\Psi_t$：

$$
\Psi_t
\coloneqq
\begin{cases}
\operatorname{pr}_1\circ i,
& \text{L-Iter、L-Finish 或让当前迭代落地的 L-Divert},\\
g,
& \text{L-Unload},\\
\operatorname{id}_{\Gamma},
& \text{其他规则。}
\end{cases}
$$

$\operatorname{pr}_1\circ i$ 只取迭代器返回的新上下文，$g$ 是卸载时应用的逆操作累加器，$\operatorname{id}_{\Gamma}$ 表示上下文保持不变。随后再由 $\operatorname{edit}_t$ 修改生命周期等控制字段，因此完整的一步可以分解为：

$$
\gamma_{t+1}
=
\operatorname{edit}_t\bigl(\Psi_t(\gamma_t)\bigr).
$$

$\Psi_t$ 负责效应函数或逆操作造成的状态变化，$\operatorname{edit}_t$ 只负责生命周期状态、退休标记和注册表条目等控制信息。把两部分分开后，后面的证明可以分别检查业务状态有没有正确恢复，以及控制字段是否仍然良构。

后面的结论使用两种不同的等价关系：

$$
\begin{aligned}
\gamma\simeq\delta
\coloneqq\;&
\sigma_\gamma\simeq\sigma_\delta
\land
\operatorname{dom}(F_\gamma)=\operatorname{dom}(F_\delta)\\
&\land
\forall n,\ c\in\{\theta,\tau,\pi,d,p,e\}.\
c(\gamma(n))\simeq c(\delta(n)).
\end{aligned}
$$

- $\simeq$ 比较运行时能够观察到的共效应状态，同时要求注册表包含相同的纤程，并逐项比较生命周期、退休标记、父纤程、依赖声明、提供声明和效应函数等控制字段；
- $\approx$ 精确比较共效应表和外部状态，但忽略注册表中的生命周期控制字段。

撤销一个组件后，可能还留下已经退休、空表、没有子节点的注册表条目。它在 $\approx$ 下等同于不存在，但在控制结构上仍能被看见，所以两种关系不能混为一种“观察等价”。

表1把十条规则的写入位置集中在一起。

![](<images/Harness-13.-Spatiotemporal-Composability-Cordis-04.png>)

> 表1：十条动态组合规则对目标纤程的写入。表中列出规则执行前后的生命周期状态、对上下文施加的变换，以及被修改的控制字段。

解释：编排规则只改变注册表、退休标记等控制信息。L-Begin 只进入加载状态，真正执行组件效应的是 L-Iter、L-Finish 和需要让异步步骤落地的 L-Divert；只有 L-Unload 会应用完整累加器。后面的证明都建立在这张写入清单上。

五条系统保证还依赖三个辅助事实：

- **字段写入受限**：涉及注册表时，一个步骤只能修改它所作用纤程的提供表和控制字段；组件效应仍可修改注册表之外的环境状态，但必须返回对应的逆操作。注册子组件是唯一例外，它可以新增子纤程条目，之后再由逆操作将该子纤程标记为退休；
- **名称可以一致重命名**：纤程名称只是身份标记，只要父指针和已提交视图一起改名，执行结果不变；
- **残留条目通常不影响其他规则**：退休、非活动、空表且没有子节点的纤程条目，不会改变其他纤程的目标视图和依赖解析；但只要条目尚未删除，它仍占用自己的名称和提供集合，因此可能阻止 O-Insert 重用相同名称或声明重叠的键。

### 4.4.1 保持性：每一步都不会破坏注册表结构

良构注册表要求：

- 每个父指针都指向现有纤程或根；
- 不同纤程对外提供的键互不重叠；
- 已安装纤程的已提交视图覆盖全部声明依赖；
- 已提交视图指向的提供者也处于已安装状态。

父指针构成无环树这一点不需要额外规则检查，因为父纤程必须先存在，子纤程才能注册。保持性定理证明，只要初始注册表良构，任意一条规则执行后仍然良构。生命周期切换不会制造悬空依赖，也不会让两个纤程争用同一个键。

其中最容易忽略的是 L-Unload 的守卫。它不只负责安排退出顺序，还保证提供者离开已安装状态前，所有指向它的已提交视图都已经消失。这样，纤程删除后不会留下仍指向旧名称的依赖。

### 4.4.2 全局时间可组合性：其他组件穿插运行也不影响恢复

恢复精确性把第 3 节的局部结论推广到整段系统轨迹。先把一个迭代器能够到达的所有后续迭代收集起来：

$$
\operatorname{reach}(i)
\coloneqq
\bigcap
\left\{
S\ \middle|\
i\in S
\land
\forall i'\in S,\gamma.\
i'(\gamma)=(-,-,\operatorname{Just}(i''))
\Rightarrow i''\in S
\right\}.
$$

再用这些迭代产生的正向状态变换和逆操作，生成变换集合 $\mathfrak M(i)$：

$$
\mathfrak M(i)
\coloneqq
\left\langle
\{\operatorname{pr}_1\circ i'\mid i'\in\operatorname{reach}(i)\}
\cup
\{\operatorname{pr}_2(i'(\gamma))\mid i'\in\operatorname{reach}(i),\gamma\in\Gamma\}
\right\rangle.
$$

$\operatorname{reach}(i)$ 包含初始迭代器以及沿 `Just` 能继续到达的全部后续步骤；$\mathfrak M(i)$ 则包含这些步骤的正向操作、逆操作及其组合。允许失败时，这里的三元组按成功的 $\operatorname{Right}$ 分支读取。

两个迭代器 $i$ 和 $j$ 独立，需要满足：

$$
\forall f\in\mathfrak M(i),\ g\in\mathfrak M(j).\
f\circ g\simeq g\circ f,
$$

$$
\forall i'\in\operatorname{reach}(i),\ g\in\mathfrak M(j),\gamma\in\Gamma.\
\operatorname{pr}_{2,3}\bigl(i'(g(\gamma))\bigr)
\simeq
\operatorname{pr}_{2,3}\bigl(i'(\gamma)\bigr),
$$

并且交换 $i$、$j$ 后也要满足同样条件。第一条要求两个组件的正向操作和逆操作可以交换顺序；第二条要求另一个组件改变状态后，当前迭代器仍然返回相同的逆操作和后续迭代。只满足第一条还不够，因为操作顺序即使可以交换，后面选择的撤销函数或下一步也可能已经变了。

在这个前提下，某个纤程从开始加载到准备卸载之间，即使夹杂了其他组件的步骤，运行它的累加器仍会准确删除自身贡献，同时保留其他独立步骤造成的变化：

$$
g_n^u(\gamma_u)
\approx
(\Psi_{t_l}\circ\cdots\circ\Psi_{t_1})(\gamma_b).
$$

$b$ 是该安装期开始的位置，$u$ 是应用累加器前的位置，$t_1,\ldots,t_l$ 是期间由其他纤程执行的步骤。右侧表示只保留这些外部步骤得到的状态。

只有该纤程注册的子纤程没有在这段时间继续执行步骤时，右侧才能直接理解成“这个纤程从未开始加载”。如果子纤程已经运行，还需要在后面的合流证明中连同它们的步骤一起处理。

终止恢复进一步说明：一个组件无论正常卸载、加载中转向，还是加载失败，结束时都不会在共效应表和外部状态中留下自己的贡献。注册表里可能保留退休的空条目，但它只属于控制信息，不会继续影响依赖解析。

若安装期在 $u$ 处由 L-Unload 正式结束，终止后的状态满足：

$$
\gamma_{u+1}
\approx
(\Psi_{t_l}\circ\cdots\circ\Psi_{t_1})(\gamma_b).
$$

和前一条公式相比，这里左侧已经是卸载完成后的 $\gamma_{u+1}$。无论最终结果是正常退出、转向退出还是错误，业务状态中的自身贡献都已经被撤销。

### 4.4.3 全局空间可组合性：依赖顺序和解析结果保持一致

空间保证包含两部分：

- **顺序保证**：消费者只会在依赖提供者已经可用后开始加载；提供者完成撤回之前，所有仍依赖它的消费者必须完成停用；消费者在自己的清理阶段仍能访问当初提交的依赖。
- **解析一致性**：一次加载过程从开始到结束只能使用同一组依赖提供者。若提供者在加载途中变化，当前加载会转向清理，而不是把两套依赖混在一次执行里。

消费者开始加载的前提可以直接写成：

$$
\operatorname{step}_t=\operatorname{L\text{-}Begin}(m)
\Rightarrow
\gamma_t\vDash d_m.
$$

也就是执行 L-Begin 时，$m$ 声明的全部依赖已经得到满足。

顺序关系可以写得更直接一些。若消费者 $m$ 的一次安装期把键 $k$ 解析到提供者 $n$，那么 $n$ 的安装期开始得更早；若提供者 $n$ 的安装期也结束，$m$ 的安装期一定先结束：

$$
b_n<b_m,
\qquad
u_m<u_n.
$$

在消费者整个安装期内，$\omega_m(k)=n$ 保持不变，而且 $n$ 提供的 $k$ 一直可用，其值也不会改变：

$$
k\in\operatorname{dom}(\sigma_n^t),
\qquad
\sigma_n^t(k)=\sigma_n^{b'}(k).
$$

这里，$b'$ 是消费者安装期开始的时刻，$\sigma_n^t(k)$ 是时刻 $t$ 提供者 $n$ 对键 $k$ 提供的值。消费者因此能在整个安装期以及最后的清理代码中，继续使用开始时解析到的同一个依赖值。

解析一致性则约束整个 Reloading 区间。设 $[b,r]$ 是纤程 $n$ 处于 Reloading 的初始区间，$\omega$ 是 L-Begin 保存的已提交视图，那么：

$$
\forall t\in[b,r].\
\operatorname{step}_t\in
\{\operatorname{L\text{-}Iter}(n),\operatorname{L\text{-}Finish}(n)\}
\Rightarrow
\operatorname{target}_n^t=\omega.
$$

只有当前目标仍等于加载开始时保存的 $\omega$，L-Iter 和 L-Finish 才能继续执行。目标一旦变化，正常加载路径就会停止，改走 L-Divert；当前步骤抛出错误时则走 L-Raise。

异步操作带来一个例外：目标视图变化时，已经发出的迭代仍要先落地。它可能短暂产生基于旧依赖的修改，但纤程不会进入 Active，而是直接转入 Unloading，并由累加器把这部分修改撤销。

### 4.4.4 进展：没有循环依赖时，生命周期不会永久卡住

先定义依赖优先关系：

$$
n\prec m
\coloneqq
p_n\cap d_m\neq\varnothing.
$$

$n\prec m$ 表示 $n$ 可能提供 $m$ 声明的某个依赖，因此 $n$ 必须先完成加载。这个关系是否无环是一项额外假设，定义本身不会自动排除循环依赖。

在以下条件下，生命周期能够持续推进并最终安静下来：

- 依赖优先关系无环；
- 每个效应迭代器长度有上界；
- 整段执行涉及的纤程名称集合有限；
- 外部编排操作已经停止，后续只执行生命周期规则。

如果 $S(n)$ 表示作用在纤程 $n$ 上的生命周期步骤数量，目标视图发生变化的次数定义为：

$$
V(n)
\coloneqq
\left|
\left\{
t\mid
\operatorname{target}_n^t
\neq
\operatorname{target}_n^{t+1}
\right\}
\right|.
$$

$V(n)$ 统计相邻两个状态之间，纤程 $n$ 的目标视图一共改变了多少次。若每个迭代器最多有 $K$ 步，那么：

$$
S(n)
\leq
(K+4)\bigl(V(n)+1\bigr).
$$

目标每保持一段时间不变，纤程最多经历一次离开、一次卸载、一次重新开始、至多 $K$ 次迭代，以及失败时额外的一次卸载。目标变化次数又受到依赖图中前置提供者步骤数量的约束，因此整个系统的生命周期步骤总数有限。

进展定理由此给出两个结论：非静止状态下一定有生命周期规则可以执行；任何无法继续延长的生命周期序列最终都会停在静止状态。这个结论不覆盖外部编排器持续插入新纤程的无限运行。

### 4.4.5 合流性：动态折腾之后，结果等同于从头静态装一遍

要描述最终应该活动的组件，先把依赖关系和父子注册关系合在一起：

$$
m\mathbin{\triangleright}n
\coloneqq
m\prec n
\lor
\pi_n=m.
$$

一个纤程属于最终的**支持集合** $A$，需要同时满足三点：它没有退休；创建它的父纤程仍受支持；它声明的每个依赖都能由受支持的纤程提供：

$$
n\in A
\coloneqq
\neg\tau_n
\land
(\pi_n=\operatorname{root}\lor\pi_n\in A)
\land
\forall k\in d_n.\
\exists m\in A.\ k\in p_m.
$$

依赖优先关系无环时，这个递归定义才有唯一结果。

支持集合根据声明的提供键 $p$ 计算，而实际运行状态取决于纤程真正写入了哪些键。为使两者一致，还要求组件对自己的提供声明是**完备的**：一次激活只要正常完成，就必须满足 $\operatorname{dom}(\sigma_n)=p_n$，安装 $p$ 中声明的全部键。

合流性成立需要满足：

- 最终状态已经静止；
- 没有纤程失败；
- 组件效应两两独立；
- 依赖优先关系无环；
- 每个组件激活完成后确实提供它声明的全部键。

在这些条件下，支持集合恰好等于最终处于 Active 的纤程集合：

$$
A
=
\{n\mid\theta_n=\operatorname{Active}(-,-)\}.
$$

这个等式把静态声明与运行结果连接起来：支持关系判断哪些纤程应该存在，生命周期最终让这些纤程进入 Active，其余纤程保持非活动。

满足这些条件时，任意执行轨迹都可以整理成一种规范顺序：保留相同的编排步骤，把最终支持集合中的纤程按依赖和父子关系排序，每个纤程只加载一次，不再卸载。不同轨迹即使采用不同调度，只要编排步骤相同，最终状态也只可能在新建纤程的名称上存在一致重命名，业务状态与控制结构保持相应等价。

可以把结论理解为：系统增加组件、替换提供者，再撤销替换，最后会回到直接按最终配置重新加载一次所得到的状态。中间经历过多少次加载和卸载，不会在最终状态里留下额外修改。

失败被明确排除在合流结论之外，因为某一步是否失败可能与执行时看到的状态有关。不同调度可能让一个纤程失败、另一个不失败；但失败纤程的效应仍会被清理，不会继续污染其他组件状态。

这项保证只针对最终状态，不覆盖系统在运行途中已经向外发送的内容。例如已经写入文件的字节，或已经发送到网络的数据报，不会因为状态回滚自动消失，这也是后面区分内部资源获取和外部发射的原因。

# 5 实现与案例

Cordis 是一个面向时空可组合性的**元框架（meta-framework）**。它不规定网页服务（Web）、数据库或用户界面（User Interface, UI）等具体业务能力，只负责动态组合语义。实现分三层：核心库、组件加载器，以及建立在它们之上的应用框架。

- 开源代码：[Cordis GitHub 仓库](https://github.com/cordiverse/cordis)
- 入门参考：[Cordis Primer](https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer)

## 5.1 核心库

表2把理论对象和 Cordis 运行时对象逐项对应起来。

![](<images/Harness-13.-Spatiotemporal-Composability-Cordis-05.png>)

> 表2：第 3、4 节理论结构与 Cordis 实现之间的对应关系，包括统一上下文、效应函数、依赖表、纤程字段、生命周期状态、目标视图和十条演算规则。

解释：理论里的 $\Gamma_\infty$ 对应一等对象 `ctx`，可回退效应对应 `ctx.effect`，依赖读写对应 `ctx.get/ctx.set`，组件实例对应 `fiber`。生命周期演算也不是停留在纸面：加载、迭代、离开和卸载分别落到 `execute`、`refresh`、`reload`、`unload` 等运行时代码中。

### 5.1.1 效应追踪：所有上下文修改都经过 `ctx.effect`

这里是在落地 3.1 节**可回滚效应**。`ctx.effect(callback)` 是唯一的上下文修改入口。回调可以返回一个逆操作，也可以作为迭代器连续产生多个逆操作。

执行过程可以概括为：

1. 调用回调，取得效应迭代器；
2. 每得到一个逆操作，就把它放到恢复链最前面；
3. 每步前检查防护条件 `guard`，若组件目标已变化，就停止后续迭代；
4. 返回组合后的清理函数 `dispose`；
5. `dispose` 最多执行一次，并被继续组合到父上下文的恢复链中。

运行时不会自动证明组件提供的逆操作一定正确。逆操作能否恢复对应效应，仍然是组件开发者的义务；Cordis 保证的是只要原子逆操作正确，组合后的执行顺序和生命周期回收就是结构化的。

### 5.1.2 共效应操作：提供、通知、隔离和拦截

每个上下文保存三张内部表：

- `@@store` 保存隔离域到依赖值；
- `@@isolate` 保存依赖键到隔离域；
- `@@intercept` 保存每个键的访问元数据。

`ctx.set` 通过 `ctx.effect` 安装值，同时返回删除值的逆操作。安装和删除都会调用通知函数 `notify`：运行时遍历所有仍然存在的纤程，找到声明了对应键且处于相同隔离域的消费者，再调用刷新函数 `refresh` 重新计算目标状态。这里不能只检查已经激活的纤程，因为正在等待依赖的非活动纤程，也需要在提供者出现时被重新激活。

一个依赖只有在安装它的提供者处于 Active 状态时才算真正可用。提供者一旦进入 Unloading，就会先停止对外提供依赖，消费者因此开始退出；但提供者安装的值此时仍然保留，要等消费者完成清理后才会被逆操作删除。

隔离和拦截采用派生上下文，不修改父上下文本身：隔离重定向键对应的域，拦截合并访问元数据。子上下文被丢弃时，这些调整也自然消失，不需要显式逆操作。

### 5.1.3 组件生命周期：重新加载（Reload）与卸载（Unload）互相衔接

`ctx.use` 把组件实例化为纤程，并让子组件注册本身成为父组件的一项可回退效应。父组件退出时，它注册的子组件会自动收到退休请求。

`refresh`、`reload` 和 `unload` 共同实现四状态生命周期。`fiber.target` 保存的不是依赖值本身，而是各项依赖当前对应的提供者纤程标识：

- `refresh` 重新计算目标提供者，若已有转换正在进行，只更新目标而不启动第二个并发转换；
- `reload` 提交当前依赖视图并运行组件效应，完成时再次比较目标；
- 目标不变就进入 Active，并通知正在等待该组件所提供依赖的消费者；目标已变则直接转入 Unloading；
- 进入 Unloading 后，组件先停止对外提供依赖，通知并等待相关消费者退出，再按后进先出顺序运行所有逆操作；
- 清理结束后，目标为空就进入 Inactive，否则立刻重新 Reload。

提供者标识是每个纤程唯一且不会复用的，因此，即使新旧提供者给出的依赖值完全相同，更换提供者仍然会被识别出来并触发重新加载。反过来，同一个提供者原地覆盖自己的依赖值不会触发这种切换；如果组件希望消费者感知替换，需要先撤回旧绑定，再安装新绑定。

这里同时有两个检查粒度：每个效应迭代步骤之间检查一次目标，整个加载或卸载完成时再检查一次。前者支持部分回滚，后者负责异步操作完成后的状态衔接。

### 5.1.4 上下文访问：代理对象在读取时检查依赖声明

除了 `ctx.get(key)`，Cordis 还允许像访问普通属性一样写 `ctx[key]`。TypeScript 的代理对象 `Proxy` 会拦截这次属性读取，并沿当前纤程的父链向上解析：

- 在已提交视图里找到键，返回对应依赖；
- 当前纤程声明过该键但尚未激活，抛出“非活动访问”；
- 一直走到根仍没有声明，抛出“未声明访问”。

因此，属性访问不只是查找当前绑定，还会检查当前组件是否声明并提交了这项依赖。消费者在清理阶段仍会读取自己激活时确认的提供者，不会切换到后来临时出现的新值。

这些检查目前发生在运行时。由于组件依赖是静态声明的，同类违规原则上也可以由宿主语言的类型系统或编译期元编程提前发现，第 6.4 节会继续讨论这一方向。

## 5.2 组件加载器

核心库为组件开发者提供命令式原语，组件加载器则面向系统编排者，把整个运行系统表示成声明式配置，并让实际纤程持续与配置保持一致。

### 5.2.1 声明式配置与增量协调

每个配置条目记录：

- 稳定标识 `id`；
- 组件模块地址 `url`；
- 隔离配置 `isolate`；
- 拦截配置 `intercept`；
- 组件配置 `config`；
- 是否禁用 `disabled`。

配置条目组成树形结构，是“系统应该加载什么”的权威记录。条目与纤程之间采用双向同步：条目字段变化时，加载器调整对应纤程；组件主动修改自己的配置或禁用状态时，变化也会写回条目。

条目既可以直接对应一个纤程，也可以成为包含子条目的分支。`@cordisjs/group` 根据子条目列表加载一个分组，`@cordisjs/include` 则把 YAML 或 JSON 配置文件作为子树接入当前配置。配置变化后，加载器选择代价最小的操作：

- `id` 或 `url` 变化，重建组件；
- `isolate` 变化，重新分配隔离域并通知受影响的消费者；
- `intercept` 变化，直接更新元数据，不触发重载；
- `config` 变化，由组件判断哪些差异需要重载；
- `disabled` 变化，停用或重新激活纤程。

编排器不必手工安排加载顺序。模块可以并发载入，缺少依赖的组件会停在激活之前，等提供者出现后再自动启动。在各组件会安装自己声明提供的全部依赖键等合流条件成立时，最终静止状态只由最终配置决定，中间经历过怎样的增量修改不会改变结果。如果组件只在某些配置下提供自己声明的键，最终激活哪些组件还会受到这些配置的影响。

`isolate` 支持两种作用域：值为 `true` 时创建条目私有的本地隔离域，这个隔离域会跟随条目一起移动；值为字符串时使用全局隔离域，所有写有相同名称的条目共享同一个域。没有条目继续使用某个隔离域时，该域才会被丢弃。

隔离域的动态迁移更细：条目在配置树中移动时，加载器先为发生变化的键写入新的分隔标记，记录新旧隔离域并重新加载条目；随后再用这些标记判断依赖绑定是否属于该条目自己的作用域。只有属于它的绑定会被移动，也只有因为域变化而得到或失去依赖的消费者会收到通知。

### 5.2.2 热模块替换：分类、检测、事务重载

Cordis 把热模块替换（Hot Module Replacement, HMR）建立在组件恢复机制上。旧纤程退出会撤销组件全部效应，新纤程再从更新后的模块重新安装，因此不要求开发者手工标注 HMR 接受边界。

HMR 分三阶段：

1. **模块分类**：从发生变化的文件出发，在导入图上求固定点，把模块分为可热替换和必须触发完整重启两类；导入循环中无法判断的模块默认拒绝热替换。
2. **过期条目检测**：计算每个配置条目的传递依赖，若依赖树与可替换模块相交，就把该条目标为过期。
3. **事务式重载**：备份并清理模块缓存，停用旧纤程，重新导入模块并创建新纤程；任一导入失败，就恢复缓存并用旧模块重新建立全部过期条目。

事务回滚保证系统不会停在“部分组件已经更新、部分仍是旧版本”的中间状态。

## 5.3 Koishi 案例

开源代码：[Cordis GitHub 仓库](https://github.com/koishijs/koishi)

Koishi 是建立在 Cordis 上的开源聊天机器人框架，四年间积累了 4000 多个社区插件，包括即时通信适配器、数据库驱动、管理控制台和业务功能。服务器端和浏览器控制台还是两套独立的 Cordis 应用，说明这套抽象没有绑定单一运行环境。

案例主要支持三个判断：

- **表达能力**：上下文、效应和共效应原语足以支撑完整生产框架，Koishi 本身只需要补充聊天机器人领域词汇；
- **时间可组合性**：插件可以在控制台中在线禁用，开发时也可以保存后热替换，而其他插件的缓存和连接继续保留。只要插件通过上下文执行操作，Cordis 就会追踪相应逆操作并按顺序清理，插件作者不需要另写一套完整的卸载流程；
- **空间可组合性**：通信适配器和数据库驱动对外提供依赖，功能插件声明并消费依赖；更换提供者时，只重新激活解析结果发生变化的消费者。依赖暂时不可用的插件会保持非活动状态，等依赖重新出现后再自动激活，而不是直接报错退出。

这项案例只能说明方案已经在一个生态和一种宿主语言中落地，不能当作与其他架构的受控性能比较。运行开销和开发效率仍缺少量化实验。另一个细节是，当前 Koishi 使用 Cordis v3，文中给出的 Cordis v4 重构了效应、共效应语义和加载器，但两代共享核心组合模型。

这里还有一些有趣的事情，是关于这些作者的，大家肯定和我一样好奇为什么四年前的项目和目前的这篇论文会有联系。

# 6 讨论

## 6.1 系统边界：有些操作可以撤销，有些只能补偿

系统边界由“运行时能否独占修改并恢复某个位置”决定。边界内的变化进入 $\Gamma$，可以被追踪；边界外的变化被视为对内部状态没有可恢复修改。

共效应可以改变这条边界。它把某个外部位置包装成一组受控操作，并为这些操作提供逆操作，原本落在边界外的修改便可能进入 $\Gamma$，变成可以追踪和恢复的变化。边界因此是按具体位置划分的，不是按介质笼统划分：一块只有当前系统能够写入的内存可以位于边界内，被其他进程共同修改的内存则位于边界外；私有临时目录中的文件可以位于边界内，其他程序也会读写的文件则位于边界外。扩大边界可以获得更强的恢复能力，但每次访问都要经过受控接口，也会带来额外成本。

一个外部操作往往分成两步：

- **获取（acquisition）** 在边界内留下可追踪记录，例如 `open` 产生文件描述符、`malloc` 产生内存块、`fork` 产生子进程；对应的 `close`、`free`、`kill` 可以撤销这项获取。
- **发出（emission）** 把数据送到边界外，例如写入外部文件、发送网络数据。数据一旦被其他主体观察或修改，内部运行时就无法保证完整撤回。

外发结果若也要恢复，只能选择延迟提交，或者执行补偿操作，例如删除已创建文件、退回已扣款项。补偿仍可按后进先出组合，但只能恢复到业务定义的较粗等价状态，前面的形式化定理不能直接照搬。

## 6.2 服务复用：在组件层实现负载均衡和滚动更新

同一接口可以由多个组件实现。若消费者直接绑定具体提供者，更换提供者会让所有消费者重新加载。另一种方式是在中间加入**服务代理（service broker）**：消费者和后端提供者都依赖代理，代理负责把请求路由给多个实现。

这样可以支持：

- **负载均衡**：按轮询、负载或延迟分配请求；提供者注册本身是可回退效应，卸载后自动离开路由集合；
- **滚动更新**：先加载新提供者，逐步切流，等旧提供者没有进行中的请求后再卸载；
- **跨进程调用**：协调组件把远程提供者包装成本地接口，不过跨进程接口必须采用异步契约，以处理延迟和中途失败。

## 6.3 访问控制与沙箱

依赖声明可以充当能力申请：组件只能通过代理访问自己声明过的依赖，编排器也可以在加载前审查完整能力集合。拦截元数据还能进一步限制调用，例如把文件系统依赖约束到特定路径或只读模式。

这种语言层访问控制挡不住恶意代码绕过代理直接接触宿主对象。对于不可信的组件，仍需要软件故障隔离、独立运行时、沙箱进程或虚拟化容器。此时桥接组件负责把经过削减的依赖暴露给沙箱。

## 6.4 语言无关性与宿主语言要求

时间维度至少需要闭包，用来把逆操作及其恢复状态保存为值；还需要运行时加载和撤回代码的能力。托管运行时可以依靠模块注册表和垃圾回收，原生代码可以依靠动态链接；对于 WebAssembly（可移植的二进制指令格式），资源如何回收由它的宿主运行时决定。

空间维度需要：

- 在类型层表达依赖接口，例如 Haskell 类型类、Rust 特征接口（trait）、TypeScript 模块扩展；
- 在运行时拦截属性访问，例如 JavaScript `Proxy`、Python 描述符；
- 或通过注解、装饰器、过程宏等元编程工具，同时生成类型声明和访问器。

## 6.5 相互依赖与组件粒度

依赖环会让相关组件永久无法激活，因为每一方都在等待另一方先提供键。这类问题来自依赖声明本身，可以直接从声明图中提前发现，不属于运行调度偶然造成的死锁。

双向关系原则上可以拆成更细的单向绑定。例如服务器核心和访问控制核心互不依赖，再用“请求拦截”和“策略管理”两个集成组件分别依赖双方。这样能消除环并提高可选组合能力，但当 $n$ 个组件相互交互时，集成组件的数量最坏可能随 $n$ 近似平方增长，增加配置和理解成本。

工程上可以用包级捆绑、约定式连接和脚手架生成来隐藏这部分复杂度。

## 6.6 依赖类型与版本

只用键名连接依赖会产生两个问题：

- **接口漂移**：提供者升级后仍使用同一个键，但接口已经与旧消费者不兼容；
- **键冲突**：两个无关包恰好使用同一个名字，消费者可能拿到完全错误的对象。

可选方案包括：

- 把包命名空间加入键身份，直接避免无关包冲突；
- 使用包管理器的**对等依赖（peer dependency）**约束版本，Cordis 当前采用这条路线；
- 在运行时检查提供接口是否在结构上满足消费者要求。

命名空间依赖外部包注册体系，对等依赖依赖语义化版本约定且不方便多版本共存，结构兼容面对行为契约和泛型时又很复杂。统一解决仍是开放问题。

## 6.7 与语言和操作系统共同设计

若语言原生支持上下文范式，可以把上下文重新变成隐式对象，同时保持多上下文的生命周期语义。编译器还能把效应迭代器编译成单个状态机，减少每一步为逆操作创建闭包的成本，并在编译期发现依赖环和结构不兼容。

若操作系统也配合，可以把组件声明的依赖变成它能访问的全部能力，为每个组件提供隔离与拦截，并把内存、文件描述符等系统资源直接作为共效应提供。系统本来就在记录资源归属，因此可以在内核或运行时边界统一完成细粒度回收。

操作系统还可能让部分原本只能延迟提交或执行补偿的持久化操作具备真正的回退能力。例如，事务式持久化可以撤销尚未提交的写入，写时复制或不可变存储则可以通过切换指针回到更早的状态。

# 7 相关研究

## 7.1 效应、共效应与统一类型系统

这一领域已有四组相关工作。

**单子式效应系统。** ZIO、Effect-TS 和 fp-ts 把结果、错误以及计算需要的服务编码进类型，再交给运行时执行。程序需要写进对应的效应类型才能获得追踪能力。服务被撤回后，已经造成的修改仍会保留。Cordis 无需把整个程序重写进单子类型，同时要求每项效应带上逆操作。

**把代数效应理解为能力。** Effekt 用效应类型表达计算需要从上下文取得哪些能力，这和共效应依赖有相似之处。Effekt 关注同一操作如何交给不同处理器解释，并在类型层约束能力的作用域；Cordis 处理运行时的修改追踪、恢复和依赖重解析。

**可逆效应语义。** 可逆箭头和逆箭头把副作用放进可逆计算结构，整个计算的双侧逆由结构给出。Cordis 只要求每个原子效应在执行时提供单侧逆操作，整体恢复过程由运行时逐步组合。

**统一效应和共效应的分级类型。** Granule 等工作用分级模态类型同时描述计算做了什么、需要什么，检查发生在编译期，作用域也已经由程序文本确定。Cordis 把这两个概念放到运行时，已加载组件发生变化时，依赖和恢复过程也会随之调整。

## 7.2 其他编程范式

**面向上下文编程（Context-Oriented Programming, COP）** 根据位置、用户或模式激活不同层，从而改变方法分派。这里的上下文也是运行时对象，但层产生的副作用不会被追踪和撤销，激活条件也和依赖是否满足无关。Cordis 可以用共效应完成“根据全局上下文值选择实现”，无法直接表达 COP 在动态作用域中临时激活某一层的完整语义。

**面向切面编程（Aspect-Oriented Programming, AOP）** 通过切点和通知把横切逻辑织入基础程序。Cordis 用共效应承担相近的中介作用，多个组件明确声明对它的依赖，编排器可以集中调整相关行为。影响范围以组件声明的依赖为界。组件退出后，这些调整会自动撤销，依赖它的组件也会收到变化。

## 7.3 时间可组合性的四类既有路线

- **状态前向迁移。** 动态软件更新、Erlang/OTP 和常见 HMR 把旧版本状态转换给新版本，能够保留组件内部状态，但迁移逻辑需要手写。Cordis 会撤销旧效应，再从干净状态重建；需要长期保存的状态应放进生命周期更长的依赖。
- **开发者手写恢复。** 插件卸载钩子、命令模式（Command）的撤销操作（undo）、Saga 补偿事务和事件溯源，都依赖开发者补全清理逻辑。React 的效应钩子 `useEffect` 已经把效应与清理函数放在一起，但受顶层调用、同步函数和固定钩子调用顺序约束，难以自由组合控制流。
- **静态作用域回退。** 事务内存、可逆计算、线性类型、RAII 和 Rust 所有权可以自动回收资源，回退边界需要提前固定在语言结构或词法作用域中。
- **接口拦截式回收。** Nooks、shadow drivers 和 Akeso 在运行时可控接口上记录组件取得的资源，发生故障后统一释放。这类方法和可回退效应最接近，但平台只能回收预先认识的资源类型。Cordis 允许组件定义新效应，并为每项原子效应提供逆操作。

## 7.4 空间可组合性的三类既有路线

- **初始化时依赖注入。** Spring、Guice、Angular 以及 React/Vue 的上下文机制会在启动或创建组件时注入依赖。提供者在运行中被替换后，现有消费者通常不会自动停用并重新初始化。
- **按服务可用性反应。** OSGi Declarative Services、iPOJO 和 R-OSGi 会随服务的出现和消失启停组件，这是最接近响应式共效应的一类方案。不过，停用回调通常需要手写，而且只能同步执行，无法先等待消费者完成异步清理，再撤回提供者。
- **值级反应系统。** 函数式反应编程和响应式信号（Signals）在单个值变化时重新计算下游，粒度比 Cordis 更细。它们通常按照依赖图在同一轮传播中更新，避免下游同时读到新旧输入，这称为**无毛刺一致性（glitch freedom）**。Cordis 没有统一的传播轮次，处理的是组件级异步生命周期，只保证一次组件转换不会横跨两套依赖解析结果。两者可以组合，共效应内部仍然可以携带反应式值。

# 8 结论

动态组合包含两个问题：**怎样撤销组件造成的修改，以及怎样在环境变化后重新解析组件依赖**。可回退效应把原子修改和逆操作放在一起，响应式共效应根据依赖是否满足触发组件生命周期。两者共享同一个上下文，再由动态组合演算处理多个组件交错运行时的恢复和依赖协调。

Cordis 按照形式化模型实现了这些机制。`ctx.effect` 追踪效应，`ctx.set/get` 处理共效应，`fiber` 保存组件实例，`refresh/reload/unload` 驱动生命周期；声明式配置和 HMR 建立在这些原语之上。Koishi 已在四千多个社区插件组成的生态中使用这一组合模型。现有材料没有提供跨框架性能对比和开发效率实验，Koishi 当前使用的也是 Cordis v3，文中重构的 v4 还需要更多生产环境验证。

接下来可以在**自我演化 Agent Harness**中检验 Cordis。Agent 持续生成和替换 Harness 组件时，旧组件留下的修改能否完整恢复，频繁变化的依赖能否保持协调，都会直接暴露这套机制的边界。目前还没有实际系统证明 Cordis 足以支撑长期自我演化。
