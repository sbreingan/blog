---
title: Patterns for building with LLMs
date: 2026-04-23 00:00:00 Z
categories:
- Tech
summary: Integrating an LLM into your application starts with an API call, but the architectural decisions run deeper. How many calls do you make? Who controls the flow — your application or the model? What information does the model have access to? This post works through those choices with increasing complexity, from single calls to autonomous agents and from bare prompts to RAG and tool use.
author: sbreingan
---

Cloud-managed services like AWS Bedrock make it straightforward to call a large language model from your application code. A few lines of configuration and an API call, and you have access to cutting-edge reasoning capabilities.
 
Much has been written about the possibilities this brings, with exciting new architectures involving multi-agent systems with advanced reasoning.
 
But where should we start when thinking about architecting a service with an LLM? What choices are we making?
 
As this space is constantly evolving, it can be easy to get lost in the latest developments. But for organisations just starting out thinking about LLMs, it can be helpful to take a step back and start small. I find it helpful to think of two key considerations:
 
- How are calls to LLM models orchestrated?
- What context does the model have?

These are independent choices. A simple, single call might need sophisticated context engineering. A multi-step agent might operate with only basic tooling. The architecture you need depends on where you sit on each axis.
 
In this post I'll work through both, with increasing complexity — from single calls to autonomous agents, and from bare prompts to RAG and tool use.
 
I've provided examples using AWS, where my expertise lies, but these patterns are generally applicable.

---

## Part 1: Orchestration

Orchestration is about **how your application calls the model**. Who controls the flow? How many calls are made? Who decides what happens next?
 
### The Simple Call
 
As many have encountered through the use of tools such as ChatGPT, these foundation models bring huge amounts of power on their own, understanding a wide variety of domains.
 
The simplest integration is to treat the LLM as an API: send input, get output. This can be surprisingly powerful, and works well for bounded tasks such as classification, extraction, summarisation, and formatting.
 
![bedrock-simple.png]({{ site.baseurl }}/sbreingan/assets/bedrock-simple.png)
 
Bedrock provides a managed service for accessing these models — you don't need to think about hosting or deploying, you can simply treat them as an API call, such as from a Lambda function. Any reasoning and generation happen in a single interaction.
 
#### Considerations
 
- Best suited to narrow, self-contained tasks
- No coordination overhead
- Lacks flexibility — if complexity increases this may become too narrow
---
 
### The Workflow
 
Some tasks are better suited to be decomposed into multiple calls, particularly when we may want to use different models.
 
Suppose you're handling incoming requests for your service that have different levels of complexity. You _could_ write one large system prompt to handle everything, but this has downsides. More capable models like Claude Sonnet cost an order of magnitude more per token than lightweight models like Haiku, and use more energy. If most of your requests are simple, you can save time, money, and energy by routing to cheaper models.
 
In a workflow pattern, the application controls a sequence of steps explicitly — for example classifying a request, choosing an appropriate model, generating a response, and validating the result before continuing. The logic for this is deterministic — encoded within the application itself.
 
![workflow-pattern.png]({{ site.baseurl }}/sbreingan/assets/workflow-pattern.png)
 
We can use AWS Step Functions to coordinate the calls to different models. A classification step determines the path, different branches use different models based on complexity, and a validation step checks the result before it is returned.
 
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
 
![bedrock-agent.png]({{ site.baseurl }}/sbreingan/assets/bedrock-agent.png)
 
The diagram shows a Bedrock Agent reasoning in a loop — calling tools, evaluating results, and deciding the next action at each step. In AWS, Bedrock Agents is a managed service for creating these agents, where you define its role and what it has access to, without needing to maintain or deploy infrastructure.
 
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
Respond only in the specified JSON format."
~~~
 
This allows an application to classify incoming requests for further processing and triage.
 
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
 
That is Retrieval-Augmented Generation: the model generates its answer using retrieved material, that *augments* the input, providing additional context beyond its training data.
 
Retrieval can take several forms. The most common is vector search over chunked documents — essentially the system searches for sections of a document that are closest to the input query, after applying an *embedding* algorithm across both. Retrieval does not have to mean vector search alone. Keyword search, hybrid search, and structured retrieval from knowledge graphs or databases can all play the same architectural role.
 
What matters is not the retrieval mechanism by itself, but that relevant information is selected before generation rather than hoping the model already "knows" the answer.
 
![rag-pattern-kb.png]({{ site.baseurl }}/sbreingan/assets/rag-pattern-kb.png)
 
