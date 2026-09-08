# OpenCode v1.18.27

OpenCode 是一个开源的 AI 编码助手，运行在终端中，可以帮助你阅读代码、编写功能、执行重构等。它支持任意 LLM 提供商，并支持扩展规则、自定义命令（提示词）、MCP 服务器和 Skill。

- 官方文档：https://opencode.ai/docs
- 下载地址：https://opencode.ai/zh/download

## 1. 安装

### 1.1 环境要求

- 桌面版无需额外环境；若使用 `npm` 方式安装则需要 Node.js 16+（可用 `node -v` 检查）

### 1.2 安装方式

打开下载页，选择 `Windows (x64) 桌面版 OpenCode Desktop Installer` 下载安装。

也可通过命令行方式安装：

```powershell
npm install -g opencode-ai
```

### 1.3 验证安装

```powershell
opencode --version
```

## 2. 配置

| 范围 | 路径 |
| --- | --- |
| 全局配置 | `~/.config/opencode/opencode.json` 或 `opencode.jsonc`（允许注释） |
| 项目配置 | 项目根目录 `opencode.json` 或 `opencode.jsonc`（允许注释） |

以 `jsonc` 格式为例的示例配置

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // MCP 服务器配置
  "mcp": {
    "db-analyzer": {
      "type": "local",
      "command": ["python", "-m", "nexusdb.server"],
      "enabled": true
    },
    "my-mcp-server": {
      "type": "remote",
      "url": "http://localhost:8080/mcp",
      "enabled": true
    }
  },

  // 插件列表
  "plugin": [
    "codex-search-opencode"
  ]
}
```

## 3. 高级

### 3.1 Rules

规则写在 `AGENTS.md` 中，内容会写入 LLM 的上下文，用于约束它在当前项目中的行为。

**存放位置**：

| 范围 | 路径 |
| --- | --- |
| 全局规则 | `~/.config/opencode/AGENTS.md`（对所有会话生效，未被提交到 Git，适合放个人偏好） |
| 项目规则 | 项目根目录 `AGENTS.md`（启动时从当前目录向上遍历至 Git 根目录查找） |

**示例**：在项目根目录创建 `AGENTS.md`，仅在该项目生效

```markdown
## 文档创建规则
当用户要求“创建需求文档”或类似表述时，请自动在 `D:\code\老系统\开发记录` 目录下创建文件。
如果用户没有指定文件名，请根据需求内容生成一个合适的文件名，并增加前缀使用YYYY-MM-DD_。
```

### 3.2 提示词（prompt）

通过自定义命令（command）把常用的提示词固化为快捷键命令，输入 `/命令名` 即可触发。

**方式一：Markdown 文件**

放在 `.opencode/commands/`（项目）或 `~/.config/opencode/commands/`（全局），文件名即命令名。

`.opencode/commands/review.md`：

```markdown
---
description: 代码评审
agent: build
---

