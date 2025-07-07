---
title: 简单读懂Agent、MCP、Prompt、FunctionCaling
draft: false
tags:
  - 个人
date: 2025-05-06
---
 
# 📝 最简单的一问一答

`open ai` 刚推出时只有一个最简单的聊天框，采用一问一答的形式。用户向 `chatgpt` 的提问词就叫做 `user prompt` （用户提示词）。

**但**，现实生活中哪怕是相同的问题去问不同职业的人会有不同的答案。比如，我有点头痛这个问题去问医生，医生会给出一定的诊断建议；去问女朋友，则可能会让你滚。同样的问题不同的答案造成这样的结果是由于“人设”的不同。 `ai` 中并没有人设这一概念，于是我们就希望给 `ai` 增加人设，最最直接的方法就是在提问的时候将 人设 + `user prompt` 打包发给 `ai`。举个例子：

```markdown
"你扮演我的女朋友，我现在头疼"
```

“你扮演我的女朋友”这类人设提示词总是在 `user prompt` 中显得有些突兀，于是人们将“人设”这一概念提为 `system prompt` 。

`system prompt` 主要用来描述 ai 的**角色**、**性格**、**背景信息**、**语气**等等。每次用户输入 `user prompt` 时系统会自动将`system prompt` 加进去发给 `ai` 模型。

# 🤗 AI 去完成任务

第一步做出这种尝试的是 AutoGpt 项目。

![[简单读懂Agent、MCP、Prompt、FunctionCaling-1751868844392.png]]

1. 首先自己定义一些工具函数，函数的作用、路径都交给 `autogpt`
2. 当用户提出问题时系统会将 `autogpt` 中定义的工具及描述都带上，模型如果够聪明的话会返回该如何调用这些函数
3. `autogpt` 根据模型返回的结果调用工具，并将工具执行结果返回给模型。

<aside> 🍿

人们将 `autogpt` 这种在工具、模型、人类之间传话的工具叫做 **`ai agent`**。供 agent 使用的工具叫做 agent tools

</aside>

> 会有什么样的问题？

agent 要求 ai 模型返回程序的位置“d:/塞尔达.exe”，但是模型可能只返回了个描述“我要玩塞尔达”。 此时，agent 会再次将工具信息与返回要求发送给ai模型。一次不行就多次（cline就是这样），反复的重试总是让人不靠谱。

# Function Calling

核心思想：统一描述，规定返回格式。

之前的提示词：

```markdown
user prompt:
	我要玩原神
system prompt:
	如果想使用就返回文件地址，工具列表如下：
	1. 原神应用程序在d:/原神.exe
	2. 塞尔达 d:/塞尔达.exe
```

`function calling` 标准出现后的提示信息

```markdown
user prompt:
	我要玩原神
system prompt:
	""
function calling:
{
    "name": "list_files",
    "desc": "列出目录",
    "params": {
        "path": "str"
    }
}
{
	返回的规范
}
```

> 还会有什么问题？

1. 由于 function calling 是由各家模型厂商制定的，所有每家的规范都有所差异，这使得统一变得困难。
2. 有的模型厂商还不支持function calling

# MCP

之前讲的都是关于提示词之间的内容。接下来是解决 agent 与 agent tools 之间产生的问题。刚开始人们为了让 agent 可以很方便的调用tools于是将 agent 与 tools 打包到一个程序中由相同的进程进行控制调用。由此引发的问题就是，一个工具没办法复用。

于是，将 tool 变成服务，可以让多个 agent 进行调用，运行 tool 的服务叫做 `mcp server` ,调用服务的 `agent` 叫做 `mcp client` 。

`mcp` 只负责管理工具、资源和提示词他与用什么样的模型、哪家的厂商并没有任何关系。

# 现在的流程

1. 我向 `mcp client` 或 `ai agent` 提问 “女朋友肚子疼怎么办”
2. `client` 通过 `mcp` 协议调用 `mcp server` 获取工具列表及功能描述
3. `ai agent` 将tools 列表转为 system prompt 或 function calling 的形式（厂商不同方式不同） + 用户提示词，一起发送给 ai 模型
4. `ai` 模型发现有“浏览网页”的工具，于是将工具信息组装成 `function calling` 或普通回复，将信息返回给 `agent`
5. `agent` 根据工具信息与参数调用工具，搜索答案
6. `agent` 将答案再扔给 ai 模型，ai 模型在根据网页内容 + 自身内容返回给用户。