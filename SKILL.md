---
name: codemap
description: Generate a navigable code map for a specific task or flow in the current project — a focused slice of call paths with real code excerpts — saved to `.codemaps/<slug>.md`. Serves as both an `@`-referenceable source for Claude and a visualization input for the local viewer.
triggers:
  - make a codemap / generate a codemap / codemap this
  - draw a code map for X flow/endpoint/event
  - distill this call chain into a reusable codemap
---

# Codemap Generator

Produce a precise, task-driven Call-Path Slice code map for a coding project. **Anti-vibeslop**: build shared understanding before writing code.

The full output spec lives in §5 of `schema-design.md` at the project root; this SKILL only guides the generation steps and hard constraints.

**Core shape**:
- **Three node types**: Code Node (has ID + code) / Pointer Node (no ID, has file:line) / Group Node (no ID, no code)
- **Node ID = section-local compound number**: `<section-num><letter>`, e.g. `1a` / `1b` / `2a` / `3c`
- **Flat H2 document structure**: `## Overview` / `## 1. Section` / `## 2. Section` / ... / `## Narrative` / `## Notes`
- **Overview must contain a main flow reference chain**: `[1a]→[1b]→[3c]`, at least 3 IDs joined with `→`

## Activation Prerequisites

1. **Working directory is a coding project** (has source code; not a docs-only repo)
2. **User has provided a task / entry description** (e.g. "login flow", "POST handler for /api/orders", "WebhookProcessor extension point")

If either is missing: fill the gap before entering Step 1.

## Thinking Process (6 phases — mandatory for every codemap)

1. **Interpret** — Map the user's entry description to code symbols / files / endpoints / events / extension points. **Preserve the user's original language** as `title` and section names; do not force translation. Expand vague terminology into code concepts; **never fabricate behavior you have not observed**.
2. **Explore** — Starting from the entry, walk real callable paths (concrete follow paths in Step 2 below); chase **upstream data sources** as well as **downstream side effects**; prefer code paths that explain execution order and component relationships.
3. **Evidence Graph** — Collect real nodes (all three kinds: Code / Pointer / Group) + file:line + code excerpt; every Code Node must carry a stable section-local id (`1a` / `2c` ...); strip low-value implementation details that do not support the main story.
4. **Group** — Aggregate nodes into **4–8 sections**, each describing a coherent flow or subsystem; order by execution / data / control flow; attach a one-liner **Core packages - <package/module scope description>** to each.
5. **Render** — Distill the title from the entry and the scope you discovered; **Overview must contain ≥1 main flow reference chain** (`[1a]→[1b]→[3c]`, ≥3 IDs joined with `→`) + ≥5 inline references in total + an explicit non-goals statement; write the nested markdown bullet tree per the §5.4 schema.
6. **Validate** — Verify every path / line / code excerpt is real; remove unsupported claims; ask yourself one final question: "**Can I navigate the codebase with this map?**" — A document-only artifact does not pass.

## Steps

### 1. Align Task Boundaries (interactive with the user)

- **Restate the task goal in one sentence**
- **Confirm the entry anchor**: type (`function` / `endpoint` / `event` / `extension-point`) + ref (symbol / HTTP path / event name)
- **Propose a slug**: derive a kebab-case slug from the title, e.g. `user-login-flow` / `webhook-processor-extension`; if `.codemaps/<slug>.md` already exists, append `-2` / `-3`; final slug ≤ 50 chars
- **Declare boundaries**: state explicitly what this codemap **will not** cover (to prevent exploration creep)

### 2. Explore the Call Chain (iterative retrieval)

- **Retrieval strategy**: iterative retrieval — coarse-scan candidates → score relevance → only read details after converging; do not swallow whole files in one shot
- **Tool priority**: prefer the host environment's semantic retrieval (LSP-aware MCP / symbol index / semantic search); when unavailable, fall back to the grep / glob / Read trio to trace a few hops outward and inward from the entry
- **Walk these real callable paths** (by relevance — list as needed; skip if a concept does not apply):
  - **Imports** — the dependency closure of the entry file
  - **Hooks / lifecycle** — `useEffect` / `onMount` / middleware / interceptors / signal subscriptions
  - **Component composition** — parent/child components, `children` / slot, provider, HOC
  - **Event paths** — `emit` / `on` / `dispatch` / listener / pub-sub
  - **Model / state** — store, reducer, context, the **read & write sites** of reactive objects
  - **Routing / data / schema** — route config, loader / resolver, ORM model, API schema
  - **Styling / design system** — theme, design tokens, variants, materials
  - **Design-time vs runtime variants** — feature flags, env branching, build-time injection
  - **Upstream data sources** — where parameters originate, how context is injected, external input boundaries
  - **Downstream side effects** — DB writes / network egress / logging / queue enqueues / file IO
