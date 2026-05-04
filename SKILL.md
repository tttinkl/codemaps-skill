---
name: codemap
description: 为当前 coding project 按任务 prompt 生成 Call-Path Slice codemap（单一结构化 Markdown，v0.1.3 schema），写入 <project>/.codemaps/。产物同时供 Claude `@` 注入和本地 viewer 可视化阅读。
triggers:
  - 用户说"建 codemap / 生成 codemap / codemap this"
  - 用户说"给 X 流程做一张代码地图"
  - 用户说"帮我理解这条调用链"
---

# Codemap Generator (v0.1.3)

按任务驱动的方式为 coding project 产出精准 Call-Path Slice 代码地图。**反 "vibeslop"**：先建立共同理解再动手写代码。

产出规格完整定义在项目根的 `schema-design.md` §5；本 SKILL 仅引导生成步骤与硬约束。

**核心形态**：
- **节点三类模型**：代码节点（带 ID + code）/ 指针节点（无 ID 有 file:line）/ 分组节点（无 ID 无 code）
- **节点 ID = section-local 复合编号**：`<section-num><letter>`，如 `1a` / `1b` / `2a` / `3c`
- **扁平 H2 文档结构**：`## Overview` / `## 1. Section` / `## 2. Section` / ... / `## Narrative` / `## Notes`
- **Overview 必含主线引用链**：`[1a]→[1b]→[3c]` 至少 3 个 ID 用 `→` 串联

## 激活前提

1. **工作目录是 coding project**（有源代码、非文档仓库）
2. **用户已给出任务/入口描述**（如 "登录流程"、"/api/orders POST handler"、"WebhookProcessor 扩展点"）

若任一缺失：先补齐再进入步骤 1。

## 思考流程（6 阶段，每张 codemap 必走）

1. **解释（Interpret）** —— 把用户的入口描述映射到代码符号 / 文件 / 端点 / 事件 / 扩展点。**保留用户原始语言**作为 title 与 section 名（中文项目就用中文，不强行翻译）。模糊术语扩展为代码概念，**禁止臆造未观察到的行为**。
2. **探索（Explore）** —— 从入口出发沿真实 callable 路径展开（具体 follow paths 见步骤 2）；既追**上游数据来源**也追**下游副作用**；优先解释执行顺序与组件关系的代码路径。
3. **证据图（Evidence Graph）** —— 收集真实节点（含三类：代码 / 指针 / 分组）+ file:line + code excerpt；每个代码节点必须有 stable section-local id（`1a` / `2c` ...）；剔除不支持主线故事的低价值实现细节。
4. **分组（Group）** —— 把节点聚合成 **4–8 个 sections**（M 档；S 档 1–2 / L 档 5–12），每节描述一个 coherent flow 或 subsystem；按 execution / data / control flow 顺序排列；每节附一句 **核心包 - <package/module scope 描述>**。
5. **渲染（Render）** —— 标题从入口和已发现 scope 凝练；**Overview 必须含 ≥1 条主线引用链** (`[1a]→[1b]→[3c]`，≥3 个 ID 用 `→` 连接) + 总内联引用 ≥5 个 + 显式声明非目标；按 §5.4 schema 写嵌套 markdown bullet 树。
6. **验证（Validate）** —— 校验每条 path / line / code 真实存在；移除无证据声明；最终自问"**这张图能用来导航吗？**"——只作文档不算合格。

## 步骤

### 1. 对齐任务边界（与用户交互）

- **一句话复述**任务目标
- **确认 scope 档位**（MVP 默认 M；用户可选 S 深读单函数 / L 扩展点契约）
- **确认入口锚点**：类型（function / endpoint / event / extension-point）+ ref（符号 / HTTP 路径 / 事件名）
- **提出 slug**：从 title 派生 kebab-case，如 `user-login-flow` / `webhook-processor-extension`；若 `.codemaps/<slug>.md` 已存在则追加 `-2` / `-3`；最终 slug ≤ 50 字符
- **宣布边界**：明确告诉用户这张 codemap **不会**覆盖的内容（避免探索膨胀）

### 2. 探索调用链（迭代检索）

