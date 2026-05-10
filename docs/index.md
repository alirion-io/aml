# Welcome to the Agent Modeling Language (AML)

AML was born out of a frustration: existing agent definition formats are tightly coupled to specific tools — Anthropic Claude, GitHub Copilot in VS Code, Cursor, and others. Each ecosystem has its own conventions, and none of them are designed for what companies actually need: agents running reliably in the cloud, governed by teams, audited by compliance, and maintained across versions.

The idea is not to reinvent the wheel. AML extends what is already familiar — Markdown and YAML — and shapes it for cloud deployment, team ownership, and production governance.

!!! note
    To facilitate authoring and reviewing AML files, we have implemented an [AML studio](https://alirion-io.github.io/aml-studio/). It can run locally or in the cloud, connected or not to a Git system for full auditability of changes.


---

## What is AML?

* **A human-readable format** for describing cloud agents, understandable by business people, product owners, non-developer agent authors, platform architects, and AI governance teams — not just engineers.
* **An abstraction layer** that decouples the description of an agent's behavior from the runtime framework that executes it. Change your framework; keep your definitions.
* **A governance surface** that makes agent capabilities, tool access, and guardrails explicit, versionable, and reviewable — just like infrastructure-as-code.
* **A deployment pipeline target**: AML files are compiled, validated, and deployed using [Strands Agents](https://strandsagents.com/), with compiler support for other frameworks planned.

## What isn't AML?

* **An agent framework**: AML is framework-agnostic. Any runtime that has a compiler for AML can execute agents described with it.
* **A workflow or orchestration engine**: AML describes what an agent is and what it is allowed to do. It can define which agents an agent can use for what but not a full workflow.
* **A prompt library**: AML files carry behavioral instructions, but they are compiled into enforced runtime configuration, not prompt strings passed verbatim to a model.

---

## What is an agent, really?

Strip away the hype and an AI agent is, at its core, **a function** — a plain Python or NodeJS function that runs in the cloud like any other backend service.

```python
def run_agent(user_message: str, context: dict) -> str:
    # 1. Build the conversation so far
    messages = build_messages(context, user_message)

    # 2. Call the model
    response = llm.chat(model="claude-3-7-sonnet", messages=messages, tools=TOOLS)

    # 3. If the model wants to use a tool, run it and loop back
    while response.tool_calls:
        tool_results = execute_tools(response.tool_calls)
        messages.append(tool_results)
        response = llm.chat(model="claude-3-7-sonnet", messages=messages, tools=TOOLS)

    # 4. Return the final answer
    return response.text
```

That is it. There is no magic. What makes an agent *feel* intelligent is not the function itself — it is everything that gets wired into it:

* **A system prompt** that gives the model its persona, its rules, and its scope. This is the text that tells the model "you are a customer support agent for Acme Corp, you never discuss competitors, and you escalate billing disputes to a human."
* **A list of tools** the model is allowed to call — functions that look up data, send emails, create tickets, query a database. The model decides *which* tool to call and with *what* arguments; the function executes the actual call.
* **A knowledge base** the agent can search — a vector store of documents, policies, FAQs, or product data. When the user asks something the model cannot answer from training data alone, it retrieves relevant chunks and includes them in the prompt.
* **Memory** — short-term (the conversation so far) or long-term (facts persisted across sessions about the user or the task).
* **Guardrails** — code that runs before or after the model response to block harmful outputs, strip PII, or enforce business rules regardless of what the model returns.

The loop — call the model, execute tools, call the model again until done — is called the **agentic loop**. Every agent framework (LangChain, LlamaIndex, Strands, AutoGen, CrewAI…) is ultimately a library that manages this loop and wires those five components together in a structured way.

**So what is an AML file?** It is the declaration of all those components — the system prompt, the tools, the knowledge bases, the guardrails, the memory policy — written in a format that a compiler turns into the actual wired-up function. The function is generated; the human authors the intent.

---

## Why cloud agents — and why they are different

### Local and IDE-embedded agents are not production agents

Tools like Claude Desktop, GitHub Copilot in VS Code, or Cursor are developer/user tools. They run on a single machine, serve a single user, and operate with that user's ambient permissions. They are excellent for individual productivity. They are not designed for:

* serving dozens, hundreds, or millions of end users simultaneously
* enforcing access controls that differ per user or per team
* being audited by legal or compliance teams after the fact
* being updated, rolled back, or A/B tested without restarting the developer/user's machine
* outliving the employee who set them up

When a company decides to deploy an agent to customers or internal teams at scale, the constraints change entirely.

### What changes when you move agents to the cloud

| Concern | Local / IDE agent | Cloud production agent |
|---|---|---|
| **Availability** | Runs when the developer/user's machine is on | Always-on, replicated, fault-tolerant |
| **Identity & access** | Inherits operator's credentials | Fine-grained IAM per agent, per tool, per caller |
| **Multi-tenancy** | Single user | Potentially thousands of users, isolated data |
| **Observability** | Console logs, maybe | Structured traces, metrics, alerting, cost attribution |
| **Guardrails** | Best-effort prompt instructions | Machine-enforced at runtime, independently of the model |
| **Knowledge freshness** | Static files or context window | Managed vector stores with versioned, freshness-controlled retrieval |
| **Governance** | None — agent does what the developer/user wants | Approval workflows, policy review, lifecycle states |
| **Auditability** | Difficult or impossible | Every decision, tool call, and refusal is logged and attributable |
| **Change management** | Edit a file, restart | Pull requests, code review, CI/CD pipeline, staged rollout |

### What "professional" agents require

When a company runs an agent in production, it is not just a model with a system prompt. It is a service with all the operational responsibilities that implies:

* **Access rights**: The agent must be able to call the tools it needs — and only those. Credentials must be managed, rotated, and scoped. For example, an agent that can send emails should not also be able to delete database records.
* **Guardrails**: Content policies, PII handling, and refusal behavior must be enforced at the infrastructure layer, not just written in the prompt. A prompt instruction like "never reveal customer data" is editorial guidance; a compiled runtime policy is an enforceable control.
* **Observability**: Every tool call, every model invocation, every refusal must be traceable. When something goes wrong — or when compliance asks — you need to reconstruct exactly what the agent did and why.
* **Knowledge bases**: Enterprise knowledge is not static context. It lives in documents, databases, and ticketing systems that change daily. Agents need managed retrieval with defined freshness policies, not copy-pasted text in a system prompt or adding as uploaded file.
* **Model flexibility**: Production teams need to evaluate, swap, or fine-tune models without rewriting agent behavior. The model is a dependency, not the agent itself.
* **Lifecycle management**: Agents have versions. They get drafted, reviewed, approved, deployed, deprecated, and retired. That lifecycle needs tooling — not a shared Slack channel and hope.

### The governance gap no one talks about

One of the most underestimated risks of enterprise AI deployment is the **governance gap**: the disconnect between what an agent is authorized to do (typically described informally in a Notion page or a Jira ticket) and what it actually does at runtime.

AML closes that gap by making the agent's definition the authoritative source of truth. If an agent references a tool, that tool is explicitly declared and reviewed. If an agent has guardrails, they are compiled into the runtime payload, not buried in a prompt. If an agent is updated, the change goes through version control and can be diffed, reviewed, and audited.

This is the same principle that made infrastructure-as-code transformative for DevOps: **if it is not in the file, it does not exist**.

---


