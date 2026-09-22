---
title: 常见问题
description: 关于 Memory、Handoff、Experience、Skill 与知识生命周期的常见问题。
---

# 常见问题

本页回答学习 PowerContext 时经常遇到的问题。更系统的概念介绍见[核心概念](./core-concepts.md)；具体操作步骤请
参阅相应的工作流指南。

## 四类 Artifact 速览

**问：我经常看到 Memory、Handoff、Experience 和 Skill，应该分别在什么时候用？**

这四个 family 在同一个工作循环里扮演不同角色。想象一个 Agent 在修复 `amount.py` 中的 CSV 解析 bug：

| Family | 在修复 bug 过程中扮演的角色 | 适用场景 |
| --- | --- | --- |
| **Memory** | 保存长期项目约定："金额以整数分存储；超过两位小数的输入必须拒绝。" | 跨越多个会话与 Agent 都应成立的事实、决策或规则。 |
| **Handoff** | 记录"我已确认 bug 在 `cents()` 函数里；明天由下一个 Agent 跑失败的测试并打补丁。" | 把未完成工作的当前状态交接给另一个 Agent 或会话。 |
| **Experience** | 记录已经验证的教训："直接调用 `int(Decimal(text) * 100)` 把 `1.999` 静默截断成了 `199`；现在我们在转换前先校验精度。" | 保留基于证据的情境教训。 |
| **Skill** | 沉淀可复用流程："如何验证金额转换修复——运行测试夹具、检查边界、记录结果。" | 供其他 Agent 加载并遵循的、已验证的可复用配方。 |

一次 bug 修复通常会同时产生这四类内容：要遵守的约定（Memory）、移交的进展（Handoff）、学到的教训
（Experience）、可复用的做法（Skill）。

详见 [Memory 与 Handoff](../workflows/memory-and-handoff.md) 以及
[Experience 与 Skill 的生命周期](../workflows/experience-and-skill-lifecycle.md)。

## Source、Candidate 与知识流水线

**问：Source、Candidate 和已批准的 Artifact 是一条强制流水线吗？**

不是。它们是可以组合的独立概念，但捕获证据本身不会产生已批准的知识：

```text
Source（证据）──┐
                ├──→ Candidate（提案）──→ Review ──→ 已批准 Artifact
已有 Artifact ──┘
```

- **Source** 是原始证据——一轮对话、一次工具结果、一份文档。
- **Candidate** 是从证据或已有 Artifact 派生的提案。在审核前处于 `pending` 状态。
- **Artifact** 是审核批准后得到的产物——带有稳定引用的不可变 Revision。

也可以通过 `remember_memory` 直接写入 Memory，跳过 Source 提取和 Candidate 审核。

**问：pending 状态的 Candidate 会被 `PreparedContext` 召回吗？**

不会。pending Candidate 不参与召回。只有已批准的当前 Revision 才能进入 `PreparedContext`。这样可以防止
未经审核的模型输出悄悄进入 Agent 的工作上下文。

**问：我批准了一个 Skill Candidate，为什么 Agent 还用不了？**

批准、安装和执行是三个独立步骤：

1. **批准**在 PowerContext 中创建一个不可变的 Skill Revision。
2. **导出 / 安装**把指定 Revision 复制到 Agent 宿主（例如通过 Remote Skill 分发，或手动调用
   `download_skill_package`）。
3. **执行**发生在 Agent 宿主发现并加载已安装的 `SKILL.md` 之后。

批准 Skill 不等于安装它；安装它也不等于执行它。每一步都是显式的，以便你掌控"什么内容在什么地方运行"。

## Memory

**问：Source 和 Memory 有什么区别？**

- **Source** 是原始证据：一条聊天消息、一份文件、一段 HTTP 调用记录。按原样保存。
- **Memory** 是经过提炼的知识：一个决策、一条约束、一项事实。它是被刻意写入的（由应用显式写入，或由经过
  批准的提取流程产生）。

Source 是输入；Memory 是策展后的结果。

**问：我通过通用 Artifact API 创建了 Memory，但 `search_memory` 找不到，为什么？**

两条写入路径的效果不同：

| 路径 | 结果 |
| --- | --- |
| `POST /v1/scopes/{id}/artifacts`，`family=memory` | 创建独立的 Memory Artifact；**不参与**日常召回。 |
| `POST /v1/memory/remember` | 写入 Scope 的默认 Memory；**可以被** `search_memory` 检索，并被 `prepare_context` 使用。 |

如果你希望 Agent 真正能召回这条事实，请使用 `/v1/memory/remember`。

**问：修订 Memory 时旧版本会被删除吗？**

不会。Revision 是不可变的。修订 Memory 会创建一个新 Revision，并把旧 Revision 从活跃召回中退役；但旧
Revision 仍可通过 `GET .../revisions/{n}` 读取。这样既能保留审计轨迹，也能在需要时恢复早期措辞。

## Handoff

**问：Handoff 是手动创建的还是 Agent 自动创建的？**