The diagram shows the retrieve-then-generate flow using Amazon Bedrock Knowledge Bases. The application retrieves relevant chunks from indexed documents, then passes them alongside the query to the model for generation. Amazon Bedrock Knowledge Bases automates the embedding and indexing process across data sources including S3, and can use vector stores such as OpenSearch Serverless.
 
#### Considerations
 
- Output quality depends on the full pipeline — how you chunk, embed, index, and retrieve — not just the model
- A system can retrieve the right document and still produce a poor answer — for example if the wrong section of the document is prioritised or context is lost
- Testing end-to-end quality is essential for understanding where the pipeline is failing and where to invest effort
---
 
### Tools and Live Data
 
Documents are not the only source of context. Sometimes the model needs access to live systems: databases, APIs, calculators, case records, or business operations.
 
Whilst RAG allows generated responses to be augmented with additional context, in a world of agents, we can go one step further and dynamically call live systems — the agent can reason about what to do, beyond just augmenting the input.
 
Tools have definitions that form part of the model's context — the model needs to know what capabilities are available. When a tool is called, the result comes back into the context window and becomes new information the model can reason over.
 
#### Considerations
 
- Useful when the model needs live or transactional data
- Tool design, permissions, and failure handling matter
- More capability usually means more complexity and more risk
---
 
## Choosing Your Pattern
 
To show how these axes combine in practice, here are two examples that could use different patterns.
 
### FOI Request Assistant — Workflow + Retrieval
 
A department wants to help staff draft Freedom of Information responses using internal guidance and policy material.
 
**Orchestration: Workflow.** The process follows a controlled sequence: classify the request by sensitivity, route to an appropriate model, retrieve relevant guidance, draft a response, and validate before returning it. A single call gives too little control over each stage. An agent is actively undesirable — FOI responses may carry legal compliance obligations, so the reasoning path needs to be deterministic and fully traceable, not left to an LLM to improvise.
 
**Context: Prompt + Retrieval.** The prompt defines role, constraints, and format. Retrieval supplies internal policies, templates, and guidance so the model drafts against real material rather than general knowledge.
 
This combination keeps the process predictable, auditable, and cost-efficient through model routing, while each stage can be tested and monitored independently.
 
### Planning Research Assistant — Agent + Retrieval + Tools
 
Now consider something less predictable. A planning department wants to assess whether a development proposal complies with local policy. The system may need to search planning guidance, inspect flood-risk data, look up precedent decisions, and synthesise what it finds — but the right sequence depends on what emerges along the way.
 
**Orchestration: Agent.** A fixed workflow would be brittle here. The steps depend on the nature of the proposal and what early searches return, so the system needs to reason about what to do next at each step.
 
**Context: Retrieval + Tools.** Policy documents come via RAG. Live data — flood maps, land registry records, previous decisions — can be accessed through tools. Both feed into the agent's context as it works.
 
This is more powerful but harder to operate. The non-deterministic flow demands careful thought about bounding (how many steps? what budget?), observability (what did the agent actually do?), and safety (what if it calls the wrong tool or misinterprets a result?).
 
---
 
## Cross-Cutting Concern: Guardrails
 
Whatever pattern you choose, production systems need guardrails — checks on inputs and outputs to keep behaviour within acceptable bounds. The key architectural point is that guardrails should be a **separate layer**, not buried in prompts. Prompt instructions can be ignored or worked around by adversarial inputs; a dedicated guardrail layer applies consistently regardless of what the model tries to do. In AWS, Bedrock Guardrails provides a managed option covering content filters, denied topics, prompt attack detection, PII redaction, and grounding checks, applied as a policy on the invocation rather than in the prompt.
 
How guardrails map onto orchestration patterns is where the architectural thinking matters. For a **simple call**, a single policy on the invocation may suffice — check the input, check the output. For a **workflow**, guardrails apply at stage boundaries, each tuned to the specific risk at that stage. For an **agent**, guardrails need to apply on every iteration of the loop, not just the final output — an agent that calls tools and reasons over results has multiple opportunities to drift before producing a response. The more autonomy the model has, the more important and operationally complex guardrails become.
 
---
 
## Choosing Your Approach
 
Designing an LLM system means making deliberate choices about orchestration and context — and resisting the pull toward unnecessary complexity.
 
A single call with a good prompt solves more problems than people sometimes expect. A workflow becomes useful when you want explicit control over stages, routing, model choice, or validation. Agents become valuable when the path genuinely depends on what the model discovers at runtime.
 
The important thing is not to reach for the most sophisticated-sounding pattern. It is to add complexity only on the axis where the simpler approach is genuinely falling short. Start simple, evaluate whether the outputs meet the bar, and move along the axes only when you have evidence that the current approach is insufficient.
 