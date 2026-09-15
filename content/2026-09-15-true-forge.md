Title: Beyond the Managed Agent: How TrueForge Cuts the Cord on Model and Tool Lock-In
Date: 2026-09-15
Category: Article
Tags: AI Agents, Open Source, Agent Harness, TrueFoundry, MCP, Self-Hosted AI, LLM Infrastructure, DevTools
Slug: trueforge-open-source-agent-harness

# **TrueForge: An Open-Source Answer to Vendor Lock-In in Managed Agents**

Managed agent platforms solve a real problem — nobody wants to hand-build streaming, sandboxing, tool orchestration, and session state from scratch every time they want an AI agent to do real work. But that convenience has historically come with a catch: you use the vendor's model, the vendor's tools, and the vendor's sandbox, all running on the vendor's infrastructure. Until recently, there wasn't a serious way to get the same agent-runtime convenience while keeping everything local and under your own control.

That's the gap TrueForge is built to close.

# **What TrueForge Actually Is**

TrueForge is an open-source agent harness — not a model, not a framework, but the runtime layer that sits between a raw language model and a working agent. Released by TrueFoundry, a San Francisco-based B2B machine learning startup founded in 2021 by former Meta engineers, TrueForge runs the full agent execution loop: model calls, MCP tool integration, skills, sandboxing, approval checkpoints, context management, and persistent session state. It exposes all of that through three surfaces — a chat UI, an HTTP API with a TypeScript SDK, and an embeddable UI SDK you can drop into your own application.

The distinction between a "model" and a "harness" is worth sitting with. A model reasons but doesn't act — ask it to do something and, on its own, you get a plan back and nothing else. It can't open a file, hit an API, execute code, or remember what happened three turns ago. The harness is what closes that loop: it plans, calls tools, executes, feeds results back, repeats, and does so while enforcing sandboxing boundaries, requiring human approval where needed, and keeping session state intact across reconnects.

# **Fully Open, No Held-Back Tier**

The headline detail is licensing: TrueForge is released under the MIT License, and TrueFoundry has been explicit that this isn't a stripped-down free tier with the valuable pieces reserved for a paid product. The entire runtime — sandboxing, approvals, context management, the works — is in the public repository. You can clone it and run it locally with a single command (npx @truefoundry/trueforge).

This matters because it changes the default assumption around managed agents. Instead of committing to one vendor's model, one vendor's toolset, and one vendor's hosted sandbox, TrueForge treats all three as swappable, bring-your-own components:

1. **Models** — Anthropic, OpenAI, Google Gemini, other catalog providers, and any OpenAI-compatible endpoint are supported out of the box. TrueForge is explicitly model-agnostic; when a cheaper or better model ships, you point at it instead of rebuilding the agent.

2. **Tools** — any MCP server or API your workflows already depend on can be connected.

3. **Sandboxes** — you run execution in your own compute environment rather than being locked into a hosted one.

For anyone who wants to run agents against local, self-hosted models rather than a cloud API, TrueForge is built to work with local model runners like Ollama — meaning the entire pipeline, from model inference to tool execution, can live entirely on your own machine.

# **An Efficient Harness Design**

One of TrueForge's more interesting architectural choices is how it treats sandboxes. Most agent harnesses keep the agent running inside a sandbox for the full session. TrueForge instead treats the sandbox as a tool that only spins up when the agent actually needs to execute code — which means a single server can support many concurrent agents, and turns that don't involve code stay cheap.

That design philosophy shows up in the benchmark numbers TrueFoundry has published. In one comparison, the same model produced comparable accuracy while using roughly a third of the tokens compared to a more conventional harness — a concrete illustration of the company's broader argument: the harness, not just the model, is often what decides your actual operating cost.

TrueFoundry backed this up with a formal benchmark against Claude Managed Agents using DevRev's Enterprise-Bench, a 14-task benchmark covering multi-step tool use across CRM, issue tracking, and document management systems:

1. Running the same model through TrueForge came in around **30% cheaper** than Claude Managed Agents at comparable quality.

2. Pairing TrueForge with an **open-weight model** (GLM-5.2) completed 11 of 14 benchmark tasks at roughly **75% lower cost** — reported as $2.90 versus $11.80 for a comparable Claude Managed Agents run on Claude Opus 4.8.

Those are TrueFoundry's own reported figures rather than independently audited numbers, so it's worth treating them as a vendor's benchmark rather than gospel — task selection and configuration choices can swing cost comparisons significantly. Independent reviewers have generally corroborated the open-source, vendor-neutral premise while cautioning that "open source" removes the license fee but doesn't remove the real costs of running a production deployment: model API usage, sandbox compute, databases, hosting, observability, and ongoing operator time all still apply.

# **Where It Fits**

TrueForge works standalone, but it's also designed to connect to TrueFoundry's broader AI Gateway for teams that need governance on top of a self-hosted setup — role-based access control, budgets, credential management, and policy-checked tool calls. That's optional infrastructure, not a requirement to use the harness itself.

For a solo developer who wants an agent running locally in one command, or a platform team looking to take a prototype into governed production without being locked into a single model provider, TrueForge represents a fairly direct bet: that the harness — the layer governing how an agent plans, calls tools, and manages context — is becoming as consequential a decision as the underlying model, and that it shouldn't require vendor lock-in to get right.