都可以。当人类判断需要移交工作时，应用可以直接调用 `handoff_current_work`；Agent 也可以在其工具流程中
提出 Handoff。两种情况下的流程相同：先返回一份临时预览，只有 `commit_handoff` 才会写入持久的 Revision。

**问：接收方 Agent 怎么知道要做什么？**

Handoff 自身带有结构化字段：目标、带证据引用的当前状态、处置方式（`continuable` / `complete`）、下一步
动作以及已声明的遗漏。接收方 Agent 阅读的是这份内容，而不是完整的先前会话。

## Experience 与 Skill

**问：Experience 和 Skill 有什么区别？**

- **Experience** 是一段经过验证的叙事：情境 → 行动 → 结果 → 教训。它告诉后来的读者*当时发生了什么、学到
  了什么*。
- **Skill** 是一个可复用流程：名称、描述、指令、验证步骤。它告诉后来的 Agent *该怎么做*。

常见模式是：先有一条已批准的 Experience，再基于它生成 Skill Candidate，审核通过后才成为可安装的
Skill。

**问：PowerContext 会自动安装 Skill 吗？**

不会。Skill 作为 Artifact 存储在 PowerContext 中。要让 Agent 能使用它，应用（或 Remote Skill Receiver）必
须显式下载并安装到 Agent 的工作目录。

## Middleware 与 MCP 工具

**问：教程里经常看到"Middleware"，它是什么？**

Middleware 是 LangChain 的 `create_agent` 暴露的拦截点。它运行在 Agent 调用模型的前后，允许你在这个阶段
插入自定义逻辑。PowerContext 提供的 `PowerContextMiddleware`（位于 `powercontext_langchain` 包中）就利用这
个钩子，为当前这一轮调用准备上下文，但不会修改 Agent 的推理循环。其他框架使用不同的扩展机制；
PowerContext 通过 MCP 与它们集成。

**问：什么时候用 Middleware，什么时候用 MCP 工具？**

这两种模式回答的是不同的接入问题：

| 模式 | 适用场景 |
| --- | --- |
| **Middleware** | 希望在携带用户消息的模型调用前自动注入背景上下文。 |
| **MCP 工具** | 希望由 Agent 自己决定何时调用 `search_memory`、`remember_memory` 等 PowerContext 操作。 |

Middleware 是被动方式——由 PowerContext 决定注入什么。MCP 工具是主动方式——由模型决定何时调用。两者可以
在同一个 Agent 中同时使用。安装与配置详见 [LangChain 集成](../integrations/langchain.md)；完整协议说明
见 [接口](../develop/interfaces.md)。

**问：接入 PowerContext 需要重写我的 Agent 吗？**

不需要。PowerContext 通过 Middleware 或工具为 Agent 增加能力，不替换 Agent 的推理循环、消息格式或已有工
具。接入 LangChain Middleware 只需要多传一个参数：

```python
from langchain.agents import create_agent
from powercontext_langchain import PowerContextMiddleware, PowerContextScope

agent = create_agent(
    model,
    tools=application_tools,
    middleware=[PowerContextMiddleware()],
    context_schema=PowerContextScope,
)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "..."}]},
    context=PowerContextScope(),
)
```

**问：Middleware 注入的内容会污染会话历史吗？**

不会。注入的内容只在单次调用中生效，并且永远不会写入 Agent 的状态。每次调用模型时都会重新注入一次，之
后便丢弃。会话历史中只保留用户与模型真实交互的消息。

**问：MCP 是什么？它解决了什么问题？**

MCP（Model Context Protocol，模型上下文协议）是一个开放协议，让任意 Agent 都能通过标准接口接入外部工具
和数据源。PowerContext 提供 MCP Server，因此兼容 MCP 的 Agent（如 Codex、Claude Code 等）可以直接使用
PowerContext，而不需要为每个框架单独写集成代码。MCP 暴露的能力覆盖 Memory 检索与写入、Source 采集、
Handoff 等相关操作。

通常在希望 Agent 主动决定何时读写 PowerContext 时选择 MCP 工具，而不是由应用自动注入上下文。

**问：可以同时使用 Middleware 和 MCP 工具吗？**

可以。这两个集成位于不同的包中（Middleware 在 `powercontext_langchain`，工具在 `powercontext_langgraph`），
设计上就是为了在同一个 Agent 中组合使用。一种常见的组合是：用 Middleware 提供"始终在线"的背景上下文，用
MCP 工具支持由模型主动发起的 Memory 写入：

```python
from langchain.agents import create_agent
from powercontext_langchain import PowerContextMiddleware, PowerContextScope
from powercontext_langgraph import powercontext_tools

agent = create_agent(
    model,
    tools=powercontext_tools(),
    middleware=[PowerContextMiddleware()],
    context_schema=PowerContextScope,
)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "..."}]},
    context=PowerContextScope(),
)
```

## 还有疑问？

- 概念模型请见[核心概念](./core-concepts.md)。
- 端到端流程请见[工作流](../workflows/index.md)章节。
- API 细节请见 `openapi/powercontext.yaml`。
- 如果本页内容不清楚或有误，欢迎[提出 Issue](https://github.com/oceanbase/powercontext/issues/new) 帮助
  我们改进。
