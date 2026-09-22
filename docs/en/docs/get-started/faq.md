---
title: FAQ
description: Common questions about Memory, Handoff, Experience, Skill, and the knowledge lifecycle.
---

# Frequently asked questions

This page answers questions that often come up when learning PowerContext. For a deeper introduction to the
underlying model, see [Core concepts](./core-concepts.md). For step-by-step procedures, follow the links to the
relevant workflow guides.

## The four Artifact families at a glance

**Q: I keep seeing Memory, Handoff, Experience, and Skill. When should I use each?**

The four families play different roles in the same work loop. Imagine an Agent fixing a CSV parsing bug in
`amount.py`:

| Family | Role in the bug-fix story | When to use it |
| --- | --- | --- |
| **Memory** | Stores the durable project constraint: "amounts are stored as integer cents; reject anything with more than two decimal places." | A fact, decision, or rule that should hold across many sessions and Agents. |
| **Handoff** | Captures "I confirmed the bug is in `cents()`; tomorrow the next Agent should run the failing tests and patch it." | Transferring the current state of unfinished work to another Agent or session. |
| **Experience** | Records the verified lesson: "Directly calling `int(Decimal(text) * 100)` silently truncated `1.999`; we now check precision before converting." | Preserving what was learned from a specific situation, with evidence. |
| **Skill** | Packages the reusable procedure: "How to verify amount-conversion fixes — run the test fixture, check edge cases, record results." | A reusable, validated recipe other Agents can load and follow. |

A single bug fix often produces all four: the constraint it honors (Memory), the state it hands over (Handoff),
the lesson it learned (Experience), and the procedure worth repeating (Skill).

For details, see [Memory and Handoff](../workflows/memory-and-handoff.md) and
[Experience and Skill lifecycle](../workflows/experience-and-skill-lifecycle.md).

## Sources, Candidates, and the knowledge pipeline

**Q: Are Sources, Candidates, and approved Artifacts stages of one mandatory pipeline?**

No. They are independent values that *can* be combined, but capturing evidence does not by itself produce
approved knowledge:

```text
Source (evidence) ──┐
                    ├──→ Candidate (proposal) ──→ Review ──→ approved Artifact
existing Artifact ──┘
```

- A **Source** is raw evidence — a conversation turn, a tool result, a document.
- A **Candidate** is a proposal derived from evidence and/or existing Artifacts. It is `pending` until reviewed.
- An **Artifact** is what you get after approval — an immutable Revision with a stable reference.

You can also write Memory directly with `remember_memory` without going through Source extraction or Candidate
review.

**Q: Can a pending Candidate be recalled into `PreparedContext`?**

No. Pending Candidates are excluded from recall. Only approved, current Revisions participate in
`PreparedContext`. This prevents unreviewed model output from silently becoming the Agent's working context.

**Q: I approved a Skill Candidate. Why can't my Agent use it yet?**

Approval, installation, and execution are three separate steps:

1. **Approval** creates an immutable Skill Revision in PowerContext.
2. **Export / install** copies an exact Revision to the Agent host (for example via Remote Skill distribution or
   manual `download_skill_package`).
3. **Execution** happens when the Agent's host discovers and loads the installed `SKILL.md`.

Approving a Skill does not install it; installing it does not execute it. Each step is explicit so that you keep
control over what runs where.

## Memory

**Q: What's the difference between Source and Memory?**

- A **Source** is raw evidence: a chat message, a file, an HTTP trace. It is stored as-is.
- **Memory** is refined knowledge: a decision, a constraint, a fact. It has been deliberately written (either by
  the application or by an approved extraction pipeline).

Sources are inputs; Memory is the result of curation.

**Q: I created a Memory through the generic Artifact API but it doesn't show up in `search_memory`. Why?**

There are two write paths with different effects:

| Path | Result |
| --- | --- |
| `POST /v1/scopes/{id}/artifacts` with `family=memory` | Creates a standalone Memory Artifact; **not** part of daily recall. |
| `POST /v1/memory/remember` | Writes to the Scope's default Memory; **is** searched by `search_memory` and used in `prepare_context`. |

Use `/v1/memory/remember` for facts you want the Agent to actually recall.

**Q: When I revise a Memory, does the old version get deleted?**

