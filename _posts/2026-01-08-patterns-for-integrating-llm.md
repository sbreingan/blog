---
title: Patterns for building with LLMs
date: 2026-01-08 00:00:00 Z
categories:
- Tech
summary: In this blog I look at the landscapes of different architectural patterns available for using LLMs.
author: sbreingan
---

Large Language Models are being integrated into applications and services across industries. But what does this actually look like architecturally? What are the main approaches available?

In this blog post, I'll give a high-level view of the key architectural patterns for LLM integration. These patterns represent different ways of structuring how your application interacts with and leverages LLMs, from simple API calls to complex agent-based systems.

This is a rapidly evolving space - that's why [architecture is exciting again](https://blog.scottlogic.com/2023/05/04/generative-ai-solution-architecture.html)! The patterns I describe here reflect how LLM usage has evolved so far. Some architectural principles may persist, but I expect new patterns and approaches to emerge as the field progresses and as models themselves become more capable.

I've kept these patterns high-level, focusing on the core approaches. In practice, each pattern brings significant architectural choices around validation, testing, observability, security, and risk mitigation. Understanding which pattern fits your use case is just the beginning - the real architectural work lies in how you implement these patterns safely and reliably in production. That's where the most interesting decisions emerge, and where patterns continue to evolve.

To make these patterns concrete, I've included implementation examples using AWS (where my experience lies). However, these patterns are platform-agnostic - the core approaches apply whether you're working with AWS Bedrock, Azure OpenAI, Google Vertex AI, or self-hosted infrastructure.

---

## Pattern #1: The LLM Wrapper

Large Language Models have been trained on huge amounts of data and know a lot about the world. We see this power in ChatGPT or Claude - you talk to them and they can write, reason, analyze, and respond across countless domains.

The first step to integrating LLMs is simple: **treat the LLM as an API** that brings that power into your application or service.

How do we make it behave in a way specific to our domain? **We tell it.** 
This is where _prompt engineering_ becomes relevant - much of the design is in how we write our system prompts, how we instruct it to behave, and what context we provide. This is all text given to the model.

Every time we call the LLM, we provide this context. These models have limited context windows so we need to think carefully about what we include and how we want the model to respond.

### Example: Citizen Enquiry Triage

A local council receives hundreds of enquiries daily. A simple LLM wrapper could handle initial triage:

~~~ python
System Prompt:
"You are a council enquiry triage assistant. Classify incoming citizen 
enquiries into: Planning, Waste Services, Council Tax, Housing, Highways, Other. 

For each enquiry provide category, urgency (Low/Medium/High), and brief summary.
Respond only in JSON format."
~~~

**Input:** `My bin hasn't been collected for two weeks`

**Output:** `{"category": "Waste Services", "urgency": "Medium", "summary": "Missed collection"}`

### Implementation

In AWS, Bedrock provides API access to frontier models. You can call it from a Lambda or your application:

~~~ python
import boto3

bedrock = boto3.client('bedrock-runtime')

response = bedrock.converse(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    messages=[
        {'role': 'user', 'content': [{'text': user_input}]}
    ],
    system=[{'text': system_prompt}],
    inferenceConfig={'temperature': 0.0, 'maxTokens': 500}
)
~~~

### Limitations

This pattern works well for contained tasks, but:

- Context window limits constrain how much information you can provide
- No access to external organizational data
- Token costs scale with context size
- Output format consistency requires careful prompting

When you need access to large document stores or organizational data, you need retrieval capabilities.

---

## Pattern #2: The Orchestrator

Now let's say you have multiple steps to perform. Rather than "call the LLM once," you want to make further calls based on responses, potentially using **different models** for different tasks.

Suppose you're handling Freedom of Information (FOI) requests. You could write one large system prompt to handle everything, but this has downsides. Reasoning models like Claude Sonnet cost significantly more (around $3-15 per million tokens) compared to lightweight models like Haiku (around $0.25-1.25 per million tokens).

If most FOI requests are simple, you can save time, money, and energy by routing to cheaper models. **Orchestration** means splitting into multiple LLM calls, checking outputs, and choosing the right model and prompt based on what came before.

