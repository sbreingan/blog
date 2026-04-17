---
title: Patterns for building with LLMs
date: 2026-03-13 00:00:00 Z
categories:
- Tech
summary: In this blog I look at the landscapes of different architectural patterns available for using LLMs.
author: sbreingan
---


Cloud-managed services like AWS Bedrock make it straightforward to call a large language model from your application code. A few lines of configuration and an API call, and you have access to frontier reasoning capabilities. But what does it actually mean to integrate LLMs into an application, and what architectural choices need to be made?

I've broken this down into two key considerations:

- How are calls to the model orchestrated?
- What context does the model have?

These are independent choices. A simple, single call might need sophisticated context engineering. A multi-step agent might operate with only basic tooling. The architecture you need depends on where you sit on each axis.
In this post I'll work through both, with increasing complexity — from single calls to autonomous agents, and from bare prompts to RAG and tool use.

I've provided examples using AWS, where my expertise lies, but these patterns are generally applicable.

---

---
title: Patterns for building with LLMs
date: 2026-03-13 00:00:00 Z
categories:
- Tech
summary: In this blog I look at the landscape of different architectural patterns available for using LLMs.
author: sbreingan
---

Cloud-managed services like AWS Bedrock make it straightforward to call a large language model from your application code. A few lines of configuration and an API call, and you have access to frontier reasoning capabilities. But what does it actually mean to integrate LLMs into an application, and what architectural choices need to be made?

I've broken this down into two key considerations:

How are calls to the model orchestrated?
What context does the model have?

These are independent choices. A simple, single call might need sophisticated context engineering. A multi-step agent might operate from nothing more than a system prompt. The architecture you need depends on where you sit on each axis.
In this post I'll work through both, with increasing complexity — from single calls to autonomous agents, and from bare prompts to RAG and tool use.

I also want to ground this in something tangible; I've used examples, inspired by my background working in public sector and thinking about the potential use cases. I've provided examples using AWS, where my expertise lies.

---

## Part 1: Orchestration

Orchestration is about **how your application calls the model**. Who controls the flow? How many calls are made? Who decides what happens next?

### The Simple Call

Many of us are used to the power of LLMs like ChatGPT or Claude — these foundation models bring huge amounts of power on their own, across multiple domains.

The simplest integration is to treat the LLM as an API: send input, get output. This can be surprisingly powerful, and works well for bounded tasks such as classification, extraction, summarisation, and formatting.

![bedrock-simple.png](/sbreingan/assets/bedrock-simple.png)

The diagram shows the simplest possible architecture: a Lambda calls the Bedrock API directly and returns the result. Reasoning and generation happen in a single interaction.

#### Considerations

- Best suited to narrow, self-contained tasks
- No coordination overhead
- Lacks flexibility — if the task outgrows a single call, the architecture has to change

---

### The Workflow

Some tasks can be handled in a single model interaction. Others are better decomposed into stages.

Why would you want to do this? Suppose you're handling incoming requests for your service that have different levels of complexity. You _could_ write one large system prompt to handle everything, but this has downsides. More capable models like Claude Sonnet cost an order of magnitude more per token than lightweight models like Haiku, and use more energy.

If most of your requests are simple, you can save time, money, and energy by routing to cheaper models.

In a workflow pattern, the application controls a sequence of steps explicitly — for example classifying a request, choosing an appropriate model, generating a response, and validating the result before continuing. The logic for this is deterministic — encoded within the application itself.

![workflow-pattern.png](/sbreingan/assets/workflow-pattern.png)

The diagram shows a typical workflow structure using AWS Step Functions. A classification step determines the path, different branches use different models based on complexity, and a validation step checks the result before it is returned.

#### Considerations

- More moving parts than a single call
- Higher latency where steps are sequential
- Stronger control over cost, quality, and routing
- Each step can be tested and monitored independently

---

### The Agent

What if the right sequence of steps is not known in advance? What if the model needs to discover information along the way and decide what to do next?

This is where orchestration shifts from your application to the model. An agent is an LLM that can reason about the next action at runtime — for example whether to search, call a tool, inspect a result, or respond directly.

The defining feature is that **the model decides the flow while the task is being executed** in a non-deterministic way.

![bedrock-agent.png](/sbreingan/assets/bedrock-agent.png)

The diagram shows a Bedrock Agent reasoning in a loop — calling tools, evaluating results, and deciding the next action at each step. In AWS, Bedrock Agents is a managed service where you define the agent's role and the tools it has access to.

This pattern can even be extended to _multi-agent_ systems, where agents delegate tasks to other agents, if the complexity requires it.

#### Considerations

