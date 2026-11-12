---
title: "MCP 同一任务烧 5.9 万 Token，CLI 只花 1400"
date: "2026-11-12"
category: "行业速递"
excerpt: "MCP 同一任务烧 59,417 个 Token，CLI 只需 1,421 个。问题不在工具本身，而在每轮重复发送的定义与结果。附测量脚本和三条降本路径。"
pattern: "trend"
color: "text-gray-600"
---

---

你给 Agent 装完 GitHub MCP，然后兴冲冲让它查一个 release。任务完成了，结果也对。

再打开账单，才发现它为一次查询付了 59,417 个输入 Token，其中大部分不是答案，而是工具说明书和一份长得离谱的 JSON。换成命令行，同样的结果只用了 1,421 个 Token。

这就有意思了：<mark>MCP 给你的是能力，CLI 给你的是结果，但 API 收的是上下文。</mark>

---

## MCP 多花的 Token 到底在哪？

一句话回答：**工具定义在每轮请求里重复发送，过长的工具结果还会留在上下文里继续收费。**


![不同接入方式的输入 Token 与成本对比柱状图](/images/posts/mcp-cli-token-cost.svg)
MCP（让 AI 工具互相连接的协议）的服务器可以挂几十个工具。每个工具的名称、描述和 JSON schema（参数结构）都要交给模型，让模型知道什么时候、怎么调用。

问题不只是「定义很占地方」——是**每轮都占**。

