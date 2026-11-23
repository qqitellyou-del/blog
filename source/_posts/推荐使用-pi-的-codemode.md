---
title: 推荐使用 Pi 的 Codemode
date: 2026-11-24 01:12:00
tags:
  - Pi
  - Codemode
---

在上一篇[《Pi Agent CLI 使用简要说明》](/blog/2026/11/10/pi-agent-cli-使用简要说明/)里，我们介绍了 [Pi](https://pi.dev) 的安装、会话和常用命令，文末也提到可以用 `pi --tools +codemode` 打开 Codemode。这篇文章专门说明它是什么、为什么值得打开，以及使用时要注意什么。

Codemode 是 Pi 的内置工具，从 [Pi 0.99.0](https://pi.dev/changelog/releases/0.99.0) 起提供。开启后，模型可以写一段 JavaScript，在 QuickJS 沙箱里调用 Pi 的其他工具，必要时还能运行分类器、图像模型等非对话模型。官方文档写得很明确：**只有脚本的输出会回到模型**。因此脚本可以并行发起多次调用，并在结果进入上下文之前先过滤、汇总。

## 为什么值得用

**一次脚本完成多步编排。** Earendil 把 Codemode 称为在 harness 一侧编排、协调工具调用的机制：用 JavaScript 决定调用顺序，并把多个工具的结果组合起来。循环、分支、并发都可以写在脚本里，不必每一步都再绕回模型再决定下一步。

**并行调用，少跑几轮对话。** 脚本里可以用 `Promise.allSettled()` 同时发起多个工具调用，并保留其中成功的结果。读一批文件、跑几条互不依赖的命令，不必一轮一轮地等模型看到上一次结果后再继续。

**先处理结果，再交给模型，有助于节省上下文。** 官方说明里，脚本可以在模型看到之前过滤掉大量输出。一个具体差别是：模型直接看到的 `bash` 结果通常限制在 2000 行或 50KB，而 Codemode 脚本拿到的 `output` 最多可达 1 MiB（更长的内容会截取首尾各 512 KiB，并给出完整输出的临时文件路径）。需要的时候，脚本只 `return` 或 `text()` 一段摘要即可。

**不必先配置 MCP 也能受益。** 官方 CLI 文档专门指出，即使没有 MCP，Codemode 也有用：并行调用、过滤输出，以及通过 `models.classify()`、`models.generateImages()` 使用分类器和图像模型。配置了 MCP 时，Pi 默认会自动启用 Codemode，方便在脚本里组合这些工具；若只想单独打开它，把它加进默认工具即可。

**中间状态可以留下来。** `store()` / `load()` 能在多次 Codemode 调用之间保存小型 JSON（例如 ID、游标、摘要）。成功执行的脚本会把这些值写入会话，恢复会话或从某条分支继续时仍然可用。Earendil 在[《“You Said No MCP!”》](https://earendil.com/posts/you-said-no-mcp/)里用 Linear 议题加上 Jev 分类器做过一次演示：脚本并发拉取评论、分类情绪，再把结果存起来，后续不必重新拉取。

工具很多时，脚本还可以通过 `searchTools()`、`describeTool()` 等按需查找未直接列在描述里的工具（包括默认以 `codemode` 暴露方式接入的 MCP 工具），避免把整份工具清单都塞进上下文。

## 如何开启

单次启动：

```bash
pi --tools +codemode
```

这会保留默认的 `read`、`bash`、`edit`、`write`，并加上 `codemode`。若希望每次会话都开启，在 `~/.pi/agent/settings.json` 或项目的 `.pi/settings.json` 里写入：

```json
{
  "defaultTools": ["+codemode"]
}
```

你也可以直接让 Pi 帮你改配置。需要模型尽量只通过脚本调用其他工具时，可以把设置里的 `codemode.mode` 设为 `"only"`（默认是 `"on"`，已声明的工具仍会直接暴露给模型）。

日常使用时，你仍然用自然语言下任务，不必自己写脚本。模型会把脚本作为 `codemode` 工具的输入交给沙箱执行。下面这段展示的是官方 API 的典型写法：并行读取几个文件，只把各自开头一段交给模型。

```js
const paths = ["package.json", "README.md", "tsconfig.json"];
const results = await Promise.allSettled(
  paths.map((path) => tools.read({ path }))
);
const summary = {};
for (let i = 0; i < paths.length; i++) {
  const r = results[i];
  summary[paths[i]] =
    r.status === "fulfilled"
      ? String(r.value).slice(0, 400)
      : String(r.reason);
}
return summary;
```

脚本在异步函数体中运行，因此可以使用顶层 `await` 和 `return`。沙箱没有 Node API、文件系统、网络和定时器，对外只能通过 `tools` 和 `models`。

## 使用时要注意

Codemode 里的工具调用是真实发生的。脚本中途失败时，已经执行的调用不会被撤销；仍在进行的调用会被取消，未 await 的 Promise 会被丢弃。重要操作仍然建议在版本控制或沙箱环境中进行。

此外还有一些硬限制：单个脚本的虚拟机内存为 256 MB；脚本不能再启动另一个 `codemode` 脚本；同一 Pi 进程里默认最多同时有两个 Codemode worker。输出过长时，文档建议用工具把数据写到文件，而不是全部打印回模型。

如果任务只是读一个文件或改一两处代码，直接使用默认工具通常就够了，不必强求走脚本。Codemode 更适合需要多步组合、并行处理，或先过滤大量输出的场景。

## 结语

Codemode 并没有改变 Pi「核心保持精简」的设计，只是给模型多了一种编排工具的方式：用代码把多次调用、过滤和汇总放在一次脚本里完成，再把真正有用的结果送回对话。无论你是否使用 MCP，都值得打开试试。安装和基本用法见[《Pi Agent CLI 使用简要说明》](/blog/2026/11/10/pi-agent-cli-使用简要说明/)；完整接口、限制和设置请以官方文档为准：[Codemode](https://pi.dev/docs/latest/codemode)、[命令行（如何启用）](https://pi.dev/docs/latest/cli)、[设置参考](https://pi.dev/docs/latest/settings)。
