# dsh-retrieval-guard

一个 **DeepSeek Harness（DSH）插件**：把 AGENTS.md 里的「铁律一：优先检索现成轮子」从
**靠模型自觉** 变成 **harness 层的强制拦截**。

- 触发时机：`tools/pre-execute`（每次工具调用前，`prepend` 最前）
- 粒度：**每轮**（最近一次 `user/message` 之后）而不是整个会话——上一轮检索过，本轮没检索照样拦
- 拦截方式：返回 `{ kind: "ask" }` 弹确认框，模型无法绕过（不靠提示词自觉）

## 它做什么

| 层 | 行为 | 触发条件 |
|---|---|---|
| 1. 检索门控 | 本轮还没调用过检索工具时，拦截两类动作，弹框让用户决定「允许跳过 / 先检索」 | ① `write`/`edit` 目标为代码/脚本/skill 文件（`.py/.js/.ts/.sh/.ps1/...`、`SKILL.md`、`AGENTS.md`、`persona.md`）；② `bash`/`pwsh` 命令命中装依赖/拉新（`npm/pip/git clone/cargo/npx create/pnpm add/...`） |
| 2. 安全检测（可选融合） | 调 `guardian/check` 检查危险命令、明文密钥、SSRF、提示注入；命中弹框强制确认 | 每次工具调用前；**dsh-guardian 未安装时静默跳过** |
| 3. 观测告警（只记日志，不阻断） | 文字问决策却没弹 `ask_user_question`；提示重启却没附日志路径；`Get-Content G:\KEY`/直接 `read` token 文件 | `session/event` 流 |
| 4. 附带工具 | `scripts/dsh-replay-web.mjs` + `scripts/launch-dsh-replay.cmd`：DSH 会话回放 web 查看器（只读） | 手动运行 |

> `scripts/` 里的回放查看器是独立项目 [launch-dsh-replay](https://github.com/hpsks416/launch-dsh-replay)
> 的前身；那个项目现在支持桌面端 v4 会话、离线单文件 HTML 与一键安装，本插件只保留一份脚本副本。

## 安装

桌面端 / CLI 的 profile（以 `desktop` 为例，路径换成你本机）：

```jsonc
// %USERPROFILE%\.dsh\profiles\desktop\package.json
"dependencies": {
  "dsh-retrieval-guard": "link:G:/CodexDS/DSH/INTRO/dsh-retrieval-guard"
},
"dsh": { "profile": { "bundles": [ /* ... */, "dsh-retrieval-guard" ] } }
```

```powershell
cd $env:USERPROFILE\.dsh\profiles\desktop
pnpm install --ignore-scripts --no-frozen-lockfile
# 重启桌面端 DSH 后生效
```

## 配置（可选）

插件 `apply(ctx, config)` 支持：

| 配置项 | 默认 | 说明 |
|---|---|---|
| `retrievalTools` | `["web_search", "open-source-scout"]` | 算作「已检索」的工具名集合 |

## 覆盖范围与边界

- **只拦「写代码 / 装依赖 / 克隆 / 新建项目」**；写 `.md` 笔记、普通读取、检索工具本身都放行。
- **不拦**：没有会话上下文（`agent/session` 缺失）时放行，避免误伤。
- 检测失败只 `warn` 日志，**绝不打断会话**（fire-and-forget 原则）。
- 第 3 层是**观测**不是拦截：只写 `logger.warn`，供事后排查（例如「又没弹窗就问决策」）。
- 它不是安全边界：真正的安全拦截依赖可选的 `dsh-guardian`；本插件只做「先检索」行为门控 + 告警。

## 兼容性

- 当前 `package.json` 兼容声明：`^0.1.5-rc.2 || ^0.2.2-rc.1`（同时覆盖旧 CLI profile 与桌面端 0.2.2-rc.1）。
- 桌面端 `dsh-desktop 0.2.2-rc.1` 实测可用（插件通过 `link:` 装入 profile）。

## 版本历史（摘要）

- `feat` 门控插件（write/edit + 安装命令，轮次判定）
- `feat` 并入会话回放 web 查看器到 `scripts/`
- `feat` 第 3 层：直接读 `G:\KEY` token 的非插件式检测
- `fix` 门控弹框文案（消除二重否定）
- `feat` 融合 dsh-guardian 通用安全检测（`pre-execute` 前置调 `guardian/check`）
- `fix` 回放查看器缓存按日志 mtime 失效
- `docs` 补 README / LICENSE，兼容桌面端 0.2.2-rc.1

## 许可

MIT，见 [LICENSE](LICENSE)。