- **首选激活 `iterative-retrieval` skill**（若 serena_mcp 可用则优先 serena 做搜索）
- grep / glob / read 三件套追踪入口向外 & 向内各若干跳
- **沿这些真实 callable 路径展开**（按相关性，不必全列；找不到对应概念就跳过）：
  - **Imports** —— 入口文件的依赖闭包
  - **Hook / 生命周期** —— `useEffect` / `onMount` / 中间件 / 拦截器 / signal 订阅
  - **组件组合** —— 父子组件、`children` / slot、provider、HOC
  - **事件路径** —— `emit` / `on` / `dispatch` / listener / pub-sub
  - **Model / State** —— store、reducer、context、reactive 对象的**读写点**
  - **路由 / 数据 / Schema** —— route 配置、loader / resolver、ORM model、API schema
  - **样式 / 设计系统** —— theme、design token、variant、material
  - **设计时 vs 运行时变体** —— feature flag、env 分支、build-time 注入
  - **上游数据来源** —— 参数从哪里来、context 如何注入、外部输入边界
  - **下游副作用** —— DB write / 网络出 / 日志 / 队列入 / 文件 IO
- **优先级**（高→低，配额吃紧时按此剪枝）：
  入口 > 分支转折 > 失败处理 > 跨层跳点（同步↔异步 / 进程间 / 网络 IO）> 上游数据源 > 下游副作用 > 平凡直通

**防盲猜硬约束（不可违反）**：
- 所有 file:line 必须来自**真实文件读取**（Read 工具），禁止凭训练语料臆造符号名或行号
- 每个代码节点的 code 必须是**源文件真实连续行**，允许用 `// ...` 省略标记跳过无关内部分支，**禁止改写/拼接/臆造代码**

### 3. 构造文档（v0.1.3 三类节点模型）

#### 3.1 节点三类模型

| 类型 | 何时使用 | 行格式（看清楚精确空格与缩进！） |
|---|---|---|
| **代码节点** | 关键代码段（入口 / 分支转折 / 失败处理 / 跨层跳点 / 重要调用） | `- **<id>** <动作 label> — \`<file:line>\`` 紧跟 fenced code block |
| **指针节点** | 上下文锚点：函数入口签名、组件挂载点、父级文件位置（让用户跳源） | `- **<label>** — \`<file:line>\`` |
| **分组节点** | 纯结构性标签：循环描述、分支组、子系统标题 | `- **<label>**` （可选 `— \`<file:line>\``） |

**节点 label 风格**：
- **中文动作描述**（"获取路由数据" / "渲染 Nav 子组件" / "用户点击菜单项"），动词开头
- 函数符号名只出现在 fenced code block 里，**不写在 label 中**
- 不带 `(kind:name)` 标注（v0.1.2 已废弃）

**ID 编号规则**：
- 形式：`<section-num><letter>`，如 `1a` / `1b` / `2c`
- section-num 必须等于所在 section 的 H2 编号（`1a` 必须在 `## 1. ...` 内）
- letter：a-z 在 section 内顺次（**软规则：建议连续，允许 gap，parser 不报错；超过 26 个节点拆 section，不引入 `aa` / `ab`**）
- 同一 section 内 letter 不能重复（parser 报错）
- 不同 section 可复用 letter（`1a` 与 `2a` 互不冲突）
- 每 section ≥ 1 个代码节点

#### 3.2 文档骨架（扁平 H2）

```markdown
---
slug / scope / kind / title / entry / created_at / last_touched / tags / history
---

## Overview              # 元 section：3-5 句 + ≥1 条主线引用链 + 显式非目标
## 1. <Section Title>    # flow section（M 档 4-8 个）
## 2. <Section Title>
...
## N. <Section Title>
## Narrative             # 元 section（可选）
## Notes                 # 元 section（可选）
```

**严禁**：`## Nodes` / `## Edges` / `## Node Details` / `## Flow` 顶级章节（v0.1.2 已废弃，parser 直接报错）。

#### 3.3 单 section 内部结构（**字面 verbatim 模板，照着拷贝缩进**）

```markdown
## 1. RouteNav 组件渲染流程
> 核心包 - RouteNav 主组件如何包装 Nav 子组件并传递路由数据

- **RouteNav 主组件** — `RouteNav.tsx:8`
  - **1a** 获取路由数据 — `RouteNav.tsx:16`
    ```ts
    const routes = useRoutes() || props.routes;
    ```
  - **1b** 渲染 Nav 子组件 — `RouteNav.tsx:17`
    ```tsx
    return <Nav key={props.theme} {...props} routes={routes} />;
    ```
- **Nav 子组件** — `nav/main/index.tsx:1`
  - **1c** Nav 组件接收路由 — `nav/main/index.tsx:24`
    ```ts
    const routes = props.routes ?? useRoutes();
    ```
  - **渲染 Ant Design Menu**
    - **routes.map() 遍历路由** — `nav/main/index.tsx:65`
      - **1f** 渲染主菜单项 — `nav/main/index.tsx:70`
        ```tsx
        <Menu.SubMenu key={route.key} title={route.name}>
        ```
```

