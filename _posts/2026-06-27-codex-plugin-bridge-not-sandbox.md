---
title: "桥，不是沙箱：从零拆解一个 Codex Plugin —— 以 codexpro 为例"
description: 以开源项目 codexpro 为样本，拆解一个让 ChatGPT/Codex 访问本地代码仓库的 Codex Plugin（MCP Server + Apps SDK）是怎么造出来的，重点落在路径逃逸、能力开关、命令白名单这套安全边界设计上。
categories:
  - software
tags:
  - MCP
  - ChatGPT
  - Codex
  - AppsSDK
  - coding-agent
  - tutorial
---

![桥，不是沙箱：从零拆解一个 Codex Plugin](/images/2026-06-27-codex-plugin-bridge-not-sandbox.png)

如果你想让 ChatGPT 直接读本地仓库、改文件、看 diff，第一个问题其实不是"工具怎么注册"，而是"边界画在哪里"：它能碰哪些目录？能不能跑命令？凭据会不会被读出去？公网隧道地址被别人拿到怎么办？

`codexpro` 正好是一个适合拆开的样本：它把 MCP Server、ChatGPT 连接器、Apps SDK 工具卡片、本地文件系统访问和安全策略都揉在一起。本文就以它为蓝本，讲清楚"让 ChatGPT / Codex 访问本地代码仓库"这类插件是怎么造出来的。

> 本文不追求手把手写完每一行代码，而是按一个真实项目拆出一套可复用的插件设计模板。
> 风格：**先跑起来，再拆原理**。关键技术细节穿插在 step 之间的"深入"小节里。
>
> 读完你会得到：一个能在 ChatGPT Developer Mode 里调用、可以读/改本地仓库、能跑受限命令、带 UI 卡片的 MCP 插件，以及把它做安全、做发布的完整心智模型。

---

## 先搞清楚：什么是"Codex Plugin"

"Codex Plugin" 不是一种新格式，它就是一个 **MCP Server**（Model Context Protocol 服务器），再加上 ChatGPT 的 **Apps SDK** 渲染约定。拆开看只有三件事：

1. **MCP Server**：一个进程，向客户端声明自己有哪些 *tools*、*resources*，并实现这些工具的处理函数。协议是 JSON-RPC over 某种 transport。
2. **Transport（传输层）**：MCP 定义了两种主流通道——`stdio`（本地子进程，最简单）和 `Streamable HTTP`（一个 HTTP 端点，ChatGPT 云端要的就是这个）。
3. **Apps SDK / MCP Apps 约定**：在工具描述或返回值的 `_meta` 里声明 UI 模板（比如标准的 `_meta.ui.resourceUri`，以及 ChatGPT 兼容的 `openai/outputTemplate`），ChatGPT 就会把结果渲染成一张 UI 卡片，而不只是一段文本。

codexpro 把这三件事组合成了一个具体产品：**ChatGPT 通过一个 token 保护的本地 MCP 桥，操作你机器上的某一个仓库**。它能直接 `read` / `write` / `edit` / `bash` / `search` / `show_changes` 操作工作区；此外还能把任务**交接**出去——把一份实现计划写进 `.ai-bridge/`，由本地的实现 agent 读取后异步执行（机制见第 9 章）。注意这两类能力性质不同：前者是 ChatGPT 同步亲自动手，后者只是写一份"任务说明书"到文件、落地交给另一个本地 agent 完成。

一句话定位（来自仓库 `design.md`）：

> Use ChatGPT like your local coding agent. It is a local bridge, not a quota bypass, model proxy, hosted SaaS, or OS sandbox.

记住最后半句，它会贯穿整个安全设计：**这是一座桥，不是一个沙箱**。沙箱边界要你自己用代码画出来。

### 技术栈一览（codexpro 实际用的）

| 关注点 | 选型 | 文件 |
| --- | --- | --- |
| MCP SDK | `@modelcontextprotocol/sdk` | 全仓库 |
| 参数校验 | `zod` | 每个工具的 `inputSchema` |
| HTTP 服务 | `express@5` + `cors` | `src/http.ts` |
| 路径匹配 | `minimatch`（blocked globs） | `src/guard.ts` |
| 语言/构建 | TypeScript + `tsc`，ESM（`"type":"module"`，NodeNext） | `tsconfig.json` |
| 运行时 | Node ≥ 20 | `package.json` engines |

