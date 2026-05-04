---
name: codemap
description: 为当前 coding project 按任务 prompt 生成 Call-Path Slice codemap（单一结构化 Markdown），写入 <project>/.codemaps/。产物同时供 Claude `@` 注入和本地 viewer 可视化阅读。
triggers:
  - 用户说"建 codemap / 生成 codemap / codemap this"
  - 用户说"给 X 流程做一张代码地图"
  - 用户说"帮我理解这条调用链"
---

# Codemap Generator

按任务驱动的方式为 coding project 产出精准 Call-Path Slice 代码地图。**反 "vibeslop"**：先建立共同理解再动手写代码。

产出规格完整定义在 `~/.claude/codemaps/schema-design.md`（frontmatter + 7 sections + S/M/L scope）；MVP 仅实现 **M 档**（端到端调用链，节点 15–60）。

## 激活前提

激活前请确认：

1. **工作目录是 coding project**（有源代码、非文档仓库）
2. **用户已给出任务/入口描述**（如 "登录流程"、"/api/orders POST handler"、"WebhookProcessor 扩展点"）

若任一缺失：先补齐再进入步骤 1。

## 思考流程（6 阶段，每张 codemap 必走）

1. **解释（Interpret）** —— 把用户的入口描述映射到代码符号 / 文件 / 端点 / 事件 / 扩展点。**保留用户原始语言**作为 title 与 section 名（中文项目就用中文，不强行翻译）。模糊术语扩展为代码概念，**禁止臆造未观察到的行为**。
2. **探索（Explore）** —— 从入口出发沿真实 callable 路径展开（具体 follow paths 见步骤 2）；既追**上游数据来源**也追**下游副作用**；优先解释执行顺序与组件关系的代码路径。
3. **证据图（Evidence Graph）** —— 每个节点必须有 stable id (Nx) / 简短 label / `path:line` / 关键代码片段；**节点间关系通过 Edges section 显式记录**（不靠 Flow 嵌套表达父子）；剔除不支持主线故事的低价值实现细节，**只留支撑主线的节点**。
4. **分组（Group）** —— 把节点聚合成 **4–8 个 sections**（M 档；S 档 1–2 / L 档 5–12），每节描述一个 coherent flow 或 subsystem；按 execution / data / control flow 顺序排列；每节附一句 package / module scope。
5. **渲染（Render）** —— 标题从入口和已发现 scope 凝练；**Overview 必须含 ≥2 个 `[Nx]` 内联引用**关键节点（可选 `[§N]` 引用 Flow section）；Flow 按 section 嵌套树状叙述；每个 leaf 链回精确源位置。
6. **验证（Validate）** —— 校验每条 path / line / snippet 真实存在；移除无证据声明；最终自问"**这张图能用来导航吗？**"——只作文档不算合格。

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
- **Token 预算约束**：M 档节点数逼近 **60** 即停止扩展，按上述优先级剪枝

**防盲猜硬约束（不可违反）**：
- 所有 Node 的 `path:line` 必须来自**真实文件读取**（Read 工具），禁止凭训练语料臆造符号名或行号
- Edge 必须能在代码中找到**具体调用点**（`foo()` 调用 `bar()` 的那一行）；无法定位的 edge 不要写
- 每个 excerpt 必须是**源文件真实连续行**，允许用 `// ...` 省略标记跳过无关内部分支，**禁止改写/拼接/臆造代码**

### 3. 构造 Nodes / Edges / Flow

按 `schema-design.md` §5.1 的 YAML-ish 多行 bullet 格式：

```yaml
## Nodes
- id: N1
  name: login(req, res)
  path: src/auth/login.ts:42-58
  kind: handler          # function | class | middleware | handler | hook | external | branch
  role: HTTP 入口，校验凭证后分派 session
  notes: 含有 rate-limit feature flag（可选）
  excerpt_lang: ts       # 有 excerpt 时必填
  excerpt_lines: "42-58" # 与 path 的行号区间对齐

## Edges
- from: N1
  to: N2
  kind: calls            # calls | emits | handles | extends | reads | writes
  when: 凭证通过         # 可选：触发条件 / 分支
```