**关键缩进规则**（CommonMark 要求严格）：
- 一级 list item：`- ` 后接内容（缩进 0）
- 二级 list item（嵌套）：`  - ` 缩进 **2 空格**
- list item 下的 fenced code block：缩进 **4 空格 × depth + 4** 与 list 内容对齐（如二级 list 内的代码块缩进 4 空格）
- 嵌套深度 ≤ 4 层

**fenced code block 长度**：1–5 行真实代码；超长函数仅摘签名 + 关键调用行，用 `// ...` 省略中段。

#### 3.4 Overview 内联引用规则

Overview 段落 3–5 句话；**必须**满足：

1. **≥1 条主线引用链**：`[xa]→[xb]→[xc]`，至少 3 个 ID 用 `→` 连接，描述端到端主路径
2. **总内联引用 ≥5 个**：含主线 + 副线引用（如失败路径、设计器路径）
3. **`[<id>]` 形式引用代码节点**（`[1a]`，无 `(kind:name)`）；可选 `[§<num>]` 引用 Flow section
4. **显式声明非目标**：用 `**非目标**：...` 或同等清晰的句式
5. **用户原始语言**

**示例**：

```markdown
## Overview
本代码地图展示了 RouteNav 导航组件的完整实现，包括组件渲染流程、Router 状态管理、用户交互处理、Schema 解析、设计器集成和样式系统。关键流程包括：路由数据获取 [1a] → Nav 渲染 [1b] → 菜单点击 [3c] → History 更新 [3g] → 路由状态计算 [2g]。设计器环境通过 TreeNode 过滤 [5b] 实现独立的路由管理，样式系统通过素材配置 [6a] 和动态计算 [6e] 实现主题化。

**非目标**：不覆盖 SSR 渲染路径、单元测试代码、历史 v1 实现。
```

#### 3.5 Section 标题与 scope blockquote

- 标题格式：`## <序号>. <Section Title>`（数字 + 点 + 空格 + 中文标题）
- 标题下一行 blockquote：`> 核心包 - <一句话 package / module scope 描述>`
- section 排序：execution / data / control flow 顺序；通常"入口 → 主路径 → 副作用 → 失败回收"是好默认

### 4. 跨 section 关系表达

无独立 Edges section。需要表达跨节跳转时：
- **首选**：在 Overview 主线引用链中体现（`[1a]→[1b]→[3c]→[2g]` 自然贯穿多 section）
- **次选**：在 `## Narrative` 自由段落里散文描述
- **不要**在 list 节点 label 末尾写结构化 `→ [2g]` —— parser 不抽，纯文字也容易误导

### 5. 写盘

**两步原子操作**：

1. 写 `<project>/.codemaps/<slug>.md`（frontmatter + Overview + N flow sections + Narrative + Notes）
2. 更新 `<project>/.codemaps/index.md` —— **必须 Read-first → 内存拼接 → 整体覆盖写**，禁止流式 append（防中断损坏；index 条目含 slug / scope / title / tags / last_touched）

**Frontmatter 日期字段写法规范**：
- **统一用 `YYYY-MM-DD` 纯日期格式，不加引号、不写时分秒**
- 适用字段：`created_at` / `last_touched` / `history.introduced_at` / `history.last_refactor`
- 示例：`created_at: 2026-04-24`（✅）；`created_at: '2026-04-24'`（✅ 兼容但啰嗦）；`created_at: 2026-04-24T01:15:00Z`（❌ 会被 parser 截断）

**字符串字段引号规则**（避免 YAML 陷阱）：
- 含冒号 `:` / 井号 `#` / 方括号 `[]` 的标题和描述**必须单引号**（如 `title: 'POST /api/orders: 下单流程'`）
- 纯拉丁字母+中文+连字符的简单字符串可省引号

**不自动写 `.gitignore`**：`.codemaps/` 默认跟踪，作为团队共享的任务上下文档案。

### 6. 自检清单

生成完毕**自行**核对（全部 ✅ 才向用户交付）：

**Frontmatter**：
- [ ] 必填字段齐全：slug / scope / kind / title / entry / created_at（last_touched / tags / history 可选）
- [ ] kind 值为 `call-path-slice`

**Document 结构**：
- [ ] 顶级 section 仅 `## Overview` / `## <num>. <title>` / `## Narrative` / `## Notes`，**无 `## Nodes` / `## Edges` / `## Node Details` / `## Flow`**
- [ ] Flow section 数 4–8（M 档；S 档 1–2 / L 档 5–12）
- [ ] Section 编号从 1 顺次连续（1, 2, 3, ... 无跳号）
- [ ] 每 section 标题格式 `## <数字>. <中文标题>`
- [ ] 每 section 标题下一行有 `> 核心包 - <描述>` blockquote