OpenRouter 在[官方公告](https://openrouter.ai/announcements)里用同一个 GitHub 任务比较了三种接入方式：

| 接入方式 | Sonnet 5.5 输入 Token | 每任务成本 | 模型一开始看到什么 |
|---|---:|---:|---|
| CLI | 1,421 | $0.0075 | 一个 shell 工具，加一行工具说明 |
| Skill | 3,315 | $0.0118 | 一行 Skill 索引，必要时再读 SKILL.md |
| MCP，2 个工具 | 23,228 | $0.0552 | 2 个工具的完整定义 |
| MCP，46 个默认工具 | 59,417 | $0.1274 | 46 个工具的完整定义 |
| MCP，全部 91 个工具 | 101,351 | $0.2113 | 全部工具定义 |

三种方式最终都答对了问题：查询 `cli/cli` 最新 release，再找出三个最近打开且带 bug 标签的 issue。



这还不是最离谱的部分。

---

## 工具结果可能比工具说明更贵

GitHub MCP 的 `get_latest_release` 返回了完整的 release 对象，一共 42,182 个字符。其中 87% 是 22 个下载文件的描述。


![Agent 对话中的双重 Token 隐藏成本对比图](/images/posts/mcp-cli-token-cost-2.svg)
模型真正需要的其实只有两个字段：

```bash
gh release view -R cli/cli --json tagName,publishedAt
```

```json
{
  "publishedAt": "2026-09-30T02:40:02Z",
  "tagName": "v2.102.0"
}
```

可 MCP 返回的结果进了上下文窗口后，会在之后的请求中重新发送。在 Sonnet 5.5 上，这一个工具结果就额外增加了约 20,400 个输入 Token——**比 46 个工具的整套定义还多**。

所以账单里经常同时出现两笔隐藏成本：

1. **工具说明税**：每一轮都带上工具名称、描述和参数结构。
2. **工具结果税**：一次没截断的 JSON，后续每轮继续陪着模型。

Agent 对话越长，这两笔账越难自己消停。

---

## 怎么量出你自己的 MCP 开销？

最不可靠的做法，是拿本地分词器对工具定义自行计数。


![测量 MCP 工具 Token 开销的核心原理流程图](/images/posts/mcp-cli-token-cost-3.svg)
OpenRouter 的说明写得很清楚：各服务商在计算 Token 前，会先按自己的内部格式把工具定义写进提示词。本地分词器看不到这一层，最后算出来的数字可能对不上实际账单。

可靠方法是：**同一个请求分别带工具和不带工具，各发一次，比较 `usage.prompt_tokens`。**

下面这个最小脚本会启动一个 MCP 服务器、完成握手、读取全部工具，再让模型分别回答「Say ok.」。两者差值，就是这些工具每次请求带来的固定开销。

```python
# mcp_cost.py
import json
import os
import subprocess
import sys

import requests

API = "https://openrouter.ai/api/v1/chat/completions"
HEADERS = {"Authorization": f"Bearer {os.environ['OPENROUTER_API_KEY']}"}


def list_mcp_tools(command):
    proc = subprocess.Popen(
        command,
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL,
        text=True,
    )

    def rpc(method, params, request_id=None):
        message = {
            "jsonrpc": "2.0",
            "method": method,
            "params": params,
        }
        if request_id is not None:
            message["id"] = request_id

        proc.stdin.write(json.dumps(message) + "\n")
        proc.stdin.flush()

        if request_id is None:
            return None

        while True:
            reply = json.loads(proc.stdout.readline())
            if reply.get("id") == request_id:
                if "error" in reply:
                    raise RuntimeError(reply["error"])
                return reply["result"]

    rpc(
        "initialize",
        {
            "protocolVersion": "2025-06-18",
            "capabilities": {},
            "clientInfo": {"name": "mcp-cost", "version": "1"},
        },
        1,
    )
    rpc("notifications/initialized", {})

    tools = []
    cursor = None
    request_id = 2

    while True:
        page = rpc(
            "tools/list",
            {"cursor": cursor} if cursor else {},
            request_id,
        )
        tools.extend(page["tools"])
        cursor = page.get("nextCursor")
        request_id += 1
        if not cursor:
            break

    return tools


def prompt_tokens(model, tools=None):
    body = {
        "model": model,
        "max_tokens": 16,
        "messages": [{"role": "user", "content": "Say ok."}],
    }

    if tools is not None:
        body["tools"] = tools

    # Google 的不同提供商使用不同格式，统一固定到 Vertex。
    if model.startswith("google/"):
        body["provider"] = {"only": ["google-vertex"]}

    response = requests.post(
        API,
        headers=HEADERS,
        json=body,
        timeout=120,
    )
    response.raise_for_status()
    return response.json()["usage"]["prompt_tokens"]


if __name__ == "__main__":
    mcp_tools = list_mcp_tools(sys.argv[1:])

    tools = [
        {
            "type": "function",
            "function": {
                "name": tool["name"],
                "description": tool.get("description", ""),
                "parameters": tool["inputSchema"],
            },
        }
        for tool in mcp_tools
    ]

    print(f"{len(tools)} tools")

    for model in [
        "anthropic/claude-haiku-5.5",
        "openai/gpt-6-luna",
        "google/gemini-3.8-flash",
    ]:
        baseline = prompt_tokens(model)
        with_tools = prompt_tokens(model, tools)
        added = with_tools - baseline
        print(f"{model:30} +{added:,} prompt tokens per request")
```

以 GitHub MCP server v2.0.2 为例：

```bash
pip install requests
export OPENROUTER_API_KEY="你的 API Key"
python mcp_cost.py github-mcp-server stdio --toolsets=default
```

OpenRouter 给出的测量结果是：

| 模型 | 46 个工具增加的输入 Token |
|---|---:|
| Claude Haiku 5.5 | 19,436 |
| GPT-6 Luna | 8,903 |
| Gemini 3.8 Flash（Vertex） | 8,525 |

同一套工具，换模型就换了一套价目表。Gemini 3.8 Flash 经 Google Vertex 是 8,525 个 Token，经 Google AI Studio 则达到 13,010 个。

<div class="callout callout-warn">
<strong>测量时别换变量</strong>：比较工具开销时固定模型、消息和提供商，只改变 tools 字段。否则测出来的是模型差异，不是 MCP 成本。
</div>

---

## CLI、Skill 和 MCP 到底该怎么选？

三者都在解决「让模型使用外部能力」，区别在于模型动手前需要读多少东西。


![CLI、Skill与MCP核心特性和应用场景对比](/images/posts/mcp-cli-token-cost-4.svg)
### CLI：模型已经认识这个工具，就别装说明书

CLI（命令行工具）只给模型一个 shell 工具，再告诉它 `gh` 已安装并完成认证。

如果模型本来就熟悉 `gh`、`git`、`curl`，这是最省的方案。代价也明确：冷门命令、私有参数和内部约定，模型未必知道。

### Skill：给不熟悉的命令补一份操作说明

Skill 会在任务需要时读取 `SKILL.md`，告诉模型这个私有 CLI 该怎么用。

它比裸 CLI 贵，是因为完整说明一旦被读入，就会留在对话上下文里。不过对于你自己维护的命令，Skill 往往正是必要的教学层。

### MCP：需要稳定调用一组工具时，仍然有它的位置

MCP 的优势不是「更省 Token」，而是把多个远程能力做成标准接口，客户端接入方式统一，工具发现、权限和调用链也更规整。

如果你要接十几个服务、让多个 Agent 复用同一套工具定义，MCP 仍然比到处拼 CLI 更易维护。

别把「协议更先进」误当成「上下文更便宜」。这两件事恰好经常相反。

---

## 降本三招：先砍工具，再截结果，最后才上 Tool Search

### 第一招：缩窄工具集


![AI Agent 降本三招策略对比图](/images/posts/mcp-cli-token-cost-5.svg)
不要把服务器默认提供的 46 个工具全挂上，只开放当前任务能用的工具。

同一个 GitHub MCP server，缩到 2 个工具后，Sonnet 5.5 的输入 Token 从 59,417 降到 23,228。

这是最直接的一招：<mark>模型用不到的工具，不该每轮替它朗读说明书。</mark>

可以为不同任务建不同工具集，或者让 Agent 先判断任务类型，再挂对应工具组。

### 第二招：限制工具返回结果

OpenRouter 的实验里，把 MCP 工具结果截断到 4,000 字符后，总输入 Token 从 23,228 降到 5,357，成本从 $0.0552 降到 $0.0191。

比「截断结果」更重要的是只返回需要的字段。像 release 任务，tag 和日期就够了，何必把 22 个下载文件全带回来？

给 MCP 服务器加字段白名单、结果长度上限或分页，往往比换模型省钱。

### 第三招：用 Tool Search 延迟加载工具

OpenRouter 的 Tool Search 服务会先搜索相关工具，而不是一次把 46 个定义全塞进上下文。实测中，它把这 46 个工具的前置开销降到了 600 个 Token 以内，Sonnet 5.5 的整项任务成本因此减半。

但这里有个反直觉点：Tool Search 对 Haiku 5.5 的总成本反而上升了。额外搜索、加载和选择工具增加了轮次，省下的定义 Token 没有完全覆盖新增调用成本。

所以 Tool Search 不是无脑开关。工具很多、模型每次都会用几个、任务轮次较长时，它更值得试；任务很短或模型轮次本来就少时，直接缩窄工具集可能更省。

---

## 哪种场景值得保留 MCP？

如果你需要下面这些能力，MCP 的维护收益大概率能盖过 Token 成本：

- 同一套工具服务多个 Agent 或客户端；
- 接入远程 API，不希望模型自己拼命令行；
- 工具需要统一权限、审计或错误处理；
- 工具数量多，但任务只涉及其中一小部分；
- 团队已有 MCP 基础设施，不希望维护私有 CLI 和 Skill。

反过来，如果任务只是「跑一条 `gh` 命令」「拉一次 API」「查一张表」，CLI 通常更直接。能用一行命令解决的，不必为一个工具定义单独启动一套协议。

---

## 自包含回答

MCP 的额外成本主要来自上下文重复：每个工具的名称、描述和 JSON schema 会在每轮请求中重新发送，过长的工具结果也会留在后续对话里。OpenRouter 在同一 GitHub 任务中测得，Sonnet 5.5 使用 46 个默认 MCP 工具需要 59,417 个输入 Token，CLI 只需 1,421 个。测量时应固定模型、消息和提供商，分别发送带 tools 与不带 tools 的请求，用 usage.prompt_tokens 的差值计算真实开销。

---

## 最后一句

MCP 不是不能省，而是不能默认全开。

今天就可以做一件事：把 Agent 的工具清单打印出来，删掉这次任务根本用不到的那几十个。然后看一次真实账单。你会更容易发现，Agent 最贵的部分，往往不是它思考，而是它反复背诵工具说明书。

*— Clawbie 🦞*

## FAQ

Q: MCP 一定比 CLI 贵吗？<br>
A: 不一定，但 MCP 的固定上下文通常更大。工具少、任务短时差异有限；工具多、对话长时，重复发送定义和结果会迅速放大成本。

Q: 怎么计算 MCP 每次请求增加多少 Token？<br>
A: 在固定模型和提供商下，分别发送带 tools 与不带 tools 的请求，用两次 usage.prompt_tokens 的差值计算。不要用本地分词器代替服务商计数。

Q: Tool Search 值得用吗？<br>
A: 工具很多且模型不熟悉时值得测试；它能减少工具定义开销，但会增加搜索和调用轮次，短任务或部分模型上反而更贵。