**kind 词表**：`function / class / middleware / handler / hook / external / branch`。
`branch` 专门用于"状态机分支 / 逻辑节点"（无独立 callable 目标，如 `line-loop` / `close-frontmatter`）；
这类节点通常没有 path:line，写"同一函数内的逻辑段 + 所在行号"即可。

#### 3.1 Flow 节点引用必须带标注（首次出现）

Flow 是人类叙述 + AI 识别两用章节，裸 `Nx` 不够 —— 需要读者回跳 Nodes section 才能知道 N1 是什么。
**规则**：同一 Nx 在 Flow 里**首次出现**必须写成 `Nx(kind:name)`，同段后续复述可只写 `Nx`。

```markdown
## Flow
1. **N1(fn:main)** 启动 → 调用 **N2(fn:parseInput)**。
2. **N2** 内部进入 **N3(br:line-loop)** 状态机，每行触发 **N4(ext:fs.readFileSync)**。
3. 返回路径：**N2** → **N1**。
```

**kind 缩写表**（Flow 里优先用缩写保持密度；parser 和全名都接受）：

| 全名        | 缩写   | 适用                              |
|-------------|--------|-----------------------------------|
| function    | fn     | 项目内函数                        |
| class       | cls    | 类                                |
| middleware  | mw     | 中间件 / 拦截器                   |
| handler     | hdlr   | 路由 handler                      |
| hook        | hook   | 生命周期 / React hook             |
| external    | ext    | 第三方库 / Node API / 系统调用    |
| branch      | br     | 状态机分支 / 逻辑节点             |

**Node Details 的 `### Nx` 标题也建议带同样标注**（便于搜索对齐）：`### N1(fn:main) — 入口`。

**parser 校验**：Flow 里的 kind（归一化后）必须与 Nodes section 对应项的 kind 一致；不一致直接报错。

#### 3.2 Flow 多 section 结构（必须）

Flow **不是单一线性叙述**，而是 **4–8 个独立 section**，每节解释一个 coherent flow 或 subsystem。这是 codemap 与"贴满路径的笔记"的关键区别。

**结构**：
- 每节用 `### <序号>. <Section Title>` 起头（序号是阿拉伯数字 `1` / `2` / `3` ...，**不是 `N1` 节点编号**——避免与 Node Details 的 `### Nx — <name>` 冲突）；标题用用户语言
- 标题下一行用 blockquote `> ` 写一句 **package / module scope**（这节属于哪个层 / 模块 / 文件夹）
- 然后是该节内的有序节点叙述（用 `Nx(kind:name)` 引用 Nodes，遵循 §3.1 NodeRef 规则）
- 节内允许嵌套（`  - ` 二级列表）描述分支与并列调用

**示例**：

```markdown
## Flow

### 1. HTTP 入口与凭证抽取
> src/auth/* — 接收 POST /login 并解析 body 中的 credential

1. **N1(hdlr:loginHandler)** 接收请求 → 调用 **N2(mw:authMiddleware)** 抽取 JWT
2. 成功 → 转发 N3；失败 → 冒泡至 N5

### 2. 凭证校验
> src/auth/validate.ts — bcrypt 比对密码哈希

1. **N3(fn:validateCredentials)** 调用 **N4(ext:bcrypt.compare)**
2. 不匹配 → 抛 InvalidCredentialError

### 3. Session 签发
> src/auth/session.ts — 颁发 JWT 并写入 Redis

...

### 4. 失败回收
> src/middleware/error.ts — 统一错误响应

...
```

**section 数量约束**：
- **M 档：4–8 个 section**（少于 4 → 主线太单薄，请合并为更大叙述；多于 8 → 拆成两张 codemap）
- S 档：1–2 个 section（节点本身少）
- L 档：5–12 个 section（扩展点多）

