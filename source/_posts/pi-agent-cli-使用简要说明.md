---
title: Pi Agent CLI 使用简要说明
date: 2026-11-10 16:46:00
tags:
---

[Pi](https://pi.dev) 是由 Earendil Works（Mario Zechner）开发的终端编程代理（coding agent）。它在本地终端中工作，默认提供 `read`、`write`、`edit`、`bash` 四个工具，让模型直接阅读和修改文件、执行命令。设计理念是核心保持精简，通过扩展、Skills、提示模板和主题来适配你的工作流，而不是反过来要求你迁就工具。

官方仓库：[earendil-works/pi](https://github.com/earendil-works/pi)，文档见 [pi.dev](https://pi.dev)。

## 安装

macOS 或 Linux 推荐使用官方安装脚本（会固定依赖版本，之后用 `pi update` 升级）：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

Windows：

```powershell
powershell -c "irm https://pi.dev/install.ps1 | iex"
```

也可以用 npm 安装（需要 Node.js 22.19 或更高版本；这种方式不会固定传递依赖）：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

安装完成后可用 `pi --version` 验证。

## 快速开始

进入你希望 Pi 工作的目录，然后启动：

```bash
cd /path/to/project
pi
```

工作目录会用于发现项目文件和配置，并按目录分组保存会话。

在交互界面中输入 `/login`，选择供应商，按提示使用订阅或保存 API Key。如需更换模型，再运行 `/model`。

之后直接用自然语言下达任务即可。输入 `@` 可以搜索并引用文件，例如：

```text
Explain how this repository is structured and how to run its checks.
```

Pi 会展示它读取的文件、执行的命令和编辑操作。需要注意：它默认不会在每次调用工具前征求确认。重要工作请配合版本控制；不受信任或无人值守的任务，建议在容器或其他沙箱中运行。

## 常用命令与选项

默认进入交互模式。也可以一次运行后退出，或把结果交给脚本：

```bash
pi --print "Summarize this repository"
git diff | pi --print "Review this change"
pi --continue
```

- `-p` / `--print`：处理提示后，把最终回复打印到标准输出并退出
- `-c` / `--continue`：继续当前目录最近一次会话
- `-r` / `--resume`：打开会话选择器，恢复指定会话
- `--model <name>`：按 ID 或模糊匹配选择模型，可用 `provider/id` 形式
- `--provider <name>`：限制模型查找的供应商（需同时指定 `--model`）
- `--no-session`：使用内存会话，不落盘

交互模式下常用斜杠命令：

- `/login`、`/logout`：添加或移除供应商认证
- `/model`：选择模型
- `/thinking`：设置思考级别
- `/new`：开始新会话
- `/resume`：恢复其他已保存会话
- `/name`：给当前会话命名
- `/settings`：打开设置
- `/quit`：退出

退出后若想接着干，在同一目录执行 `pi --continue` 即可。

## 配置模型与 API Key

最简单的方式是在 Pi 里运行 `/login`，按向导用浏览器登录订阅或保存 API Key。凭据会写入 `auth.json`，请勿提交到版本库。也可以在会话外执行 `pi auth login <provider>` 完成登录。

如果不想把密钥写入本地文件（例如 CI 环境），可以在启动前设置对应的环境变量：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

常见变量还包括 `OPENAI_API_KEY`、`GEMINI_API_KEY`、`OPENROUTER_API_KEY`、`DEEPSEEK_API_KEY` 等，完整列表见[官方 Providers 文档](https://pi.dev/docs/latest/providers)。

命令行也可临时指定模型和思考级别：

```bash
pi --model sonnet:high
```

`:` 后面是思考级别（如 `high`）。也可用 `--thinking` 单独设置，取值包括 `off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`，实际可用范围取决于模型能力。

## 几点提示

1. 会话会自动保存。用 `/fork`、`/clone` 或 `/tree` 可以在不丢失现有工作的前提下探索其他方案；上下文过长时可用 `/compact` 压缩。
2. 在编辑器里以 `!` 开头可运行终端命令并把输出纳入对话，例如 `!git status`；`!!` 则只执行命令，不把输出发给模型。
3. 可通过 Skills、提示模板、TypeScript 扩展和 Pi Packages 增强能力。项目目录下的 `AGENTS.md` 可作为该文件夹的持久说明。
4. 用官方安装脚本安装的版本，升级执行 `pi update`；用 npm 安装则相应更新该 npm 包。
5. 默认开启的工具是 `read`、`bash`、`edit`、`write`。可用 `--tools` 调整，例如 `pi --tools +codemode`。

更多细节请参考官方文档：[快速开始](https://pi.dev/docs/latest/quickstart)、[命令行](https://pi.dev/docs/latest/cli)、[斜杠命令](https://pi.dev/docs/latest/slash-commands)。