- **Priority order**:
  entry > branch turning point > failure handling > cross-layer hop (sync↔async / inter-process / network IO) > upstream data source > downstream side effect > trivial pass-through

**Stopping criteria** (when to stop exploring — replaces the v0.1.3 size-tier rules):
- The end-to-end main path reaches its terminal failure handling / output side effect
- Every branch at the entry has been covered by at least one path in the map
- Exploring further would only add trivial pass-through nodes (helpers, wrappers, formatters that do not change the story)

**Size guidance** (advisory, not parser-enforced):
- Most codemaps land at 8–40 code nodes; aim for the smallest map that still covers the user's task end-to-end
- At ~50 code nodes, pause and re-examine the task boundary — if the "task" is actually a composite of subtasks, ask the user to split it into smaller tasks and invoke the skill once per subtask (the "one task, one codemap" principle holds — do not split a single codemap into multiple files)
- At ~80 code nodes, the task scope is likely too broad — re-scope with the user before continuing

**Anti-fabrication hard rules (non-negotiable)**:
- Every file:line must come from a **real file read** (Read tool); never invent symbol names or line numbers from training-set memory
- Every Code Node's code must be **real consecutive lines** from the source file; `// ...` may be used to skip irrelevant inner branches; **never rewrite / splice / synthesize code**

**Secret-safety hard rules (non-negotiable — the codemap is git-tracked, a leaked secret enters history)**:

The verbatim-excerpt mandate above would, unguarded, copy any API key / token / password / private key sitting in source or config straight into `.codemaps/<slug>.md`. Two defenses, applied at selection and excerpt time (a third, mechanical backstop runs in §5.0):

- **Avoid (selection time)** — Secret-bearing files (`.env` / `.env.*`, `*.pem` / `*.key` / `*.p12`, `secrets.*`, `credentials*`, `*.tfvars`, anything matching a gitignored-secret pattern) are represented as **Pointer Nodes** (`**<label>** — \`file:line\``, no fenced code). Point at *where* config is loaded; never quote the secret value itself.
- **Redact (excerpt time)** — If a chosen excerpt still contains a credential literal, replace **the secret value inside its string literal** with the typed placeholder `‹REDACTED:<kind>›` (e.g. `const KEY = "‹REDACTED:api-key›";`). This is the **single allowed exception** to "never rewrite code" — it is honest masking in the same category as `// ...` elision: the line shape stays real, only the secret token is hidden. Do **not** delete the line, replace it with a comment, or paraphrase the surrounding code.
- **Patterns to treat as secrets**: provider-prefixed tokens (`AKIA…` / `ASIA…` / `AROA…` / `ghp_…` / `github_pat_…` / `xox[baprs]-…` / `sk-…` / `sk_live_…` / `AIza…`), `-----BEGIN … PRIVATE KEY-----` blocks, JWTs (`eyJ….eyJ….…`), connection strings with inline credentials (`scheme://user:pass@host`), and quoted hardcoded assignments to `password` / `secret` / `api_key` / `access_token` / `client_secret` / `private_key`.

### 3. Construct the Document

> The examples in this section use Chinese labels because the canonical example project's input language was Chinese. In an English-input project, the labels would be in English. Examples are about structure and indentation, not language.

#### 3.1 Three Node Types

| Type | When to use | Line format (mind exact spacing & indentation!) |
|---|---|---|
| **Code Node** | Critical code segments (entry / branch turn / failure handling / cross-layer hop / important call) | `- **<id>** <action label> — \`<file:line>\`` immediately followed by a fenced code block |
| **Pointer Node** | Context anchor: function signature entry, component mount point, parent file location (so the user can jump to source) | `- **<label>** — \`<file:line>\`` |
| **Group Node** | Pure structural label: loop description, branch group, subsystem heading | `- **<label>** ` (optional `— \`<file:line>\``) |