---

## 先以"用户"身份跑一遍（建立直觉）

写插件之前，先体验一遍它对终端用户长什么样。codexpro 把启动收敛成一个 CLI。

```bash
npm install -g codexpro      # 或在仓库内 npm install && npm run build
codexpro setup               # 交互式生成连接配置
codexpro start               # 起本地 MCP server + 隧道，打印一个 Server URL
```

`codexpro start` 干了两件事：在 `127.0.0.1:8787/mcp` 起 HTTP MCP 端点，再用 Cloudflare/ngrok 隧道把它暴露成一个 `https://xxx/mcp` 公网地址。先在 **ChatGPT → Settings → Apps & Connectors → Advanced settings** 打开 Developer Mode，再到 **Settings → Connectors → Create** 粘贴这个地址，ChatGPT 就连上了你的本地仓库。

之后在对话里，ChatGPT 会自己调用 `open_current_workspace`、`tree`、`read`、`edit`、`show_changes` 这些工具，像一个坐在你电脑前的 coding agent。

> **为什么要隧道？** ChatGPT 的连接器是从 OpenAI 云端发起 HTTP 请求的，它够不到你的 `localhost`。隧道把回环地址换成一个公网 HTTPS URL。这也立刻带来一个安全问题——任何拿到这个 URL 的人都能操作你的仓库——所以非回环访问必须带 token，详见第 5 章。

动手目标想清楚了，下面从最小的 server 开始造。

---

## 最小可用插件：一个 stdio MCP server

最快的反馈循环不是 HTTP，而是 `stdio`：MCP 客户端把你的脚本当子进程拉起来，用 stdin/stdout 收发 JSON-RPC。codexpro 的 stdio 入口只有十几行（`src/stdio.ts`）：

```ts
#!/usr/bin/env node
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { loadConfig } from "./config.js";
import { createCodexProServer } from "./server.js";

async function main(): Promise<void> {
  const config = loadConfig();                 // 解析 env / argv → 配置
  const server = createCodexProServer(config); // 构造 McpServer，注册所有工具
  const transport = new StdioServerTransport();
  await server.connect(transport);             // 接上传输层，开始监听
}

main().catch((error) => {
  console.error(error instanceof Error ? error.stack ?? error.message : String(error));
  process.exit(1);
});
```

**关键设计：传输层和业务逻辑彻底解耦。** `createCodexProServer(config)` 返回一个跟 transport 无关的 `McpServer`。stdio 入口和后面要讲的 HTTP 入口共用同一个 `createCodexProServer`，只是套上不同的 `transport`。你想支持新通道，只加一个入口文件即可，工具一行都不用动。

> **stdio 上不要往 stdout 打日志。** stdout 是 JSON-RPC 信道，任何杂音都会破坏协议帧。注意上面所有日志都走 `console.error`（stderr）。这是 stdio MCP 最常见的坑。

### 深入：`McpServer` 是什么

`createCodexProServer` 里第一句就是：

```ts
const server = new McpServer(
  { name: "CodexPro", version: "0.28.5" },
  { instructions: serverInstructions(config) }
);
```

`instructions` 是一段对模型的系统级提示，告诉它**该怎么用这些工具**。codexpro 在这里写死了一套工作流纪律，例如：

> 1. 先 `open_current_workspace`。
> 2. 用 `tree`/`search`/`read` 看代码，**不要**用 bash 去 `cat`/`grep`/`git status`。
> 3. 改完文件调一次 `show_changes` 看 diff。

这段 instructions 会随 `config` 动态变化（比如 write 关了就改说"写工具不可用，请用 handoff"）。**这是把"产品规则"喂给模型的主要手段**，比纯靠工具 description 更强。

---

## 注册第一个工具：`read`

工具是插件的原子能力。MCP SDK 的注册签名是 `registerTool(name, options, handler)`。codexpro 包了一层 `registerCodexTool` 来统一加埋点、错误处理和模式过滤，但内核就是这个。看一个被简化过的 `read`：