### Example: Orchestrated FOI Handler

~~~python
# Step 1: Classify with fast, cheap model
classification = bedrock.converse(
    modelId='anthropic.claude-3-5-haiku-20241022-v1:0',
    messages=[{'role': 'user', 'content': [{'text': f'Classify as simple or complex: {request}'}]}]
)

complexity = classification['output']['message']['content'][0]['text']

# Step 2: Route to appropriate model
if complexity == 'simple':
    model = 'anthropic.claude-3-5-haiku-20241022-v1:0'  # Fast, cheap
else:
    model = 'anthropic.claude-3-5-sonnet-20241022-v2:0'  # Powerful, expensive

response = bedrock.converse(modelId=model, messages=[...])
~~~

This trades some latency (multiple calls) for cost efficiency and appropriate model selection. However, it still relies on knowledge baked into the models. When you need organizational data - policies, records, documentation - you need retrieval.

---

## Pattern #3: The Retriever

So far, our LLM calls rely on:

- Knowledge from training
- Context provided in our prompts

But real power often comes from applying LLMs to proprietary organizational data. You may have many documents too large or numerous to include as context.

This is where **Retrieval-Augmented Generation (RAG)** comes in - "when generating a response, retrieve relevant information first."

### 3a: RAG - Document Retrieval

Perhaps you have hundreds of planning policy documents. When enquiries arrive, you want to generate responses using the most relevant ones. How does the LLM search hundreds of documents?

We use **embedding**:

- Chunk documents into pieces (paragraphs/sections, typically 500-1000 tokens with overlap)
- Convert chunks into vectors using an embedding model
- Store vectors in a vector database

When you receive a query:

- Embed the query into numbers
- Find nearest document chunks using vector similarity
- Add relevant chunks to the LLM's context

#### Example: Planning Permission Assistant

~~~ python
bedrock_agent = boto3.client('bedrock-agent-runtime')

# Retrieve relevant chunks from Knowledge Base
retrieval = bedrock_agent.retrieve(
    knowledgeBaseId='your-kb-id',
    retrievalQuery={'text': question},
    retrievalConfiguration={
        'vectorSearchConfiguration': {'numberOfResults': 5}
    }
)

# Extract context and pass to LLM
context = '\n\n'.join([r['content']['text'] for r in retrieval['retrievalResults']])

response = bedrock.converse(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    messages=[{'role': 'user', 'content': [{'text': f'Context:\n{context}\n\nQuestion: {question}'}]}]
)
~~~

Managed services like AWS Bedrock Knowledge Bases handle chunking, embedding, and retrieval automatically - you simply point it to an S3 bucket of documents.

**Limitations:** Document chunking can split important context, semantic search isn't perfect and might miss relevant content, and hallucination risk remains when retrieved context is incomplete.

### 3b: Knowledge Graphs - Database Retrieval

When data is highly relational - where connections between entities matter as much as content - you need a different approach.

**Knowledge graphs** store data and their relationships. They use query languages like Cypher to query relationships. Rather than embedding, we use an LLM to transform natural language into Cypher queries.

#### Example: Regulatory Compliance Navigator

~~~ python
# Step 1: Convert natural language to Cypher using LLM
cypher_response = bedrock.converse(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    messages=[{'role': 'user', 'content': [{
        'text': f'Convert to Cypher query for regulatory compliance graph: {question}'
    }]}]
)

cypher_query = cypher_response['output']['message']['content'][0]['text']

# Step 2: Execute against Neptune graph database
results = graph_client.run(cypher_query)

# Step 3: Format results with LLM
formatted = bedrock.converse(
    messages=[{'role': 'user', 'content': [{'text': f'Format these results: {results}'}]}]
)
~~~

**Why graphs over relational databases?** While you could use text-to-SQL with traditional databases, Cypher queries map more naturally to natural language. Relationships in graphs are first-class citizens - "which regulations apply to public sector in Scotland" translates more directly to graph traversal than complex SQL joins.

### 3c: Hybrid RAG

