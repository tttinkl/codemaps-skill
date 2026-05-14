---
name: codemap
description: 按任务/流程为当前 coding project 生成可导航的代码地图——聚焦调用链切片 + 真实代码片段，写入 `.codemaps/<slug>.md`。既供 Claude `@` 引用，也作为本地 viewer 的可视化输入。
triggers:
  - 建 codemap / 生成 codemap / codemap this
  - 给 X 流程/接口/事件做一张代码地图
  - 把这条调用链整理成可复用的代码地图
---

> English version: [SKILL.md](./SKILL.md)

# Codemap Generator

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

1. **解释（Interpret）** —— 把用户的入口描述映射到代码符号 / 文件 / 端点 / 事件 / 扩展点。**保留用户原始语言**作为 title 与 section 名，不强行翻译。模糊术语扩展为代码概念，**禁止臆造未观察到的行为**。
2. **探索（Explore）** —— 从入口出发沿真实 callable 路径展开（具体 follow paths 见步骤 2）；既追**上游数据来源**也追**下游副作用**；优先解释执行顺序与组件关系的代码路径。
3. **证据图（Evidence Graph）** —— 收集真实节点（含三类：代码 / 指针 / 分组）+ file:line + code excerpt；每个代码节点必须有 stable section-local id（`1a` / `2c` ...）；剔除不支持主线故事的低价值实现细节。
4. **分组（Group）** —— 把节点聚合成 **4–8 个 sections**，每节描述一个 coherent flow 或 subsystem；按 execution / data / control flow 顺序排列；每节附一句 **核心包 - <package/module scope 描述>**。
5. **渲染（Render）** —— 标题从入口和已发现 scope 凝练；**Overview 必须含 ≥1 条主线引用链** (`[1a]→[1b]→[3c]`，≥3 个 ID 用 `→` 连接) + 总内联引用 ≥5 个 + 显式声明非目标；按 §5.4 schema 写嵌套 markdown bullet 树。
6. **验证（Validate）** —— 校验每条 path / line / code 真实存在；移除无证据声明；最终自问"**这张图能用来导航吗？**"——只作文档不算合格。

## 步骤

### 1. 对齐任务边界（与用户交互）

- **一句话复述**任务目标
- **确认入口锚点**：类型（function / endpoint / event / extension-point）+ ref（符号 / HTTP 路径 / 事件名）
- **提出 slug**：从 title 派生 kebab-case，如 `user-login-flow` / `webhook-processor-extension`；若 `.codemaps/<slug>.md` 已存在则追加 `-2` / `-3`；最终 slug ≤ 50 字符
- **宣布边界**：明确告诉用户这张 codemap **不会**覆盖的内容（避免探索膨胀）

### 2. 探索调用链（迭代检索）

- **检索策略**：迭代检索 —— 粗扫候选 → 评分相关性 → 收敛后再读细节，避免一次性吞整个文件
- **工具优先级**：优先使用 host 环境提供的语义检索能力（LSP-aware MCP / symbol index / semantic search 等）；不可用时用 grep / glob / Read 三件套追踪入口向外 & 向内各若干跳
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
- **优先级**
  入口 > 分支转折 > 失败处理 > 跨层跳点（同步↔异步 / 进程间 / 网络 IO）> 上游数据源 > 下游副作用 > 平凡直通

**停止条件**（何时停止探索 —— 替代 v0.1.3 的档位规则）：
- 端到端主路径已抵达终点失败处理 / 输出副作用
- 入口的每个分支都已被至少一条路径覆盖
- 继续探索只会加入平凡直通节点（helper、wrapper、formatter 等不改变故事的节点）

**规模指引**（建议性，parser 不强制）：
- 大多数 codemap 落在 8–40 个代码节点；目标是用最小的图覆盖任务端到端
- 接近 50 节点时停下重新审视任务边界 —— 若"一个任务"实际由多个子任务组成，请用户把它拆为多个具体任务，每个子任务单独 invoke skill 生成 codemap（坚守"一个任务一个 codemap"原则；不在工具层面把单份 codemap 拆成多份文件）
- 接近 80 节点时任务范围多半过宽 —— 必须与用户重新缩限

**防盲猜硬约束（不可违反）**：
- 所有 file:line 必须来自**真实文件读取**（Read 工具），禁止凭训练语料臆造符号名或行号
- 每个代码节点的 code 必须是**源文件真实连续行**，允许用 `// ...` 省略标记跳过无关内部分支，**禁止改写/拼接/臆造代码**

### 3. 构造文档

#### 3.1 节点三类模型