**section 排序**：按 execution / data / control flow 顺序；通常"入口 → 主路径 → 副作用 → 失败回收"四段式是好默认。

#### 3.3 Overview 内联引用（必须）

Overview 段落（3–5 句）必须**至少含 2 个 `[Nx]` 形式的内联引用**锚定关键节点；可选 `[§N]` 引用 Flow section。

**示例**：

```markdown
## Overview

本 codemap 覆盖 POST /login 端到端：从 HTTP 入口 [N1] 经凭证校验 [N3] 到 Session 签发 [N6]。
失败路径在 [§4] 集中处理，由 [N5] 捕获并归一化错误响应。
**非目标**：SSO / OAuth / MFA / 密码重置流程不在本切片内。
```

**约束**：
- `[Nx]` 引用必须是 Nodes section 中真实存在的节点 id，否则视为悬空引用
- 引用形式：`[Nx]` 或 `[Nx(kind:name)]`（Overview 不强制 `kind:name` 标注，但允许）
- `[§N]` 必须对应 Flow 中存在的 section 编号
- Overview 必须显式声明**非目标 / out-of-scope**（防止读者误以为切片穷尽）

### 4. 摘取 Excerpt（M 档覆盖率策略）

**必带 excerpt 的节点**：
- 入口节点（entry）
- 分支转折节点（if / switch / try 核心）
- 失败处理节点（error / catch / fallback）
- 跨层跳点（同步→异步 / 进程间 / 网络出入口）

**可省 excerpt 的节点**：平凡直通、纯代理、getter/setter

**每段 excerpt 规范**：
- 3–20 行真实连续源码
- 签名 + 关键控制流（if / try / emit / return）必留
- 日志 / 平凡 getter 可删，用 `// ...` 标注
- 超长函数用 `// ... (N 行省略)` 注明跳过行数
- 必须声明 `excerpt_lang`（ts / py / go / ...）和 `excerpt_lines`（与 Nodes 里 `path` 行号区间对齐）

### 5. 写盘

**两步原子操作**：

1. 写 `<project>/.codemaps/<slug>.md`（frontmatter + 7 sections 全量）
2. 更新 `<project>/.codemaps/index.md` —— **必须 Read-first → 内存拼接 → 整体覆盖写**，禁止流式 append（防中断损坏；index 条目含 slug / scope / title / tags / last_touched）

**Frontmatter 日期字段写法规范**：
- **统一用 `YYYY-MM-DD` 纯日期格式，不加引号、不写时分秒**
- 适用字段：`created_at` / `last_touched` / `history.introduced_at` / `history.last_refactor`
- 示例：`created_at: 2026-04-24`（✅）；`created_at: '2026-04-24'`（✅ 兼容但啰嗦）；`created_at: 2026-04-24T01:15:00Z`（❌ 会被 parser 截断，信息浪费）
- 背景：parser 自动把 YAML Date 规范化为 `YYYY-MM-DD` 字符串，保证 `@` 注入和 viewer 看到一致的短日期

**字符串字段引号规则**（避免 YAML 陷阱）：
- 含冒号 `:` / 井号 `#` / 方括号 `[]` 的标题和描述**必须单引号**（如 `title: 'POST /api/orders: 下单流程'`）
- 纯拉丁字母+中文+连字符的简单字符串可省引号
- 节点 `path: src/auth/login.ts:42-58` 里的冒号属于值的一部分，YAML 能解析，无需引号

**不自动写 `.gitignore`**：`.codemaps/` 默认跟踪，作为团队共享的任务上下文档案（IMPLEMENTATION.md §0.2 决策 17）。

### 6. 自检清单

生成完毕**自行**核对（全部 ✅ 才向用户交付）：