No. Revisions are immutable. Revising a Memory creates a new Revision and retires the old one from active
recall, but the old Revision remains readable through `GET .../revisions/{n}`. This gives you an audit trail and
a way to recover earlier wording.

## Handoff

**Q: Is a Handoff created manually or by the Agent?**

Either. An application can call `handoff_current_work` directly when a human decides it's time to transfer work.
An Agent can also propose a Handoff as part of its tool flow. In both cases the flow is the same: a temporary
preview is returned first, and only `commit_handoff` writes a durable Revision.

**Q: How does the receiving Agent know what to do?**

The Handoff itself carries structured fields: the objective, the current state with evidence citations, the
disposition (`continuable` / `complete`), the next action, and any omissions. The receiving Agent reads this
content rather than the full prior conversation.

## Experience and Skill

**Q: What's the difference between an Experience and a Skill?**

- An **Experience** is a verified narrative: situation → action → outcome → lesson. It tells future readers
  *what happened and what was learned*.
- A **Skill** is a reusable procedure: name, description, instructions, validation steps. It tells future Agents
  *how to do something*.

A common pattern is that an approved Experience is used to generate a Skill Candidate, which is then reviewed
and (once approved) becomes installable.

**Q: Does PowerContext install Skills automatically?**

No. Skills live in the PowerContext store as Artifacts. To make one usable by an Agent, the application (or a
Remote Skill Receiver) must explicitly download and install it to the Agent's working directory.

## Middleware and MCP tools

**Q: I see "Middleware" everywhere in the tutorials. What is it?**

Middleware is an interception point exposed by LangChain's `create_agent`. It runs around the Agent's model call,
letting a hook inject logic before or after the model is invoked. PowerContext ships a `PowerContextMiddleware`
(in the `powercontext_langchain` package) that uses this hook to prepare context for the current turn, without
modifying your Agent's reasoning loop. Other frameworks use different extension mechanisms; PowerContext
integrates with them through MCP instead.

**Q: When should I use Middleware vs MCP tools?**

The two patterns answer different integration questions:

| Pattern | When to use |
| --- | --- |
| **Middleware** | You want background context automatically injected before a model call that carries a user message. |
| **MCP tools** | You want the Agent itself to decide when to call `search_memory`, `remember_memory`, or other PowerContext operations. |

Middleware is passive — PowerContext decides what to inject. MCP tools are active — the model decides when to
call them. They can be combined in the same Agent. See the
[LangChain integration](../integrations/langchain.md) for setup, and [Interfaces](../develop/interfaces.md) for
the full protocol surface.

**Q: Does integrating PowerContext require rewriting my Agent?**

No. PowerContext adds capability through Middleware or tools, without replacing your Agent's reasoning loop,
message format, or existing tools. Adding the LangChain Middleware is a single extra argument:

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

**Q: Does Middleware injection pollute my conversation history?**

No. Injected content is single-use and never persisted to the Agent's state. Each model call gets a fresh
injection, then it is discarded. Your conversation history contains only the messages the user and the model
actually exchanged.

**Q: What is MCP, and when does it matter?**

MCP (Model Context Protocol) is an open protocol that lets any Agent connect to external tools and data sources
through a standard interface. PowerContext exposes an MCP Server, so MCP-compatible Agents (Codex, Claude Code,
and others) can use PowerContext without per-framework integration code. The MCP surface covers Memory search
and write, Source capture, Handoff, and related operations.

You typically reach for MCP tools when you want the Agent itself to decide when to read or write PowerContext,
rather than having the application inject context automatically.

**Q: Can I use Middleware and MCP tools together?**

Yes. The two integrations live in separate packages (`powercontext_langchain` for Middleware,
`powercontext_langgraph` for tools) and are designed to be combined in one Agent. A common pattern is Middleware
for always-on background context plus tools for explicit, model-driven Memory writes:

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

## Still stuck?

- For the underlying model, see [Core concepts](./core-concepts.md).
- For end-to-end procedures, see the [Workflows](../workflows/index.md) section.
- For API details, see the OpenAPI contract in `openapi/powercontext.yaml`.
- If something here is unclear or wrong, please
  [open an issue](https://github.com/oceanbase/powercontext/issues/new) so we can improve this page.