| 类型 | 何时使用 | 行格式（看清楚精确空格与缩进！） |
|---|---|---|
| **代码节点** | 关键代码段（入口 / 分支转折 / 失败处理 / 跨层跳点 / 重要调用） | `- **<id>** <动作 label> — \`<file:line>\`` 紧跟 fenced code block |
| **指针节点** | 上下文锚点：函数入口签名、组件挂载点、父级文件位置（让用户跳源） | `- **<label>** — \`<file:line>\`` |
| **分组节点** | 纯结构性标签：循环描述、分支组、子系统标题 | `- **<label>**` （可选 `— \`<file:line>\``） |

**节点 label 风格**：
- **用户语言的动作描述**（"获取路由数据" / "渲染 Nav 子组件" / "用户点击菜单项"），动词开头
- 函数符号名只出现在 fenced code block 里，**不写在 label 中**

**ID 编号规则**：
- 形式：`<section-num><letter>`，如 `1a` / `1b` / `2c`
- section-num 必须等于所在 section 的 H2 编号（`1a` 必须在 `## 1. ...` 内）
- letter：a-z 在 section 内顺次（**软规则：建议连续，允许 gap，parser 不报错；超过 26 个节点拆 section，不引入 `aa` / `ab`**）
- 同一 section 内 letter 不能重复（parser 报错）
- 不同 section 可复用 letter（`1a` 与 `2a` 互不冲突）
- 每 section ≥ 1 个代码节点

#### 3.2 文档骨架与 Frontmatter 字段

**整体结构（扁平 H2）**：

```markdown
---
<frontmatter 字段，见下表>
---

## Overview              # 元 section：3-5 句 + ≥1 条主线引用链 + 显式非目标
## 1. <Section Title>    # flow section（4-8 个）
## 2. <Section Title>
...
## N. <Section Title>
## Narrative             # 元 section（可选）
## Notes                 # 元 section（可选）
```

**Frontmatter 字段规范**：

| 字段 | 必填 | 类型 / 取值 | 说明 |
|---|---|---|---|
| `slug` | ✅ | kebab-case 字符串，≤50 字符 | 文件名 = `<slug>.md`；与 `.codemaps/<slug>.md` 路径强绑定，冲突时追加 `-2` / `-3` |
| `kind` | ✅ | 固定值 `call-path-slice` | 文档类型 discriminator；当前唯一合法值，其他值 parser 拒绝 |
| `title` | ✅ | 字符串，用户原始语言 | 一句话任务描述；含 `:` / `#` / `[]` 必须用单引号包裹（见 §5 引号规则） |
| `entry.type` | ✅ | `function` / `endpoint` / `event` / `extension-point` 枚举 | 入口锚点类型 |
| `entry.ref` | ✅ | 字符串 | 定位入口的引用——函数符号（建议含 `file:line`）/ HTTP 路径 / 事件名 / 扩展点 ID |
| `created_at` | ✅ | `YYYY-MM-DD` | 首次生成日期；不加引号、不写时分秒（见 §5 日期规则） |
| `last_touched` | 可选 | `YYYY-MM-DD` | 最近一次更新日期；建议每次回填同步 |
| `tags` | 可选 | 字符串数组 | 用于 index.md 索引与 viewer 过滤；如 `[auth, login]` |
| `history.introduced_at` | 可选 | commit sha 或 `YYYY-MM-DD` | 流程引入时间，便于回溯历史 |
| `history.last_refactor` | 可选 | commit sha 或 `YYYY-MM-DD` | 最近一次重构时间 |

**完整示例**：

```yaml
---
slug: routenav-navigation-structure
kind: call-path-slice
title: RouteNav 导航组件结构：路由处理与二级菜单系统
entry:
  type: function
  ref: RouteNav (src/RouteNav.tsx:8)
created_at: 2026-05-04
last_touched: 2026-05-04
tags: [routenav, navigation, routing]
history:
  introduced_at: 2024-11-12
  last_refactor: 2026-04-30
---
```

> 字段写盘相关的 YAML 引号规则与日期格式硬约束见 §5 「写盘」。

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

- 标题格式：`## <序号>. <Section Title>`（数字 + 点 + 空格 + 标题，用户语言）
- 标题下一行 blockquote：`> 核心包 - <一句话 package / module scope 描述>`
- section 排序：execution / data / control flow 顺序；通常"入口 → 主路径 → 副作用 → 失败回收"是好默认

### 4. 跨 section 关系表达

无独立 Edges section。需要表达跨节跳转时：
- **首选**：在 Overview 主线引用链中体现（`[1a]→[1b]→[3c]→[2g]` 自然贯穿多 section）
- **次选**：在 `## Narrative` 自由段落里散文描述
- **不要**在 list 节点 label 末尾写结构化 `→ [2g]` —— parser 不抽，纯文字也容易误导