```ts
registerCodexTool(config, server, "read", {
  title: "Read File",
  description: "Read a UTF-8 text file inside the workspace. Returns bounded content.",
  inputSchema: {
    path: z.string().describe("File path relative to the workspace root."),
    workspace_id: z.string().optional().describe("From open_workspace; omit for default."),
    max_bytes: z.number().int().optional().describe("Override read cap."),
  },
  annotations: READ_ONLY_ANNOTATIONS,
}, async (args) => {
  const workspace = workspaces.getWorkspace(args.workspace_id);
  const result = await readTextFile(config, guard, workspace, args.path, args.max_bytes);
  return textResult(result.text, { path: result.relPath, bytes: result.bytes });
});
```

四个要点：

1. **`description` 也是 prompt**。注意 codexpro 在tool `show_changes` 的描述里直接写"Use this instead of bash git status / git diff"，主动避免模型使用`git status`和`git diff`。
2. **`inputSchema` 用 zod**。每个字段都带 `.describe(...)`——这段描述会进入工具 schema，模型靠它决定怎么填参数。**描述写得好坏直接决定模型调用得对不对**，把它当 prompt 写。
3. **`annotations`** 是给客户端的元数据，比如 `readOnlyHint`、`destructiveHint`。codexpro 预定义了 `READ_ONLY_ANNOTATIONS` / `HANDOFF_WRITE_ANNOTATIONS` 等几组，标明这个工具会不会改东西。
4. **返回值是结构化的**，不是纯字符串。

### 深入：工具返回值的结构

MCP 工具返回三段：`content`（给模型看的文本/图片）、`structuredContent`（结构化数据，客户端/widget 用）、`_meta`（渲染指令）。codexpro 用一个 `textResult` 助手统一构造：

```ts
function textResult(text, structuredContent = {}, meta = {}) {
  return {
    content: [{ type: "text", text: redactSensitiveText(text) }],
    structuredContent: redactStructured(structuredContent),
    _meta: meta,
  };
}
```

注意两个 `redact*`：**所有出站文本和结构化数据都过一遍敏感信息脱敏**（token、密钥模式等）。这是"桥"产品的 Hygiene 习惯——你永远不知道模型会把哪段输出回贴到云端日志里。脱敏放在出口的单一函数里，而不是散落在每个工具，是正确的位置。

错误也要结构化。codexpro 的包装器 `registerToolCompat` 把 handler 的异常转成 `{ isError: true, content:[...], structuredContent:{error} }`，模型能看懂"这次失败了，失败原因是 X"，而不是连接直接崩掉。

---

## 接进 ChatGPT：Streamable HTTP + 隧道 + Developer Mode

stdio 适合本地调试，但 ChatGPT 云端要的是一个 HTTP 端点。codexpro 的 `src/http.ts` 用 express 实现 MCP 的 `Streamable HTTP` transport。骨架是这样：

```ts
app.post("/mcp", express.json({ limit: "20mb" }), async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport = getTransport(sessionId);

  if (!transport && isInitializeRequest(req.body)) {
    // 新会话：建一个 transport，绑定一个新 McpServer
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => transports.set(id, { transport, ... }),
    });
    const server = createCodexProServer(config);  // ← 同一个工厂函数
    await server.connect(transport);
  } else if (!transport) {
    return res.status(400).json({ /* missing/invalid session id */ });
  }
  await transport.handleRequest(req, res, req.body);
});

app.get("/mcp", handleSessionRequest);    // SSE 回传通道
app.delete("/mcp", handleSessionRequest); // 关闭会话
```

要点：

- **会话即状态**。MCP over HTTP 是有状态的：第一个请求是 `initialize`，服务端生成一个 `Mcp-Session-Id` 回给客户端，后续请求靠这个 header 找回对应的 `transport`。codexpro 用一个 `Map<sessionId, TransportRecord>` 存活跃会话，并带 TTL/上限做 `pruneTransports()`，防止内存泄漏和会话堆积。
- **`POST /mcp` 收请求，`GET /mcp` 是 SSE 回传，`DELETE /mcp` 关会话**——这是 Streamable HTTP 的标准三件套。
- **每个会话一个独立 `McpServer` 实例**，但共享同一个 `config` 和 workspace manager。

### Step：本地起 HTTP server

```bash
npm run build
npm run start:http        # node dist/http.js，监听 127.0.0.1:8787/mcp
curl http://127.0.0.1:8787/healthz
```

### Step：开隧道并连上 ChatGPT

```bash
codexpro start            # 等价于起 http + cloudflare 隧道
# 复制打印出来的 https://<random>.trycloudflare.com/mcp
```