**Node label style**:
- **Action description in the user's input language** ("获取路由数据" / "渲染 Nav 子组件" / "用户点击菜单项" — verb-first)
- Symbol names appear only inside the fenced code block; **do not put them in the label**

**ID numbering rules**:
- Form: `<section-num><letter>`, e.g. `1a` / `1b` / `2c`
- `section-num` must equal the H2 number of the enclosing section (`1a` must live under `## 1. ...`)
- `letter`: a–z in section order (**soft rule: prefer consecutive, gaps are allowed and the parser does not error; if you exceed 26 nodes, split the section — do not introduce `aa` / `ab`**)
- Letters must be unique within a section (parser errors)
- Letters may repeat across sections (`1a` and `2a` do not conflict)
- Each section ≥ 1 Code Node

#### 3.2 Document Skeleton & Frontmatter Fields

**Overall structure (flat H2)**:

```markdown
---
<frontmatter fields, see table below>
---

## Overview              # meta section: 3-5 sentences + ≥1 main flow reference chain + explicit non-goals
## 1. <Section Title>    # flow section (M tier: 4-8)
## 2. <Section Title>
...
## N. <Section Title>
## Narrative             # meta section (optional)
## Notes                 # meta section (optional)
```

**Frontmatter field spec**:

| Field | Required | Type / Values | Description |
|---|---|---|---|
| `slug` | ✅ | kebab-case string, ≤50 chars | Filename = `<slug>.md`; bound to `.codemaps/<slug>.md`; on conflict append `-2` / `-3` |
| `kind` | ✅ | fixed value `call-path-slice` | Document-type discriminator; currently the only valid value (parser rejects others) |
| `title` | ✅ | string in user's original language | One-line task description; if it contains `:` / `#` / `[]`, single-quote it (see §5 quoting rules) |
| `entry.type` | ✅ | `function` / `endpoint` / `event` / `extension-point` enum | Entry anchor type |
| `entry.ref` | ✅ | string | Reference to the entry — function symbol (preferably with `file:line`) / HTTP path / event name / extension-point ID |
| `created_at` | ✅ | `YYYY-MM-DD` | First-generated date; no quotes, no time component (see §5 date rules) |
| `last_touched` | optional | `YYYY-MM-DD` | Last-updated date; recommended to refresh on every edit |
| `tags` | optional | string array | Used by `index.md` and viewer filter; e.g. `[auth, login]` |
| `history.introduced_at` | optional | commit sha or `YYYY-MM-DD` | When the flow was introduced; useful for tracing history |
| `history.last_refactor` | optional | commit sha or `YYYY-MM-DD` | When the flow was last refactored |

**Full example**:

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

> See §5 "Persistence" for YAML quoting rules and the date-format hard constraint when writing to disk.

#### 3.3 Internal Structure of a Section (**verbatim template — copy the indentation exactly**)

```markdown
## 1. RouteNav 组件渲染流程
> Core packages - RouteNav 主组件如何包装 Nav 子组件并传递路由数据

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

**Critical indentation rules** (CommonMark is strict):
- Top-level list item: `- ` followed by content (indent 0)
- Nested list item: `  - ` indent **2 spaces**
- Fenced code block under a list item: indent **4 spaces × depth + 4** to align with list content (e.g. a code block under a 2nd-level list item indents 4 spaces)
- Nesting depth ≤ 4 levels

**Fenced code block length**: 1–5 lines of real code; for long functions, keep only the signature + key call sites and elide the middle with `// ...`.

#### 3.4 Overview Inline Reference Rules

The Overview is 3–5 sentences and **must** satisfy:

1. **≥1 main flow reference chain**: `[xa]→[xb]→[xc]`, ≥3 IDs joined with `→`, describing the end-to-end main path
2. **≥5 inline references in total**: includes main + side references (e.g. failure paths, designer paths)
3. **Reference Code Nodes via `[<id>]`** (`[1a]`, no `(kind:name)`); optionally `[§<num>]` to reference a Flow section
4. **Explicit non-goals statement**: use `**非目标**: ...` or an equivalent unambiguous phrasing in the user's language
5. **User's original language**

**Example**:

```markdown
## Overview
本代码地图展示了 RouteNav 导航组件的完整实现，包括组件渲染流程、Router 状态管理、用户交互处理、Schema 解析、设计器集成和样式系统。关键流程包括：路由数据获取 [1a] → Nav 渲染 [1b] → 菜单点击 [3c] → History 更新 [3g] → 路由状态计算 [2g]。设计器环境通过 TreeNode 过滤 [5b] 实现独立的路由管理，样式系统通过素材配置 [6a] 和动态计算 [6e] 实现主题化。

**非目标**：不覆盖 SSR 渲染路径、单元测试代码、历史 v1 实现。
```

#### 3.5 Section Title & Scope Blockquote

- Title format: `## <num>. <Section Title>` (number + dot + space + title in the user's input language)
- The line right after the title is a blockquote: `> Core packages - <one-line package / module scope description>`
- Section ordering: by execution / data / control flow; a good default is "entry → main path → side effects → failure recovery"

### 4. Cross-Section Relationships

There is no separate Edges section. To express cross-section transitions:
- **Preferred**: encode it in the Overview's main flow reference chain (`[1a]→[1b]→[3c]→[2g]` naturally crosses multiple sections)
- **Secondary**: describe it in `## Narrative` as free-form prose
- **Do not** append a structural `→ [2g]` to the end of a node label — the parser does not extract it, and free text is misleading

### 5. Persistence

#### 5.0 Pre-write validation (mandatory — repair loop)

The schema rules listed in §3 (frontmatter required fields, top-level section
names, flow section continuity, code-node id letter uniqueness, Overview
ref → code-node binding, …) are **mechanically enforced** by the parser
shipped with the local viewer. Self-check (§6) is necessary but not
sufficient — run the validator on the in-memory markdown FIRST, and only
persist after it passes.

**How to invoke** (pipe the assembled markdown via stdin so nothing hits
disk on a failed attempt):

```bash
# `-` means read from stdin; npx pulls the published binary so this works
# in any project that doesn't have it linked locally.
printf '%s' "$GENERATED_MARKDOWN" | npx -y @tttinkl/codemaps-viewer validate -
```

If `codemap-viewer` is already on `$PATH` (globally linked), drop the `npx`
prefix.

**Interpreting the result**:

- **Exit 0** → output `✓ valid: <stdin>`. Proceed to 5.1.
- **Exit 1** → **schema violation OR an embedded credential** (validate runs
  the secret scan on every schema-clean document). A secret diagnostic looks
  like `<stdin>:<line>:<col> - error: possible <kind> credential embedded in
  codemap content (…)` and stderr **may contain multiple blocks** — one per
  finding. Fix **all** of them in one pass: apply the §2 redact rule
  (`‹REDACTED:<kind>›` inside the string literal) to every flagged line, then
  re-validate. This shares the 3-attempt repair budget below, so do not
  redact one-at-a-time. A schema violation instead carries a single
  diagnostic shaped like:
  ```
  <stdin>:<line>:<col> - error: <message>

  <line> | <offending source line>
         | <caret pointing at column>

  hint: <remediation suggestion>
  ```
  Read the message + hint, edit the in-memory markdown, then re-validate.
- **Exit 2** → validator IO/usage problem (file unreadable, missing
  argument). Surface to the user; do not retry blindly.

**Repair loop budget: at most 3 attempts.** If the 3rd attempt still fails,
**stop**: present the final diagnostic to the user verbatim and ask for
guidance. **Never write a file that fails parsing** — bad codemaps poison
the viewer, the VS Code extension, and downstream `@`-references.

#### 5.1 Atomic two-step write (only after 5.0 passes)

1. Write `<project>/.codemaps/<slug>.md` (frontmatter + Overview + N flow sections + Narrative + Notes)
2. Update `<project>/.codemaps/index.md` — **must Read-first → assemble in memory → overwrite the whole file**; never streaming-append (to avoid corruption on interrupt; index entries include slug / scope / title / tags / last_touched)

#### 5.2 Backfill freshness sha (mandatory — v0.2)

After 5.1 succeeds, immediately run the backfill CLI to populate the v0.2
freshness fields (`files:` map + `last_verified:`) on the just-written
codemap:

```bash
# `<slug>` is the same slug used in the filename (no `.md` suffix).
# `--project <projectRoot>` defaults to cwd; pass it explicitly when
# the codemap was written under a non-cwd project root.
npx -y @tttinkl/codemaps-viewer backfill <slug>
```

If `codemap-viewer` is already on `$PATH` (globally linked), drop the `npx`
prefix.

**Hard rule — do NOT write `files:` or `last_verified:` yourself.** sha
generation is the CLI's job:
- It reads each `pathRef.relPath` from the codemap
- Reads the source file contents
- Computes a deterministic 16-hex sha256 over normalized content
- Writes them back into the codemap's frontmatter

Hand-written shas will be wrong (different normalize / hash) and cause every
freshness check to false-positive "stale". Always delegate to the CLI.

**Interpreting the result**:
- `✓ <slug>  — N file(s), wrote sha for K, M missing` → done. The codemap
  is now v0.2-ready and the viewer / VS Code extension will show ✓ Fresh.
- `✗ <slug>  — parse error: <message>` → the codemap you just wrote does
  not parse. This should not happen because §5.0 already validated it; if
  it does, surface the message verbatim and ask the user to investigate.
- `M missing > 0` → some `relPath`s in your codemap point at files that
  don't exist on disk. Fix the relPaths (typo / wrong project root) and
  re-run, or proceed and let the freshness UI flag those files as missing.

**Exit codes**: 0 OK, 1 some failures, 2 IO/usage error (treat as a stop
condition).

**Frontmatter date field rules**:
- **Always plain `YYYY-MM-DD`; no quotes, no time component**
- Applies to: `created_at` / `last_touched` / `history.introduced_at` / `history.last_refactor`
- Examples: `created_at: 2026-04-24` (✅); `created_at: '2026-04-24'` (✅ tolerated but verbose); `created_at: 2026-04-24T01:15:00Z` (❌ parser truncates)

**String field quoting rules** (avoid YAML pitfalls):
- Titles or descriptions containing `:` / `#` / `[]` **must be single-quoted** (e.g. `title: 'POST /api/orders: 下单流程'`)
- Plain ASCII letters + CJK + hyphens may go unquoted

**Do not auto-write `.gitignore`**: `.codemaps/` is tracked by default — it is the team's shared task-context archive.

### 6. Self-Check Checklist

The structural rules in this list (frontmatter shape, section names &
continuity, id letter uniqueness, Overview ref binding, …) are already
**mechanically verified** in §5.0. The remaining items below — language
preservation, code authenticity, label style, etc. — are what the parser
**cannot** check; they are your responsibility.

After generating, **self-verify** (every box ✅ before delivering to the user):

**Frontmatter**:
- [ ] All required fields present: slug / kind / title / entry / created_at (scope / last_touched / tags / history optional)
- [ ] `kind` value is `call-path-slice`

**Document structure**:
- [ ] Top-level sections are ONLY `## Overview` / `## <num>. <title>` / `## Narrative` / `## Notes`; **no `## Nodes` / `## Edges` / `## Node Details` / `## Flow`**
- [ ] Flow section count is 4–8
- [ ] Section numbers are continuous from 1 (1, 2, 3, ... — no gaps)
- [ ] Each section title is `## <num>. <title>`
- [ ] Each section has a `> Core packages - <description>` blockquote on the next line

**Overview**:
- [ ] Contains ≥1 `[xa]→[xb]→[xc]` main flow reference chain (≥3 IDs joined by `→`)
- [ ] Total inline `[<id>]` references ≥5 (may include `[§<num>]`)
- [ ] All referenced IDs really exist in some Flow section
- [ ] Explicit non-goals / out-of-scope statement

**Nodes (within each section)**:
- [ ] At least 1 Code Node (with `**<id>**` + fenced code)
- [ ] Code Node ID prefix matches the section number (`1a` lives in `## 1.`)
- [ ] Letters within a section are unique
- [ ] Each Code Node has `**<id>** <label> — \`<file:line>\`` + fenced code (1–5 lines)
- [ ] Each Pointer Node has `**<label>** — \`<file:line>\``
- [ ] Each Group Node is just `**<label>**` (optional file:line)
- [ ] Node labels are **action descriptions in the user's input language** (verb-first; symbol names not in labels)
- [ ] Nesting depth ≤ 4 levels
- [ ] 2-level list indents 2 spaces; fenced code indents 4 spaces × depth

**User's language**:
- [ ] Title / Overview / section titles / node labels use the user's original language
- [ ] File paths / symbol names / line numbers are kept verbatim

**Authenticity**:
- [ ] Every file:line came from a real Read (you used the Read tool)
- [ ] Every fenced code excerpt matches the source file **verbatim** (modulo `// ...` elisions and `‹REDACTED:<kind>›` secret masking)

**Secret safety**:
- [ ] No raw API key / token / password / private key in any fenced code or frontmatter; secret-bearing files are Pointer Nodes; any unavoidable literal is masked as `‹REDACTED:<kind>›` inside its string literal
- [ ] §5.0 validate exited 0 (it also fails exit 1 on detected credentials)

**index.md**:
- [ ] An entry for this codemap was added; existing entries preserved

**Freshness backfill (v0.2)**:
- [ ] `codemap-viewer backfill <slug>` was run AFTER §5.1 wrote the file
- [ ] CLI reported `✓ <slug>  — N file(s), wrote sha for K`
- [ ] No hand-written `files:` / `last_verified:` in the frontmatter

### 7. Reporting Back to the User

Output format:

```
✅ Generated codemap: <slug>
  - Path: .codemaps/<slug>.md (<num-code-nodes> code nodes across <num-sections> sections)
  - Entry: <entry.ref>
  - Reference string: @.codemaps/<slug>.md

To launch the viewer:
  cd <project-root> && codemap-viewer
  # → http://localhost:4676

Install codemap-viewer globally:
  pnpm add -g @tttinkl/codemaps-viewer
```

## Hard Constraints (non-negotiable)

1. **Never fabricate path:line**: every file:line comes from a real Read; do not guess line numbers from function names
2. **Never rewrite code**: fenced code blocks must be real consecutive lines from the source (`// ...` elisions allowed); no splicing / rewriting / generative refactoring — the **sole exception** is masking a secret value as `‹REDACTED:<kind>›` inside its string literal (see Secret-safety hard rules in §2)
3. **Code Node ≤ 5 lines**: for long functions, keep only the signature + key call sites
4. **Node ID = section-local compound** `<num><letter>`: do not use a global `Nx` form
5. **Code Node must have a fenced code block**: 1–5 lines; Pointer Node must have file:line (no code); Group Node has only a label (file:line optional)
6. **Overview must contain a main flow reference chain + ≥5 inline references + an explicit non-goals statement**
7. **4–8 Flow sections + numbered from 1, continuous**
8. **Each section ≥ 1 Code Node + a `> Core packages - ...` scope blockquote**
9. **Nesting depth ≤ 4**: split the section if you go deeper
10. **Node labels are action descriptions in the user's input language**: symbol names live only in code blocks, not in labels
11. **Preserve the user's language**: title / Overview / section titles / node labels are in the user's input language; only code symbol names / file paths / line numbers stay in their original form
13. **`index.md` must Read-first then overwrite as a whole**: never streaming-append
14. **Date fields use `YYYY-MM-DD`**: no quotes, no time component
15. **Pre-write parser validation is mandatory**: run `codemap-viewer validate -` on the in-memory markdown before disk write; on failure, repair (≤3 attempts) and re-validate; never persist a file that fails parsing
16. **Post-write backfill is mandatory (v0.2)**: after writing the codemap to disk, run `codemap-viewer backfill <slug>` to populate `files:` sha map + `last_verified:` in the frontmatter — **never hand-write `files:` or `last_verified:` yourself** (sha generation is the CLI's job; manual values will all be wrong and trip every freshness check)
17. **Never emit a raw secret**: secret-bearing files are Pointer Nodes (no code excerpt); any unavoidable credential literal is masked as `‹REDACTED:<kind>›` inside its string literal (the sole allowed code rewrite). `validate` mechanically rejects (exit 1) any embedded credential — a leak here enters git history (see Secret-safety hard rules in §2)

## Non-Goals

- **No** whole-system topology / architectural overview / refactor-oriented dependency map
- **Do not** modify source code
- **Do not** run git commit / push or any command that mutates repo state
- **One codemap per invocation**; for multiple tasks, invoke multiple times