### 5. 写盘

#### 5.0 写前 parser 校验（必做，含修复回路）

§3 列出的所有结构性规则（frontmatter 必填字段、顶级 section 名、flow section
连续编号、代码节点 id letter 唯一、Overview 引用必须指向真实代码节点 ……）
都由本仓 viewer 自带的 parser **机器校验**。§6 自检是必要补充但不充分——
**先**用校验器跑一遍内存里的 markdown，**通过后**才能落盘。

**调用方式**（用 stdin 推入，失败时不会留下脏文件）：

```bash
# `-` 表示从 stdin 读；npx 拉取已发布的 binary，使本流程在任意未本地链接的项目都能跑
printf '%s' "$GENERATED_MARKDOWN" | npx -y @tttinkl/codemaps-viewer validate -
```

若 `codemap-viewer` 已全局 link，可省略 `npx` 前缀。

**结果解读**：

- **退出码 0** → 输出 `✓ valid: <stdin>`，进入 5.1。
- **退出码 1** → schema 不合法。stderr 输出单条诊断，格式如下：
  ```
  <stdin>:<line>:<col> - error: <message>

  <line> | <出错那一行的源文本>
         | <指向 column 的 ^ 标记>

  hint: <修复建议>
  ```
  按 message + hint 修订内存里的 markdown，再次校验。
- **退出码 2** → 校验器自身 IO/用法问题（文件不可读、参数缺失等），上报用户，
  **不要**盲目重试。

**修复回路上限：3 次**。第 3 次仍失败则**停止**：把最后一条诊断原文展示给用户，
请求人工指引。**禁止落盘任何 parser 校验失败的文件** —— 坏 codemap 会污染
viewer、VS Code 扩展，以及下游的 `@`-引用。

#### 5.1 两步原子写盘（仅在 5.0 通过后）

1. 写 `<project>/.codemaps/<slug>.md`（frontmatter + Overview + N flow sections + Narrative + Notes）
2. 更新 `<project>/.codemaps/index.md` —— **必须 Read-first → 内存拼接 → 整体覆盖写**，禁止流式 append（防中断损坏；index 条目含 slug / title / tags / last_touched）

#### 5.2 回填新鲜度 sha（必做 — v0.2）

5.1 写盘成功后，**立即**调用 backfill CLI 在刚写的 codemap 上填入 v0.2
新鲜度字段（`files:` map + `last_verified:`）：

```bash
# `<slug>` 即文件名去掉 .md 后的部分
# `--project <projectRoot>` 默认是 cwd；codemap 写在非 cwd 下的项目时显式传入
npx -y @tttinkl/codemaps-viewer backfill <slug>
```

`codemap-viewer` 已全局 link 时可省 `npx` 前缀。

**硬规则 — 不允许自己手写 `files:` 或 `last_verified:`**。sha 生成是 CLI
的工作：
- 读取 codemap 中每个 `pathRef.relPath`
- 读取对应源文件内容
- 算 16 位 sha256（normalize 过的）
- 写回 frontmatter

手写的 sha 必然算法不匹配，会让所有新鲜度检查误报"stale"。永远调 CLI。

**结果解读**：
- `✓ <slug>  — N file(s), wrote sha for K, M missing` → 完成。codemap
  现在符合 v0.2，viewer 与 VS Code 扩展会显示 ✓ Fresh。
- `✗ <slug>  — parse error: <message>` → 刚写的 codemap 解析不过。理论
  上不应发生（§5.0 已校验过）；如果发生，原文回报给用户排查。
- `M missing > 0` → codemap 中有些 `relPath` 指向不存在的文件。修正路径
  （拼写错 / projectRoot 错）后重跑，或接受让新鲜度 UI 标记为 missing。

**退出码**：0 OK；1 部分失败；2 IO/usage 错（视为停止条件）。

**Frontmatter 日期字段写法规范**：
- **统一用 `YYYY-MM-DD` 纯日期格式，不加引号、不写时分秒**
- 适用字段：`created_at` / `last_touched` / `history.introduced_at` / `history.last_refactor`
- 示例：`created_at: 2026-04-24`（✅）；`created_at: '2026-04-24'`（✅ 兼容但啰嗦）；`created_at: 2026-04-24T01:15:00Z`（❌ 会被 parser 截断）

**字符串字段引号规则**（避免 YAML 陷阱）：
- 含冒号 `:` / 井号 `#` / 方括号 `[]` 的标题和描述**必须单引号**（如 `title: 'POST /api/orders: 下单流程'`）
- 纯拉丁字母+中文+连字符的简单字符串可省引号