然后在 ChatGPT 里：先到 **Settings → Apps & Connectors → Advanced settings** 打开 Developer Mode，再到 **Settings → Connectors → Create**，粘贴 `https://.../mcp`。连上后发一句"open my workspace and show the file tree"，它就会开始调工具。

> **JS 渲染陷阱（调试时）**：如果你用浏览器或抓包看端点返回，注意 MCP 的 SSE 回传不是普通 JSON 响应。用官方 inspector 或 codexpro 自带的 `scripts/http-smoke.mjs` 来验证，比手搓 curl 靠谱。

---

## 安全边界

到这里功能已经跑通了。但"让云端 AI 操作本地文件系统"是个危险能力，**这一章才是 codexpro 大部分代码都服务于安全，也是这个架构值得学习的精华所在**。

安全模型分四层。

### Workspace 锁定 + 路径逃逸防护（`src/guard.ts`）

核心不变量：**任何文件操作的真实路径必须落在某个 allowed root 之内**。`PathGuard.resolve()` 做了这些检查：

```ts
resolve(workspace, inputPath, { forWrite }) {
  const absPath = path.resolve(path.join(workspace.root, inputPath));
  if (!isSubpath(absPath, workspace.root))          // 1. 拼接后不能跳出根
    throw new CodexProError("Path escapes workspace root");
  this.assertNotBlocked(relPath);                   // 2. 命中黑名单 glob 拒绝
  const realTarget = fs.realpathSync(absPath);      // 3. 跟随软链后再查一遍
  if (realTarget && !isSubpath(realTarget, workspace.root))
    throw new CodexProError("Path resolves outside via symlink");
  // forWrite 时还要检查"最近的已存在父目录"的真实路径
}
```

三个洞它都堵了：
1. `../../etc/passwd` 这类相对路径逃逸；
2. 黑名单目录/文件（见下）；
3. **软链接逃逸**——这是最容易漏的：仓库里一个指向 `~/.ssh` 的符号链接，naive 实现会跟着读出去。`realpathSync` 之后再校验一次根包含关系才安全。写操作还额外检查父目录的 realpath，防止"写进一个软链出去的目录"。

`WorkspaceManager.openWorkspace()` 则保证只有 `allowedRoots` 列表里的目录能被打开，根来源是 `--root` / `CODEXPRO_ROOT`，额外的要显式 `--allow-root`，开整个 home 要显式 `--allow-home`。**默认最小授权**。

`WorkspaceManager.openWorkspace()` 是真正"打开某个目录当工作区"的入口。它有一条硬规则:**请求打开的目录，必须落在 `allowedRoots`(允许根目录白名单)之内,否则直接拒绝。**

而这个白名单默认非常窄——**只包含你启动时指定的那一个根目录**(来源:`--root` 参数或环境变量 `CODEXPRO_ROOT`，都没给就用当前目录)。想让 ChatGPT 能碰更多目录,必须**显式**往白名单里加:

- 加某个额外目录 → 用 `--allow-root <dir>`(可重复);
- 把整个 home 目录都放进来 → 用 `--allow-home`。

不显式加入，codexpro就碰不到。这就是**默认最小授权(least privilege)**。

### 黑名单 globs（默认拒绝敏感路径）

`config.ts` 里写死了一份 `DEFAULT_BLOCKED_GLOBS`，无条件挡掉：

```
.git/**  node_modules/**  .env  .env.*  *.pem  *.key
id_rsa*  id_ed25519*  .ssh/**  dist/**  build/**  .next/**  coverage/**  .cache/**
```

凭据（`.env`、私钥、`.ssh`）、版本控制内部（`.git`）、和大体积噪声目录（`node_modules`、构建产物）一律不让读写。用户还能用 `CODEXPRO_BLOCKED_GLOBS` 追加。匹配用 `minimatch` 且 `dot:true`（默认匹配隐藏文件）。

### 能力开关（modes）—— 把危险功能做成可关的

codexpro 把"能做多少事"拆成几个正交的运行模式，全在 `config.ts` 解析，默认值偏保守：

