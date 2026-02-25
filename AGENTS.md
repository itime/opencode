- To regenerate the JavaScript SDK, run `./packages/sdk/js/script/build.ts`.
- ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE.
- The default branch in this repo is `dev`.
- Local `main` ref may not exist; use `dev` or `origin/dev` for diffs.
- Prefer automation: execute requested actions without confirmation unless blocked by missing info or safety/irreversibility.

## 云效工作项集成

本项目通过软链接使用 `/Users/lzw/code/my/go/go-blog/.agents` 目录下的 Agent 系统进行任务管理。

### 常用命令

```bash
# 查询待处理任务
python3 .agents/cli.py tasks

# 查询工作项详情
python3 .agents/cli.py query BLOG-123 --full

# 认领任务
python3 .agents/cli.py claim BLOG-123

# 完成任务
python3 .agents/cli.py complete BLOG-123 -d "完成说明"

# 创建 HANDOFF 任务（同步到云效）
python3 .agents/cli.py handoff create --desc "任务描述" --hours 4h --priority P1 \
  --background "需求背景" \
  --goal "具体目标" \
  --approach "技术方案" \
  --acceptance "验收标准"

# 查看 HANDOFF 任务列表
python3 .agents/cli.py handoff list
```

### Agent 系统说明

Agent 系统位于 `/Users/lzw/code/my/go/go-blog/.agents`，通过软链接到本项目的 `.agents` 目录。这样做的好处是：

- 只维护一套 Agent 系统
- 多个项目共享同一套工作流
- Agent 系统更新后所有项目自动生效

如需了解完整的 Agent 系统文档，请参考 `/Users/lzw/code/my/go/go-blog/.agents/README.md`。

## 本地开发与构建

### Agent 修改代码后

Agent 修改 opencode 源码后，需要重新编译才能生效：

```bash
cd /Users/lzw/code/my/node/opencode/packages/opencode && bun run build --single
```

`--single` 参数只编译当前架构（darwin-arm64），速度更快。

编译完成后重启 opencode 即可看到修改效果。

### 开发模式（实时修改）

```bash
cd /Users/lzw/code/my/node/opencode
bun run dev
```

### 编译二进制（让 alias 生效）

```bash
cd /Users/lzw/code/my/node/opencode/packages/opencode
bun run build --single
```

编译后重启 opencode 即可使用新功能。

## 自定义配置

### Model Context 配置

在 `~/.config/opencode/opencode.json` 中配置 model 的 context 大小：

```json
{
  "provider": {
    "my-provider": {
      "models": {
        "claude-opus-4-6": {
          "name": "claude-opus-4-6",
          "context": 1000000
        }
      }
    }
  }
}
```

支持两种格式：

- 简写：`"context": 1000000`（推荐）
- 完整：`"limit": { "context": 1000000 }`

### 状态栏 Token 显示

状态栏的 context 信息支持鼠标悬停：

- 默认显示：`🧠 ▆▆▆▆▆▆▁▁ 75%`（进度条）
- 悬停显示：`🧠 150,000/1,000,000 tok (15%)`（详细 token 数）

## Style Guide

### General Principles

- Keep things in one function unless composable or reusable
- Avoid `try`/`catch` where possible
- Avoid using the `any` type
- Prefer single word variable names where possible
- Use Bun APIs when possible, like `Bun.file()`
- Rely on type inference when possible; avoid explicit type annotations or interfaces unless necessary for exports or clarity
- Prefer functional array methods (flatMap, filter, map) over for loops; use type guards on filter to maintain type inference downstream

### Naming

Prefer single word names for variables and functions. Only use multiple words if necessary.

```ts
// Good
const foo = 1
function journal(dir: string) {}

// Bad
const fooBar = 1
function prepareJournal(dir: string) {}
```

Reduce total variable count by inlining when a value is only used once.

```ts
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

### Destructuring

Avoid unnecessary destructuring. Use dot notation to preserve context.

```ts
// Good
obj.a
obj.b

// Bad
const { a, b } = obj
```

### Variables

Prefer `const` over `let`. Use ternaries or early returns instead of reassignment.

```ts
// Good
const foo = condition ? 1 : 2

// Bad
let foo
if (condition) foo = 1
else foo = 2
```

### Control Flow

Avoid `else` statements. Prefer early returns.

```ts
// Good
function foo() {
  if (condition) return 1
  return 2
}

// Bad
function foo() {
  if (condition) return 1
  else return 2
}
```

### Schema Definitions (Drizzle)

Use snake_case for field names so column names don't need to be redefined as strings.

```ts
// Good
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})

// Bad
const table = sqliteTable("session", {
  id: text("id").primaryKey(),
  projectID: text("project_id").notNull(),
  createdAt: integer("created_at").notNull(),
})
```

## Testing

- Avoid mocks as much as possible
- Test actual implementation, do not duplicate logic into tests
- Tests cannot run from repo root (guard: `do-not-run-tests-from-root`); run from package dirs like `packages/opencode`.

## Debugging with Local @opentui/core

opencode 依赖 `@opentui/core`。调试 opentui 代码比较复杂，因为：

1. opencode 编译成独立二进制，会打包所有依赖
2. `bun link` 在 monorepo 中不能直接用于子包
3. 本地 opentui 需要先构建，且需要 native 模块（Zig）

### 推荐方法：开发模式运行

不要用编译后的二进制，直接用开发模式：

```bash
cd /Users/lzw/code/my/node/opencode
bun run dev
```

这样会直接运行 TypeScript 源码，可以实时修改 opencode 代码。

### 如果必须调试 opentui

1. 在 opentui 仓库中写测试用例复现问题
2. 或者在 opentui 的 examples 目录创建示例来调试
3. 修复后发布新版本，然后在 opencode 中更新依赖

### 调试日志方法

由于 opentui 是终端 UI 库，`console.log` 不会显示。使用文件日志：

```typescript
import { appendFileSync } from "node:fs"

const debugLog = (msg: string, data?: any) => {
  const line = `[${new Date().toISOString()}] ${msg} ${data ? JSON.stringify(data) : ""}\n`
  appendFileSync("/tmp/opentui-debug.log", line)
}
```

查看日志：`cat /tmp/opentui-debug.log`