请对本次修改做代码评审，重点关注：
1. 是否有潜在 bug 和安全隐患
2. 是否符合项目代码规范
3. 是否有性能问题
最后给出修改建议。
```

**方式二：JSON 配置**：

```jsonc
{
  "command": {
    "component": {
      "description": "创建 React 组件",
      "template": "创建一个名为 $ARGUMENTS 的 React 组件，使用 TypeScript 并给出基础结构。"
    }
  }
}
```

**使用步骤**：

1. 打开项目的 `opencode.json`（没有就新建，即第 2 节中的配置），把上面的 `command` 字段加到配置文件里
2. 保存并**重启 OpenCode**（配置只在启动时加载）
3. 在 TUI 中输入 `/component Button`

此时 OpenCode 会把 `template` 中的 `$ARGUMENTS` 替换为 `Button`，等价于发送提示词："创建一个名为 Button 的 React 组件，使用 TypeScript 并给出基础结构。"

**两种方式的对照**：

| 方式一（Markdown 文件） | 方式二（JSON 配置） |
| --- | --- |
| 文件名 `review.md` = 命令名 | JSON 键名 = 命令名 |
| `description` 头 | `description` 字段 |
| 正文 = 提示词 | `template` 字段 = 提示词 |

两种方式选其一即可：方式一每个命令一个文件，命令较多时更好维护；方式二全部集中在配置文件中。

模板支持的特殊语法：

| 语法 | 说明 | 示例 |
| --- | --- | --- |
| `$ARGUMENTS` | 命令后面输入的全部参数 | `/component Button` |
| `$1` `$2` ... | 第 N 个参数 | `/create-file config.json src` |
| `` !`命令` `` | 把 shell 命令输出注入提示词 | `` !`git log --oneline -10` `` |
| `@文件路径` | 把文件内容加入提示词 | `请评审 @src/Button.tsx` |

**内置命令**：`/init`、`/undo`、`/redo`、`/share`、`/help`。自定义命令同名时会覆盖内置命令。

### 3.3 MCP

MCP（Model Context Protocol）用于给 OpenCode 接入外部工具，配置在 `opencode.json` 的 `mcp` 字段下，分为本地和远程两种。

**本地 MCP**（命令所需的运行时需先安装，如 Node.js、Python）：

```jsonc
{
  "mcp": {
    "db-analyzer": {
      "type": "local",
      "command": ["python", "-m", "nexusdb.server"],
      "enabled": true
    }
  }
}
```

**远程 MCP**：

```jsonc
{
  "mcp": {
    "my-mcp-server": {
      "type": "remote",
      "url": "http://localhost:8080/mcp",
      "enabled": false
    },
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "oauth": {}
    }
  }
}
```

**OAuth 认证与调试**：

```powershell
# 查看所有 MCP 及认证状态
opencode mcp list
# 对某个远程 MCP 进行 OAuth 认证（会打开浏览器）
opencode mcp auth sentry
# 调试连接
opencode mcp debug sentry
# 清除认证信息
opencode mcp logout sentry
```

启用后，在提示词中直接说要使用该 MCP 即可，例如：`使用 db-analyzer 分析当前项目的数据库表结构`。

注意事项：
- `command` 是数组形式，不要写成字符串
- 本地 MCP 可设置 `enabled: false` 临时停用
- MCP 工具会占用上下文，不要启用太多
- 也可以在 `tools` 中按前缀禁用某个 MCP：`"db-analyzer_*": false`

### 3.4 Skills

Skills 是可供模型按需加载的复用指令集，通过内置 `skill` 工具读取，目录结构为固定的 `SKILL.md`。

**存放位置**：

| 范围 | 路径 |
| --- | --- |
| 全局配置 | `~/.config/opencode/skills/<名称>/SKILL.md` |
| 项目配置 | 项目根目录 `.opencode/skills/<名称>/SKILL.md` |

**示例**：`.opencode/skills/git-release/SKILL.md`

```markdown
---
name: git-release
description: 创建规范的版本发布与变更日志，用于准备发布 tagged release 时使用。
---

## 职责
- 根据已合并的 PR 起草发布说明
- 建议版本号递增方案
- 给出可直接执行的 `gh release create` 命令

## 使用时机
当用户准备发布版本时使用；版本方案不明确时先提问澄清。
```

**Frontmatter 约定**：
- `name`（必填）：小写字母、数字、单词间用单个 `-` 分隔，必须与所在目录名一致
- `description`（必填）：1-1024 字符，说明"做什么"和"何时使用"，模型靠它判断是否加载
- 可选：`license`、`compatibility`、`metadata`

**排查 Skill 不生效**：
1. 文件名必须是大写的 `SKILL.md`
2. frontmatter 必须包含 `name` 和 `description`
3. 所有位置的 skill 名称必须唯一
4. 被 `deny` 的 skill 对模型不可见

> 所有的规则、命令、MCP、Skill 修改后，都需要重启 OpenCode 才能生效。