| 维度 | 取值 | 默认 | 作用 |
| --- | --- | --- | --- |
| `writeMode` | `off` / `handoff` / `workspace` | `workspace` | 是否允许改源文件。`handoff` 只允许写 `.ai-bridge/` 计划目录 |
| `bashMode` | `off` / `safe` / `full` | `safe` | 是否能跑命令，`safe` 走白名单（第 7 章） |
| `toolMode` | `minimal` / `standard` / `full` | `standard` | 暴露哪一组工具 |
| `codexSessions` | `off` / `metadata` / `read` | `off` | 是否能读本地 Codex 历史 |

实现上的关键技巧：**模式不只是运行时拦截，而是直接决定工具是否注册**。

```ts
function shouldRegisterTool(config, name) {
  if (name === "bash" && config.bashMode === "off") return false;
  if ((name === "write" || name === "edit") && config.writeMode !== "workspace") return false;
  // ...
}
```

`writeMode=off` 时，`write`/`edit` 工具**根本不会出现在 schema 里**——模型看不见就不会试。这比"注册了再在 handler 里报错"更干净：减少了模型的诱惑面，也减少了无效往返。`writeMode=handoff` 时还有第二道运行时闸 `assertWriteToolAllowed`，只放行 `.ai-bridge/` 内的写入。**纵深防御**：注册期 + 运行期双保险。

### 网络层鉴权（`src/http.ts`）

回环（`127.0.0.1`）默认不要求 token，但**一旦走隧道或绑非回环地址，就强制要 token**：

```ts
const requireHttpToken =
  boolFrom(env.CODEXPRO_REQUIRE_HTTP_TOKEN) ||
  boolFrom(env.CODEXPRO_TUNNEL_MODE) ||             // 开隧道即视为公网
  (!isLoopbackHost(host) && !allowNoToken);
```

token 校验用**常量时间比较**防计时攻击：

```ts
function tokenMatches(value) {
  const expected = Buffer.from(config.authToken);
  const actual = Buffer.from(value);
  return expected.length === actual.length && timingSafeEqual(expected, actual);
}
```

token 可以走 `Authorization: Bearer` 或查询参数 `?codexpro_token=`。

本地 admin 接口 `/admin/profile` 额外加了**速率限制 + body 大小限制**，用来抗高频滥用和超大负载(DoS/暴力穷举)，而非访问控制。真正限定“谁能改配置”靠的是默认只绑回环 `127.0.0.1` (本机外够不到) 以及可选的 token。注意默认回环配置下若未设 token，本机其他进程仍可访问该接口——codexpro 在此假设本机可信。

> **设计取舍点**：codexpro 选择"回环免 token"是为了本地开发体验，赌的是"能在你 `127.0.0.1` 上发请求的进程已经在你机器里了"。如果你的威胁模型包含本机恶意进程，应该把 `CODEXPRO_REQUIRE_HTTP_TOKEN=1` 设成强制。

---

## 让 ChatGPT 能改代码：`write` / `edit` / `show_changes`

读懂了安全层，写工具就是水到渠成。`edit` 做的是精确字符串替换（类似你在 IDE 里的查找替换），`write` 整体写文件，两者都先过 `PathGuard.resolve(..., { forWrite:true })` 和 `assertWriteToolAllowed`。

真正值得学的是**改完之后的复盘工具** `show_changes`。它把"git status + diff stats + unified diff"合并成一次调用：