**Overview**：
- [ ] 含 ≥1 条 `[xa]→[xb]→[xc]` 主线引用链（≥3 个 ID 用 `→` 连接）
- [ ] 总内联引用 `[<id>]` ≥5 个（可包含 `[§<num>]`）
- [ ] 所有引用的 ID 都在 Flow section 里真实存在
- [ ] 显式声明非目标 / out-of-scope

**节点（每 section 内）**：
- [ ] 至少 1 个代码节点（带 `**<id>**` + fenced code）
- [ ] 代码节点 ID 前缀等于所在 section 编号（`1a` 在 `## 1.` 内）
- [ ] 同 section 内 ID letter 不重复
- [ ] 每代码节点必带 `**<id>** <label> — \`<file:line>\`` + fenced code（1–5 行）
- [ ] 每指针节点必带 `**<label>** — \`<file:line>\``
- [ ] 分组节点纯 `**<label>**`（可选 file:line）
- [ ] 节点 label 用**中文动作描述**（动词开头，符号名不进 label）
- [ ] 嵌套深度 ≤ 4 层
- [ ] 二级 list 缩进 2 空格；fenced code 缩进 4 空格 × depth

**用户语言**：
- [ ] Title / Overview / section 标题 / 节点 label / scope 用用户原始语言（中文项目用中文）
- [ ] 文件路径 / 符号名 / 行号原文保留

**真实性**：
- [ ] 每条 file:line 来自真实 Read（用 Read 工具读过对应文件）
- [ ] 每段 fenced code 与源文件对应行**逐字一致**（除 `// ...` 省略外）

**index.md**：
- [ ] 已追加本 codemap 条目，其他条目保留

### 7. 反馈给用户

输出格式：

```
✅ 已生成 codemap: <slug>
  - 路径: .codemaps/<slug>.md (<代码节点数> code nodes 跨 <section 数> sections)
  - 入口: <entry.ref>
  - 引用字符串: @.codemaps/<slug>.md

启动 viewer 可视化:
  cd <project-root> && codemap-viewer
  # → http://localhost:4676
```

## 硬约束（不可违反）

1. **禁止臆造 path:line**：所有 file:line 来自真实 Read，禁止根据函数名猜测行号
2. **禁止改写代码**：fenced code block 必须是源文件真实连续行（允许 `// ...` 省略），禁止拼接 / 改写 / 生成式重构
3. **代码节点 ≤ 5 行**：超长函数仅摘签名 + 关键调用行
4. **节点 ID = section-local 复合编号** `<num><letter>`：禁止全局 `Nx` 形式
5. **代码节点必带 fenced code**：1–5 行；指针节点必带 file:line（无 code）；分组节点仅 label（可选 file:line）
6. **Overview 必含主线引用链 + 总内联引用 ≥5 + 显式非目标**
7. **Flow section 4–8 个 + 编号 1 起顺次连续**：M 档；S 档 1–2 / L 档 5–12
8. **每 section ≥ 1 代码节点 + 必带 `> 核心包 - ...` scope blockquote**
9. **嵌套深度 ≤ 4 层**：超过提议拆 section
10. **节点 label 用中文动作描述**：符号名仅在 code block，不写在 label
11. **保留用户语言**：title / Overview / section 标题 / 节点 label 用用户输入时的语言；只有代码符号名 / file path / 行号保持原文
12. **不允许 `## Nodes` / `## Edges` / `## Node Details` / `## Flow` 顶级章节**（v0.1.2 已废弃；parser 报错）
13. **index.md 必须 read-first 整体覆盖写**：禁止流式 append
14. **日期字段统一 `YYYY-MM-DD`**：不加引号、不写时分秒

## 非目标

- **不做**全系统拓扑 / 架构俯瞰 / 重构用依赖全景图（那是其它工具的职责）
- **不修改**源代码
- **不运行** git commit / push 等改变仓库状态的命令
- **单次调用只产一个 codemap**；多任务分多次调用
- **MVP 不做**漂移检测（excerpt_sha 校验延后到 v0.2）

## 相关

- Schema 规格：项目根 `schema-design.md` §5
- 实施方案：项目根 `IMPLEMENTATION.md`
- Viewer 源码：`viewer/`（Hono + HTML 嵌套树 + Prism）
- Parser 源码：`packages/parser/`（mdast-based，60+ tests）