More recently, patterns combine both approaches - documents mapped to entities, allowing queries against both graph and vector databases for more accurate retrieval.

Different patterns exist, but they use both structured relationships and unstructured context to better find relevant information. For example: find related entities in the graph, then use those entities to search the vector database for relevant document sections. Combine both for the LLM response.

This overcomes limitations of each method - you get relational context from graphs and detailed textual information from document retrieval.

However, even with sophisticated retrieval, there are scenarios where you need the model itself to *learn* domain-specific patterns. This is where custom training comes in.

---

## Pattern #4: The Agent

We've moved from single LLM calls to orchestrated calls to using LLMs to query databases. But in all these patterns, the workflow is predetermined - in orchestration, your code decides the flow; in retrieval, it's always retrieve-then-generate.

Agents take a different approach: the LLM itself decides which actions to take and when. Document retrieval or database queries become tools the agent can choose to use, alongside other actions. The agent might retrieve from a knowledge base, call an external API, do both in sequence, or neither - it reasons about what the input requires rather than following a fixed workflow.

This is the **agent pattern** - give the LLM access to various tools and let it decide which to use.

We're essentially saying:

    - The LLM agent has access to tools with descriptions and input schemas
    - It receives input and maps it to potential actions
    - If tools are relevant, it calls them and awaits responses
    - It repeats until fully processing the input
    - It returns the result

Under the hood, it's still orchestrated LLM calls, but the model's output serves as input to external actions.

### Example: Council Service Request Handler

An agent handling multi-step citizen requests might have access to:

~~~ python
tools = [
    {
        'name': 'check_bin_schedule',
        'description': 'Check waste collection schedule for an address',
        'inputSchema': {
            'type': 'object',
            'properties': {
                'postcode': {'type': 'string', 'description': 'UK postcode'}
            }
        }
    },
    {
        'name': 'report_missed_collection',
        'description': 'Report a missed bin collection',
        'inputSchema': {
            'type': 'object',
            'properties': {
                'postcode': {'type': 'string'},
                'collection_type': {'type': 'string', 'enum': ['general', 'recycling']},
                'missed_date': {'type': 'string'}
            }
        }
    }
]
~~~

### Implementation

~~~ python
bedrock_agent = boto3.client('bedrock-agent-runtime')

response = bedrock_agent.invoke_agent(
    agentId='your-agent-id',
    agentAliasId='your-alias-id',
    sessionId='unique-session-id',
    inputText=user_request
)

# Agent autonomously:
# 1. Understands intent
# 2. Calls appropriate tools in sequence
# 3. Uses outputs to inform next actions
# 4. Returns final result
~~~

**Example request:** "My recycling wasn't collected last Tuesday at SW1A 1AA. Can you check when it's next due and report it?"

The agent would: check the schedule, report the missed collection, and confirm the next date.

### Tool Standards: MCP

**Model Context Protocol (MCP)** is emerging as a standard for defining tools that LLMs can use. MCP servers expose tools with descriptions and schemas, similar to the example above. This allows tools to be defined once and used across different LLM providers.

An MCP server for council services might expose multiple tools (bin schedules, service requests, payment checks) that any MCP-compatible agent can discover and use. This standardization makes it easier to build reusable tool libraries.

### Limitations

Agents are powerful but have significant limitations:

    - **Tool hallucination**: Might call non-existent tools or use incorrect parameters
    - **Observability**: Difficult to debug why particular action sequences were chosen
    - **Cost**: Iterative LLM calls become expensive quickly
    - **Non-determinism**: Same input might produce different tool sequences
    - **Description quality**: Vague tool descriptions lead to incorrect selection

The more freedom given to take actions, the more care needed with monitoring and understanding behavior.

---

## Pattern #5: Custom Training

While we can retrieve data and provide tool access, the final pattern is to fine-tune the model on your organizational data to **change its behavior**.

This means teaching it task-specific behavior, domain language, and output formats - not just giving it access to facts.

### Traditional ML vs LLM Fine-Tuning

Traditional machine learning:
```
Small dataset (thousands of examples)
→ Extract features
→ Train small model (thousands of parameters)
→ Predict outcomes
```