**不自动写 `.gitignore`**：`.codemaps/` 默认跟踪，作为团队共享的任务上下文档案。

### 6. 自检清单

下面清单中所有**结构性**条目（frontmatter 形态、section 命名与连续编号、ID
letter 唯一、Overview 引用绑定 ……）已在 §5.0 由 parser **机器校验**。剩余条目
——用户语言保留、代码真实性、label 风格等——是 parser **无法**机械检查的部分，
由你负责。

生成完毕**自行**核对（全部 ✅ 才向用户交付）：

**Frontmatter**：
- [ ] 必填字段齐全：slug / kind / title / entry / created_at（scope / last_touched / tags / history 可选）
- [ ] kind 值为 `call-path-slice`

**Document 结构**：
- [ ] 顶级 section 仅 `## Overview` / `## <num>. <title>` / `## Narrative` / `## Notes`，**无 `## Nodes` / `## Edges` / `## Node Details` / `## Flow`**
- [ ] Flow section 数 4–8
- [ ] Section 编号从 1 顺次连续（1, 2, 3, ... 无跳号）
- [ ] 每 section 标题格式 `## <数字>. <标题>`
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
- [ ] 节点 label 用**用户语言的动作描述**（动词开头，符号名不进 label）
- [ ] 嵌套深度 ≤ 4 层
- [ ] 二级 list 缩进 2 空格；fenced code 缩进 4 空格 × depth

**用户语言**：
- [ ] Title / Overview / section 标题 / 节点 label 用用户原始语言
- [ ] 文件路径 / 符号名 / 行号原文保留

**真实性**：
- [ ] 每条 file:line 来自真实 Read（用 Read 工具读过对应文件）
- [ ] 每段 fenced code 与源文件对应行**逐字一致**（除 `// ...` 省略外）

**index.md**：
- [ ] 已追加本 codemap 条目，其他条目保留

**新鲜度回填（v0.2）**：
- [ ] 已在 §5.1 写盘后跑 `codemap-viewer backfill <slug>`
- [ ] CLI 反馈 `✓ <slug>  — N file(s), wrote sha for K`
- [ ] frontmatter 中没有手写的 `files:` / `last_verified:`

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

安装 codemap-viewer 全局:
  pnpm add -g @tttinkl/codemaps-viewer
```

## 硬约束（不可违反）

1. **禁止臆造 path:line**：所有 file:line 来自真实 Read，禁止根据函数名猜测行号
2. **禁止改写代码**：fenced code block 必须是源文件真实连续行（允许 `// ...` 省略），禁止拼接 / 改写 / 生成式重构
3. **代码节点 ≤ 5 行**：超长函数仅摘签名 + 关键调用行
4. **节点 ID = section-local 复合编号** `<num><letter>`：禁止全局 `Nx` 形式
5. **代码节点必带 fenced code**：1–5 行；指针节点必带 file:line（无 code）；分组节点仅 label（可选 file:line）
6. **Overview 必含主线引用链 + 总内联引用 ≥5 + 显式非目标**
7. **Flow section 4–8 个 + 编号 1 起顺次连续**
8. **每 section ≥ 1 代码节点 + 必带 `> 核心包 - ...` scope blockquote**
9. **嵌套深度 ≤ 4 层**：超过提议拆 section
10. **节点 label 用用户语言的动作描述**：符号名仅在 code block，不写在 label
11. **保留用户语言**：title / Overview / section 标题 / 节点 label 用用户输入时的语言；只有代码符号名 / file path / 行号保持原文
13. **index.md 必须 read-first 整体覆盖写**：禁止流式 append
14. **日期字段统一 `YYYY-MM-DD`**：不加引号、不写时分秒
15. **写盘前必须通过 parser 校验**：用 `codemap-viewer validate -` 校验内存中的 markdown；失败时按 hint 修订（≤3 次）后重新校验；任何未通过校验的文件**禁止**落盘
16. **写盘后必须回填 sha（v0.2）**：codemap 写完后，必须运行 `codemap-viewer backfill <slug>` 在 frontmatter 填入 `files:` sha map + `last_verified:` —— **不允许自己手写 `files:` 或 `last_verified:`**（sha 生成是 CLI 的工作；手写值算法不匹配，会让所有新鲜度检查误报"stale"）

## 非目标

- **不做**全系统拓扑 / 架构俯瞰 / 重构用依赖全景图
- **不修改**源代码
- **不运行** git commit / push 等改变仓库状态的命令
- **单次调用只产一个 codemap**；多任务分多次调用