- [ ] Frontmatter 8 字段齐全：slug / scope / kind / title / entry / created_at / last_touched / tags（last_touched 可与 created_at 相同）
- [ ] Flow section 中每个 `Nx` 都在 Nodes section 定义（不能引用不存在节点）
- [ ] Flow section 中每个 `Nx` 的**首次出现**带 `(kind:name)` 标注，kind 归一化后与 Nodes 对应项一致
- [ ] Node Details section 中每个 `###` id 都在 Nodes section 定义（标题亦建议带 `(kind:name)` 标注）
- [ ] 每个 excerpt 与源文件对应 `excerpt_lines` 行号**逐字一致**（除 `// ...` 省略外）
- [ ] Edges 中每个 from/to 都指向真实存在的 Node id
- [ ] Nodes 的 `kind` 属于词表 `function/class/middleware/handler/hook/external/branch`
- [ ] 节点数在 M 档区间（15–60），若超出或不足 → 拆分成两张 codemap 或提醒用户调整 scope
- [ ] **Flow 章节分 4–8 个 `### <序号>. <title>` section**（M 档；S 档 1–2 / L 档 5–12），不可退化为单段时序
- [ ] **每个 Flow section 标题下有 `> ` blockquote 一行**描述 package / module scope
- [ ] **Overview 段落含 ≥2 个 `[Nx]` 内联引用**，且引用的 Nx 在 Nodes 中真实存在
- [ ] Overview 显式声明 **非目标 / out-of-scope**
- [ ] Title / Overview / section 标题使用**用户原始语言**（中文项目用中文，避免无谓翻译；代码符号名保持原文）
- [ ] `index.md` 已追加本 codemap 条目，其他条目保留

### 7. 反馈给用户

输出格式：

```
✅ 已生成 codemap: <slug>
  - 路径: .codemaps/<slug>.md (<节点数> nodes / <边数> edges)
  - 入口: <entry.ref>
  - 引用字符串: @.codemaps/<slug>.md

启动 viewer 可视化:
  cd <project-root> && codemap-viewer
  # → http://localhost:4676
  # 若未 link：node ~/.claude/codemaps/viewer/dist/cli.js
```

## 硬约束（不可违反）

1. **禁止臆造 path:line**：所有节点 path 必须来自真实 Read，禁止根据函数名猜测行号
2. **禁止改写 excerpt**：excerpt 必须是源文件真实连续行（允许 `// ...` 省略），禁止拼接 / 改写 / 生成式重构
3. **节点数硬上限 60**：逼近即停止扩展，按优先级剪枝；若任务本身超过，提议用户拆成两张 codemap
4. **index.md 必须 read-first 整体覆盖写**，禁止流式 append（原子性保证）
5. **不自动写 .gitignore**：`.codemaps/` 默认跟踪
6. **日期字段统一 `YYYY-MM-DD`**：不加引号、不写时分秒（parser 会把 Date 归一化截断）
7. **Flow NodeRef 首次必带 `(kind:name)` 标注**：kind 属于词表（全名或缩写均可），归一化后必须与 Nodes 一致；parser 会在不一致 / 未知 kind / Nx 未定义时直接报错
8. **Flow 必须多 section**：M 档 4–8 个 `### <序号>. <title>` section，每节带 `> ` blockquote 一行 scope 描述；不可退化为单段时序列表
9. **Overview 必须含内联引用**：≥2 个 `[Nx]` 关联到 Nodes 真实节点；不可只是抽象散文；必须显式声明非目标
10. **保留用户语言**：title / Overview / section 标题使用用户输入时的语言（中文 / 英文 / 混合），不强行翻译；只有代码符号名保持原文

## 非目标

- **不做**全系统拓扑 / 架构俯瞰 / 重构用依赖全景图（那是其它工具的职责）
- **不修改**源代码
- **不运行** git commit / push 等改变仓库状态的命令
- **单次调用只产一个 codemap**；多任务分多次调用
- **MVP 不做**漂移检测（excerpt_sha 校验延后到 v0.2）

## 相关

- Schema 规格：`~/.claude/codemaps/schema-design.md`
- 实施方案：`~/.claude/codemaps/IMPLEMENTATION.md`
- Viewer 源码：`~/.claude/codemaps/viewer/`（Hono + Cytoscape + Prism）