But how do we apply this to models with **billions** of parameters?

We want domain-specific understanding without losing general language knowledge. We don't want to start from scratch or modify all billions of weights.

### Fine-Tuning with LoRA

**LoRA (Low-Rank Adaptation)** doesn't retrain all parameters. Instead, we add small adapter layers on top and train only those.

Think of it as:

- **Base model knows English grammar** (frozen - billions of parameters)
- **Adapter adds medical terminology** (new - millions of parameters)

You don't need to relearn grammar to learn terminology.

The new information is "low rank" - it doesn't override fundamentals but adapts behavior for your specific use case.

### Example: Medical Report Generation

Train on 5,000 radiologist reports showing structure, terminology, and phrasing:

~~~ python
# Training data format
{
    "prompt": "Patient presents with acute chest pain, left arm radiation, diaphoresis",
    "completion": "CLINICAL FINDINGS:\n\nPatient History: 67M with acute chest pain\n..."
}

# Fine-tune through Bedrock
# Upload training data to S3, create customization job
# Access fine-tuned model via custom endpoint

response = bedrock.converse(
    modelId='arn:aws:bedrock:region:account:provisioned-model/your-custom-model',
    messages=[{'role': 'user', 'content': [{'text': clinical_findings}]}]
)
~~~

The fine-tuned model now uses correct medical terminology, follows hospital report structure, and maintains consistent formatting.

### When to Fine-Tune

Fine-tuning works well for:

    - Teaching specific output formats (JSON schemas, report structures)
    - Domain-specific language (legal, medical, technical terminology)
    - Style and tone consistency
    - Task-specific patterns (entity extraction, classification)

Fine-tuning does NOT work for:

    - Adding new factual knowledge (use RAG instead)
    - Fixing hallucinations (may make worse)
    - Information that changes frequently

For facts and current information, use retrieval. For patterns and behaviors, use fine-tuning. Often, you'll use both: RAG for facts, fine-tuning for format and style.

### AWS Implementation

**Bedrock Model Customization:** Upload training data to S3, create customization job, access via custom endpoint. Cost: ~$4-8 per 1,000 training tokens.

**SageMaker:** Use Hugging Face libraries with open models (Llama, Mistral) for more control. Cost: ~$1.50-8/hour for training instances.

---

## Cross-Cutting Concern: Guardrails

Regardless of which pattern you choose, production LLM systems need protective layers. Think of guardrails as safety systems wrapping any LLM integration.

### What Guardrails Provide

**Input Protection:**

    - Content filtering to block inappropriate requests
    - PII detection and redaction before reaching the model
    - Prompt injection defense against malicious override attempts

**Output Validation:**

    - Content policy enforcement for organizational standards
    - Format validation for expected schemas
    - Hallucination detection flagging confident but inaccurate responses

**Compliance & Governance:**

    - Audit trails for all interactions
    - Rate limiting to prevent runaway costs
    - Access control ensuring authorized use only

AWS Bedrock Guardrails can be configured with content filters, denied topics, word filters, PII redaction, and contextual grounding checks by applying a config parameter to API calls. 

Guardrails aren't optional for production systems - they're essential protective measures for every pattern discussed.

---

## Conclusion

These five patterns represent the current landscape of LLM integration:

1. **The Wrapper**: Treat the LLM as an API, control through prompting
2. **The Orchestrator**: Chain multiple calls, routing to appropriate models
3. **The Retriever**: Connect to your data through RAG, knowledge graphs, or hybrid approaches
4. **The Agent**: Give LLMs tool access and let them reason about actions
5. **Custom Training**: Fine-tune models for domain patterns and formats

Start simple - often just a well-crafted wrapper with good prompting. Add complexity only when simpler approaches don't work. The typical journey: wrapper → orchestration → retrieval → fine-tuning → agents.

The field evolves rapidly, but these fundamental patterns provide a solid framework for thinking about LLM architecture. Specific services and implementations will change, but core architectural decisions remain consistent.

Regardless of pattern choice, production systems require guardrails, monitoring, and careful integration into your broader architecture.