- Less deterministic than a fixed workflow — harder to test, predict, and bound
- Useful when a fixed sequence would be brittle or unnatural
- Requires thought about bounding: how many steps the agent can take, what budget it can spend, and what happens when it reaches a limit
- Observability becomes critical — you need to know what the agent actually did, not just what it returned
- Failure modes are broader: the agent might call the wrong tool, misinterpret a result, or loop without converging

---

## Part 2: Context

If orchestration is about how the model is called, context is about **what the model knows when it acts**.

Everything the model can use must arrive through its context window somehow. Broadly, there are three ways to do that.

### Providing Context in the Prompt

The simplest approach is to provide the context directly in the call itself. This includes the system prompt, instructions, examples, formatting constraints, conversation history, and any reference material small enough to fit.

For example, a council enquiry triage system might use a system prompt like this:

~~~text
"You are a council enquiry triage assistant. Classify incoming citizen 
enquiries into: Planning, Waste Services, Council Tax, Housing, Highways, Other. 

For each enquiry provide category, urgency (Low/Medium/High), and brief summary.
Respond only in JSON format."
~~~

**Input:**
`My bin hasn't been collected for two weeks`

**Output:**
`{"category": "Waste Services", "urgency": "Medium", "summary": "Missed collection"}`

#### Considerations

- Works well when the information a model needs is stable and small — keeps the architecture simple
- Will not scale if the data needed is large, proprietary, or dynamically generated

---

### Retrieval-Augmented Generation (RAG)

Sometimes the model needs information it was not trained on, or more information than you can reasonably place in the prompt. In that case, you retrieve the relevant information first and then pass it to the model as context.

That is Retrieval-Augmented Generation: the model generates its answer using retrieved material rather than relying only on its training data.

Retrieval can take several forms. The most common is vector search over chunked documents, where both the user query and the document chunks are embedded and matched for similarity. But retrieval does not have to mean vector search alone. Keyword search, hybrid search, and structured retrieval from knowledge graphs or databases can all play the same architectural role.

What matters is not the retrieval mechanism by itself, but that relevant information is selected before generation rather than hoping the model already "knows" the answer.

![rag-pattern-kb.png](/sbreingan/assets/rag-pattern-kb.png)

The diagram shows the retrieve-then-generate flow using Amazon Bedrock Knowledge Bases. The application retrieves relevant chunks from indexed documents, then passes them alongside the query to the model for generation. Amazon Bedrock Knowledge Bases automates the embedding and indexing process across data sources including S3, and can use vector stores such as OpenSearch Serverless.

#### Considerations

- Output quality depends on the full pipeline — how you chunk, embed, index, and retrieve — not just the model
- A system can retrieve the right document and still produce a poor answer if the chunking loses important context, the ranking surfaces the wrong section, or the prompt does not integrate the retrieved material well
- This retrieval-generation gap is where many real-world RAG projects struggle, and it is worth investing in evaluation and iteration across the whole chain
- Systematic evaluation — testing retrieval relevance, answer faithfulness, and end-to-end quality — is essential for understanding where the pipeline is failing and where to invest effort

---

### Tools and Live Data

Documents are not the only source of context. Sometimes the model needs access to live systems: databases, APIs, calculators, case records, or business operations.

Tools matter in two ways. First, their definitions become part of the model's context — the model needs to know what capabilities are available. Second, when a tool is called, the result comes back into the context window and becomes new information the model can reason over.

In AWS, tools can be implemented using Bedrock Agents with action groups. Each action group defines a set of tools — backed by Lambda functions or return-control handlers — that the agent can invoke. The agent receives the tool definitions as part of its context, reasons about which tool to call, and incorporates the result into its response.

Tool use does not require an agent. In a workflow, a model can produce structured output (such as a function name and parameters), which the application then executes deterministically before passing the result to the next step. The difference is who decides what to call: in a workflow, the application controls the sequence; in an agent, the model does.

#### Considerations

- Useful when the model needs live or transactional data
- Tool design, permissions, and failure handling matter
- More capability usually means more complexity and more risk

---

## Bringing the Two Axes Together

Designing an LLM integration means making two decisions, in order.

First, choose your **orchestration** — how the flow is controlled. This is the primary architectural decision: a simple call, a multi-step workflow, or an autonomous agent. Each adds complexity, operational overhead, and new failure modes.

Then, layer on **context** — what the model has access to. Prompt context is always present. Retrieval and tool use are additive capabilities you bring in when the model needs more information than the prompt alone can provide. You might use one, both, or neither — they are not exclusive choices.

![orchestration-context.png](/sbreingan/assets/orchestration-context.png)

At the agent level, the context distinction tends to collapse. Retrieval and tool use both become capabilities the agent invokes at runtime — in Bedrock, a Knowledge Base associated with an agent is called the same way as an action group. The distinction between RAG and tools matters most when choosing between simple calls and workflows, where they have genuinely different architectural shapes.

