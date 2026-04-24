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
- **Token 预算约束**：M 档节点数逼近 **60** 即停止扩展，按优先级剪枝（关键分支 > 失败处理 > 跨层跳点 > 平凡直通）

**防盲猜硬约束（不可违反）**：
- 所有 Node 的 `path:line` 必须来自**真实文件读取**（Read 工具），禁止凭训练语料臆造符号名或行号
- Edge 必须能在代码中找到**具体调用点**（`foo()` 调用 `bar()` 的那一行）；无法定位的 edge 不要写
- 每个 excerpt 必须是**源文件真实连续行**，允许用 `// ...` 省略标记跳过无关内部分支，**禁止改写/拼接/臆造代码**

### 3. 构造 Nodes / Edges

按 `schema-design.md` §5.1 的 YAML-ish 多行 bullet 格式：

```yaml
## Nodes
- id: N1
  name: login(req, res)
  path: src/auth/login.ts:42-58
  kind: handler          # function | class | middleware | handler | hook | external
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

Flow 章节用时序叙述引用 Nx（示例见 schema-design.md §5.1）。

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
- [ ] Node Details section 中每个 `###` id 都在 Nodes section 定义
- [ ] 每个 excerpt 与源文件对应 `excerpt_lines` 行号**逐字一致**（除 `// ...` 省略外）
- [ ] Edges 中每个 from/to 都指向真实存在的 Node id
- [ ] 节点数在 M 档区间（15–60），若超出或不足 → 拆分成两张 codemap 或提醒用户调整 scope
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