```ts
registerCodexTool(config, server, "show_changes", {
  description: "Summarize current changes... Use this instead of bash git status/diff.",
  // ...
}, async (args) => {
  const status = gitStatus(config, workspace, guard, scopedPath);
  const diff = normalizeGitOutput(gitDiff(config, guard, workspace, scopedPath, staged));
  const stats = diffStats(diff);  // 数 +/- 行
  const text = `# Show Changes\n\n## Changed\n${changedText}\n\n## Diff stats\n+${stats.additions} -${stats.deletions}${diffBlock(diff)}`;
  return textResult(text, { changed_files, additions: stats.additions, deletions: stats.deletions, diff });
});
```

为什么要专门做这个工具，而不让模型自己 `bash git diff`？三个原因，每个都很实际：

1. **少一次往返**。模型改完直接拿到 status+diff+统计，不用连发三条 bash。
2. **可控的预览体积**。`previewText()` 把 diff 截到 40 行 / 12KB，不会让一个大改动撑爆上下文。
3. **结构化数据喂给 UI 卡片**（第 8 章），原生的 `git diff` 做不到。

这体现了一个原则：**给模型"任务级"工具，而不是"shell 级"原语**。你越是把常见意图（看改动、开工作区、搜代码）打包成一个高层工具，模型用得越准、越省 token，你也越能在中间插入安全检查和体积控制。

---

## 安全地跑命令：`bash` 的 `safe` 白名单

`bash` 是威力最大也最危险的工具。codexpro 默认 `bashMode=safe`，用**前缀白名单 + 模式黑名单**双重过滤（`src/bashOps.ts`）。

允许的前缀只覆盖"验证类"命令：

```ts
const SAFE_ALLOWED_PREFIXES = [
  "pwd", "ls", "find",
  "git status", "git diff", "git log", "git show", "git branch", "git ls-files",
  "npm test", "npm run build", "npm run lint", "npm run typecheck",
  "pnpm test", "yarn test", "bun test",
  "pytest", "go test", "cargo test", "cargo check",
  "tsc", "eslint", "biome check", // ...
];
```

同时一票否决一堆危险模式：

```ts
const SAFE_BLOCKED_PATTERNS = [
  /(^|\s)rm\s+/, /(^|\s)mv\s+/, /(^|\s)sudo\s+/, /(^|\s)chmod\s+/,
  /(^|\s)curl\s+/, /(^|\s)wget\s+/, /(^|\s)ssh\s+/,            // 禁外联
  /(^|\s)git\s+push\b/, /(^|\s)git\s+reset\b/, /(^|\s)git\s+checkout\b/, // 禁破坏性 git
  /[;&|<>`]/, /\$\(/, /\n/,                                    // 禁命令拼接/替换/换行
  /\$(?:[A-Za-z_]\w*|\{|\[)/,                                  // 禁变量展开
  /(^|\s)(\/|~(?:\/|\s|$))/, /(^|\s)\.\.(?:\/|\s|$)/,          // 禁绝对路径/家目录/上跳
  /(\.env|\.git|node_modules|\.ssh|id_rsa|id_ed25519|.*\.(?:pem|key))/, // 禁敏感目标
  /(^|\s)(cat|grep|rg|head|tail|wc)\s+/,                       // 故意不让 bash 读文件
];
```

几个有意思的判断：

- **`safe` 模式刻意禁掉 `cat`/`grep`/`head` 等读文件命令**。因为读文件应该走 `read`/`search` 工具——它们有体积上限、有黑名单、有脱敏。让 bash 去 `cat` 等于绕过整套文件安全层。这跟 server instructions 里"不要用 bash 读文件"是一致的。
- **禁 `;` `|` `&` `$()` 反引号和换行**，就是不让命令拼接/子 shell。白名单一旦能被 `git status; rm -rf .` 这种拼接绕过就形同虚设。
- **禁变量展开 `$VAR`**，避免 `$HOME` 之类把路径间接拼出去。
- 还有 `bashSession` 守卫（可选）：`requireBashSession` 打开后，每条 bash 必须带匹配的 `session_id`，防止跨会话误用。

> **`full` 模式存在但默认不开**。这是给"我清楚风险、在隔离环境里"的高级用户的逃生舱。产品默认值应该是保守的那个——codexpro 把默认放在 `safe`，是对的。

Codexpro这套设计背后的原则是：白名单设计的核心心法是**默认拒绝 + 只放行可枚举的安全意图 + 显式封掉所有元字符**。不要试图用黑名单去"列举所有危险命令"，那是列不完的。

---

## Apps SDK 工具卡片（Widget）

到这一步插件已经能干活了。这一章是锦上添花：让工具结果在 ChatGPT 里渲染成一张**卡片**，而不是一段 Markdown。

机制分两半：

**(1) 注册一个 UI 资源**（HTML 模板），`src/server.ts` 里的 `registerToolCardResource`：

```ts
server.registerResource("codexpro-tool-card", TOOL_CARD_URI, {
  title: "CodexPro Tool Card",
  mimeType: TOOL_CARD_MIME_TYPE,           // text/html+... 的 widget mime
}, async () => ({
  contents: [{
    uri: TOOL_CARD_URI,
    mimeType: TOOL_CARD_MIME_TYPE,
    text: toolCardWidgetHtml,              // 一整页自包含 HTML（src/toolCardWidget.ts）
    _meta: {
      "openai/widgetDescription": "Renders workspace orientation, diffs, handoffs...",
      "openai/widgetPrefersBorder": true,
      "openai/widgetDomain": config.widgetDomain,
      "openai/widgetCSP": { connect_domains: [], resource_domains: [] },
    },
  }],
}));
```

**(2) 让工具指向这个模板**：在工具的 `_meta` 里声明模板 URI。codexpro 用 `toolCardMeta()` 助手同时给出 MCP Apps 标准键和 ChatGPT 兼容键：

```ts
function toolCardMeta() {
  return {
    ui: { resourceUri: TOOL_CARD_URI },        // MCP Apps 标准键
    "openai/outputTemplate": TOOL_CARD_URI,    // OpenAI 兼容键
  };
}
```

于是 ChatGPT 调用 `show_changes` 时，拿到 `structuredContent`（changed_files、additions、diff 等）+ 模板 URI 指向的 HTML，就把这堆数据塞进卡片里渲染。卡片 HTML 是一整页自包含的、读 `window.openai` 注入数据的脚本。

按当前 Apps SDK 文档，有两条要点：

- **只有"渲染型"工具该带 UI 模板指针**；纯数据工具不要带。
- 新插件优先用 MCP Apps 标准键，比如 `_meta.ui.resourceUri` 关联模板、`_meta.ui.csp` 描述资源策略；`openai/outputTemplate`、`openai/widgetCSP` 这类 `openai/*` 字段可以作为 ChatGPT 兼容/扩展键叠加。codexpro 两套都照顾到，是为了最大化客户端兼容。

codexpro 还把卡片做成**可关的**：`config.toolCards=false`（默认）时，`descriptorOptionsForConfig` 会把所有 `openai/*`、`ui` 等渲染 `_meta` 从工具描述里剥掉。**渐进增强**：没有卡片支持的客户端，工具照样能用文本结果工作。

> Widget 是体验加分项，不是功能必需。先把工具和安全做对，再加卡片。

---

## 进阶模式：Handoff 与 Pro Context（按需）

codexpro 还有两个面向特定工作流的能力，了解概念即可，不必一开始就做。

**Handoff（交接）**：当 `writeMode=handoff` 时，ChatGPT 不直接改源码，而是把一份实现计划写进 `.ai-bridge/` 目录，再由你本地的实现 agent（Codex / OpenCode / 自定义）去执行。`handoff_to_agent` 生成带"实现契约"的计划文件（要求小步提交、跑验证、回写状态到 status 文件）。CLI 的 `codexpro execute-handoff` / `watch-handoff` / `loop-handoff` 负责把计划喂给本地 agent 并回收 diff。这套适合"ChatGPT 负责规划、本地模型负责落地"的分工。

**Pro Context（上下文导出）**：给那些**不能调 MCP 工具**的场景（比如只能粘文本的模型）。`export_pro_context` 把选中的文件打成一个 `pro-context.md` 包，人工贴过去。`src/proContext.ts` 里同样有体积上限和"只导出选中文件"的约束。

两者共同点：**都不引入新的安全边界例外**——handoff 写入仍限定在 `.ai-bridge/`，pro context 仍走同一套 guard 和脱敏。新功能不破坏已有不变量，这是好架构的标志。

---

## 测试与发布

### Smoke tests（端到端冒烟）

codexpro 不太依赖单测，而是用一组**冒烟脚本**起真 server、走真协议握手、断言工具行为（`scripts/*.mjs`）：

```bash
npm run smoke
# = smoke(stdio) + http-smoke + pro-smoke + doctor-smoke + settings-smoke + execute-handoff-smoke
```

`scripts/smoke.mjs` 自己实现了一个极简的 stdio MCP 客户端（拼 JSON-RPC、读帧、配对 id），拉起 `dist/stdio.js`，发 `initialize` → `tools/list` → `tools/call`，断言返回。**这种"用最小客户端打真服务端"的冒烟测试，对协议型项目性价比极高**：它能抓住 transport 帧、schema 结构、模式过滤这些单测覆盖不到的集成问题。CI（`.github/workflows/ci.yml`）就跑 build + smoke。

每加一个工具，在对应 smoke 脚本里加一条"调用它、断言返回形状"的用例。

### 打包发布（npm bin）

`package.json` 暴露三个可执行入口：

```json
"bin": {
  "codexpro": "scripts/codexpro.mjs",      // 用户 CLI
  "codexpro-mcp": "dist/stdio.js",         // stdio 入口
  "codexpro-mcp-http": "dist/http.js"      // http 入口
}
```

`prepack` 钩子跑 `npm run build`（`tsc`）保证发布的是编译产物，`files` 字段只发 `dist`/`docs`/`scripts` 和文档。用户 `npm i -g codexpro` 后 `codexpro` 命令就可用。

---

## 全局架构回顾

把所有文件按职责对照一遍（这也是你照搬时的"复制清单"）：

| 文件 | 职责 | 抄它的理由 |
| --- | --- | --- |
| `src/config.ts` | 解析 env/argv → 强类型 `CodexProConfig`，定义所有 mode 与默认值 | **单一配置入口** + 保守默认 |
| `src/guard.ts` | `WorkspaceManager`（根锁定）+ `PathGuard`（逃逸/软链/黑名单）| 文件系统安全的全部不变量 |
| `src/server.ts` | `createCodexProServer`：建 `McpServer`、按 mode 注册工具、写 instructions | 传输无关的业务核心 |
| `src/stdio.ts` | stdio 入口 | 调试用，~15 行 |
| `src/http.ts` | Streamable HTTP transport + express + token 鉴权 + 会话管理 + 本地 admin UI | 接 ChatGPT 的生产入口 |
| `src/bashOps.ts` | bash `safe` 白名单/黑名单 | 命令执行安全 |
| `src/fsOps.ts` / `searchOps.ts` / `gitOps.ts` | read/write/edit/tree、搜索、git 封装 | 文件与 VCS 原语 |
| `src/redact.ts` | 出站脱敏 | 放在结果出口的单点 |
| `src/toolCardWidget.ts` | Apps SDK 卡片 HTML | UI 增强 |
| `src/proContext.ts` / `codexSessions.ts` / `capabilitiesOps.ts` | handoff/pro/技能发现 | 进阶工作流 |
| `scripts/*.mjs` | CLI 启动器 + 冒烟测试 | 用户体验 + 集成测试 |

**codexpro 完整的工作流总结如下**：`config` 决定能力 → `createCodexProServer` 按能力注册工具并写 instructions → transport（stdio/http）接客户端 → 每个工具调用都过 `guard` 校验、执行、`redact` 脱敏后返回，可选地带上 widget 渲染元数据。

---

## 自己造插件的检查清单

如果你要做一个类似的"让 AI 访问某个本地资源"的 MCP 插件，按这个顺序走：

1. **先 stdio 跑通一个只读工具**，用一个最小客户端脚本冒烟，建立反馈循环。
2. **把传输层和业务解耦**：一个 `createServer(config)` 工厂，stdio/http 共用。
3. **配置集中化、默认保守**：所有 mode 在一个 `loadConfig` 里解析，危险能力默认关或走白名单。
4. **安全边界用代码画出来**，别指望模型自觉：路径逃逸（含软链）、黑名单凭据、能力开关（注册期 + 运行期双闸）、出站脱敏。
5. **危险能力做成"可关 + 不注册"**：关掉的工具不进 schema，减少诱惑面。
6. **给模型任务级工具**（`show_changes`），而不是 shell 原语，顺便在中间插安全检查和体积上限。
7. **接 ChatGPT 才需要 HTTP + 隧道 + token**，非回环一律强制 token、常量时间比较。
8. **widget 最后做**，且做成渐进增强。
9. **冒烟测试打真协议**，每个工具一条用例，进 CI。

把"功能"和"边界"当成同等重要的两件事——对这类能碰本地文件系统的插件，**边界就是产品**。

---

### 参考资料

- OpenAI Apps SDK — Build your MCP server：https://developers.openai.com/apps-sdk/build/mcp-server
- OpenAI Apps SDK — Build your ChatGPT UI（widget / outputTemplate）：https://developers.openai.com/apps-sdk/build/chatgpt-ui
- OpenAI Apps SDK — Connect from ChatGPT（连接器/隧道/`/mcp`）：https://developers.openai.com/apps-sdk/deploy/connect-chatgpt
- OpenAI Help — Developer mode and MCP apps in ChatGPT：https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt
- 本仓库源码：`src/config.ts`、`src/guard.ts`、`src/server.ts`、`src/http.ts`、`src/bashOps.ts`、`design.md`