---

## Choosing Your Pattern

To show how these axes combine in practice, here are two examples from different parts of the grid.

### Example: FOI Request Assistant — Workflow + Retrieval

A department wants to help staff draft Freedom of Information responses using internal guidance and policy material, while keeping the process auditable and reviewable.

This is not a good fit for a single call — there is too much value in controlling the stages explicitly. But neither does it need an agent. The broad sequence of steps is already known.

**Orchestration: Workflow.** The process follows a controlled sequence:

1. Classify the request by complexity or sensitivity
2. Choose a model based on that classification — straightforward cases use a lightweight model, complex or sensitive cases route to a more capable one
3. Retrieve relevant guidance and supporting documents
4. Draft a response against that material
5. Validate the result before returning it

**Context: Prompt + Retrieval.** The prompt defines role, constraints, and response format. Retrieval supplies the relevant internal policies, templates, and guidance. The model drafts against that material rather than relying on general knowledge alone.

![foi-workflow.png](/sbreingan/assets/foi-workflow.png)

This combination works because it balances control and capability. The workflow keeps the process predictable, auditable, and cost-efficient through model routing. Retrieval grounds the output in internal material. Each stage can be tested and monitored independently.

### Example: Planning Research Assistant — Agent + Retrieval + Tools

Now consider something less predictable. A planning department wants help assessing whether a development proposal complies with local policy. The system may need to search planning guidance, inspect flood-risk data, look up precedent decisions, and synthesise what it finds — but the right sequence depends on what emerges along the way.

**Orchestration: Agent.** A fixed workflow would be brittle here, because the steps depend on the nature of the proposal and what the early searches return. The agent reasons about what to do next at each step.

**Context: Retrieval + Tools.** Policy documents are retrieved via RAG. Live data — flood maps, land registry records, previous decisions — is accessed through tools. Both feed into the agent's context window as it works.

This combination is more powerful but significantly harder to operate. The non-deterministic flow means you need to think carefully about bounding (how many steps? what budget?), observability (what did the agent actually do?), and safety (what happens if it calls the wrong tool or misinterprets a result?).

### What the contrast shows

The FOI assistant and the planning researcher sit in different cells of the grid, and for good reason. The FOI process is well-understood and benefits from explicit control. The planning assessment is exploratory and benefits from flexibility. Neither pattern is universally better — the right choice follows from the nature of the task.

---

## Cross-Cutting Concern: Guardrails

Whatever orchestration or context pattern you choose, production systems need guardrails — checks applied to inputs and outputs to keep the system behaving within acceptable bounds. This means filtering harmful content, blocking disallowed topics, protecting sensitive information, validating output formats, and checking whether responses stay within scope.

The key architectural point is that guardrails should be treated as a **separate layer**, not buried in prompts. Prompt instructions can be ignored or worked around by adversarial inputs. A dedicated guardrail layer applies consistently regardless of what the model tries to do.

In AWS, Bedrock Guardrails provides a managed option. You can configure content filters, denied topics, prompt attack detection, PII detection and redaction, and grounding checks that validate whether a response is supported by the retrieved context. These are applied as a policy attached to the model invocation, independent of the prompt itself.

How guardrails map onto the orchestration patterns is where the architectural thinking matters most. The complexity of your guardrail strategy scales directly with the autonomy of the model.

![guardrails-pattern.png](/sbreingan/assets/guardrails-pattern.png)

For a **simple call**, a single guardrail policy on the invocation may be sufficient — check the input, check the output. Two checks, predictable cost.

For a **workflow**, guardrails can be applied at stage boundaries. You might validate the input at the start, check for PII before retrieval, and verify grounding after generation. The explicit stages give you natural interception points, and each check can be tuned to the specific risk at that stage.

For an **agent**, guardrails need to apply on every iteration of the loop, not just the final output. An agent that calls a tool, receives a result, and reasons about it has multiple opportunities to go off-track before it produces a final response. This is where guardrails become most important and most operationally complex — you need to balance the cost of checking every loop iteration against the risk of letting problems compound unchecked.

The more autonomy the model has, the more important guardrails become.

---

## Choosing Your Approach

Designing an LLM system means making deliberate choices about orchestration and context — and resisting the pull toward unnecessary complexity.

A single call with a good prompt solves more problems than people sometimes expect. A workflow becomes useful when you want explicit control over stages, routing, model choice, or validation. Agents become valuable when the path genuinely depends on what the model discovers at runtime.

The important thing is not to reach for the most sophisticated-sounding pattern. It is to add complexity only on the axis where the simpler approach is genuinely falling short. Start simple, evaluate whether the outputs meet the bar, and move along the axes only when you have evidence that the current approach is insufficient.