# AI Agent SDKs & Frameworks Comparison

> Last updated: 2026-02-08 | Research by Scout

## Executive Summary

The AI agent framework landscape has matured rapidly. This report compares 10 major frameworks across architecture, multi-agent orchestration, strengths, weaknesses, and pricing. The key insight: **there is no single best framework** — the right choice depends on your language ecosystem (Python vs TypeScript), orchestration needs (autonomous vs deterministic), and whether you want vendor lock-in for simplicity or flexibility at the cost of complexity.

**Quick picks:**
- **TypeScript teams → Mastra or Vercel AI SDK**
- **Python + maximum control → LangGraph**
- **Python + fast multi-agent prototyping → CrewAI**
- **Enterprise/.NET → Semantic Kernel**
- **OpenAI-native → OpenAI Agents SDK**
- **Research/optimization → DSPy**
- **Microsoft ecosystem → AutoGen or Semantic Kernel**

---

## Framework Comparison Matrix

| Framework | Language | Multi-Agent | Orchestration Style | Open Source | Vendor Lock-in |
|-----------|----------|-------------|-------------------|-------------|---------------|
| LangGraph | Python, JS | ✅ | Graph-based (explicit) | ✅ MIT | Low |
| CrewAI | Python | ✅ | Role-based (autonomous) | ✅ MIT | Low |
| AutoGen | Python | ✅ | Conversational | ✅ MIT | Low |
| OpenAI Agents SDK | Python | ✅ | Handoffs | ✅ MIT | High (OpenAI) |
| Anthropic (Claude) | API-based | ⚠️ Emerging | Tool-use loops | N/A | High (Anthropic) |
| Mastra | TypeScript | ✅ | Agents + Workflows | ✅ MIT | Low |
| Vercel AI SDK | TypeScript | ⚠️ Limited | Tool-calling loops | ✅ Apache 2.0 | Low |
| Semantic Kernel | Python, C#, Java | ✅ | Plugin-based | ✅ MIT | Medium (Azure) |
| DSPy | Python | ⚠️ Limited | Declarative/compiled | ✅ MIT | Low |
| Pi SDK | N/A | ❌ | N/A | ❌ | High (Inflection) |

---

## 1. LangGraph (LangChain)

**Repository:** [github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)  
**Language:** Python (primary), JavaScript  
**License:** MIT  

### Architecture
LangGraph models agent workflows as **directed graphs** inspired by Google's Pregel and Apache Beam. Nodes represent computation steps (LLM calls, tool invocations, custom logic), and edges define transitions. State is explicitly managed through a typed `State` dictionary that flows through the graph.

Key architectural concepts:
- **StateGraph**: The core abstraction — define nodes, edges, and state schema
- **Checkpointing**: Built-in persistence for durable execution; agents can resume from failures
- **Subgraphs**: Compose graphs within graphs for modular multi-agent systems
- **Conditional edges**: Dynamic routing based on state

### Multi-Agent Orchestration
- Agents are composed as **subgraphs** within a parent graph
- Supports supervisor patterns, hierarchical delegation, and peer-to-peer collaboration
- State is shared across agents via the graph's state object
- Human-in-the-loop via interrupts at any node

### Strengths
- **Maximum control**: You define exactly how agents interact — no magic
- **Durable execution**: Agents survive failures, can pause/resume indefinitely
- **Excellent debugging**: LangSmith integration provides full trace visibility
- **Battle-tested**: Used by Klarna, Replit, Elastic in production
- **Model-agnostic**: Works with any LLM via LangChain integrations
- **Rich ecosystem**: LangSmith for observability, LangGraph Studio for visual prototyping

### Weaknesses
- **Steep learning curve**: Graph-based thinking is unintuitive for simple use cases
- **Verbose**: Simple agents require more boilerplate than higher-level frameworks
- **LangChain ecosystem complexity**: While standalone, the broader ecosystem can be overwhelming
- **Over-engineering risk**: Tempting to build complex graphs when simpler approaches suffice

### Pricing
- **Framework**: Free, open source (MIT)
- **LangSmith** (observability): Free tier → paid plans from ~$39/mo
- **LangGraph Cloud** (deployment): Usage-based pricing, contact sales
- **LLM costs**: Pass-through to your chosen provider

---

## 2. CrewAI

**Repository:** [github.com/crewAIInc/crewAI](https://github.com/crewAIInc/crewAI)  
**Language:** Python  
**License:** MIT  

### Architecture
CrewAI uses a **role-based** metaphor: you define **Agents** (with roles, goals, and backstories), **Tasks** (units of work), and **Crews** (teams that execute tasks). Two complementary systems:

- **Crews**: Autonomous agent teams that collaborate through natural language delegation
- **Flows**: Event-driven, production-grade workflows with explicit control flow — `@start`, `@listen` decorators for routing

Built entirely from scratch — no LangChain dependency (as of recent versions).

### Multi-Agent Orchestration
- **Sequential**: Agents execute tasks in order
- **Hierarchical**: A manager agent delegates to worker agents
- **Consensual** (experimental): Agents discuss and reach consensus
- Agents can delegate tasks to each other dynamically
- Crews can be embedded within Flows for hybrid autonomous/deterministic execution

### Strengths
- **Intuitive mental model**: Role-playing metaphor is easy to grok
- **Fast prototyping**: Get multi-agent systems running in minutes
- **100K+ developer community** with courses on DeepLearning.AI
- **Standalone**: No dependency on LangChain
- **Flows for production**: Event-driven architecture bridges the gap between prototype and production
- **Enterprise offering**: CrewAI AMP Suite with tracing, observability, on-premise deployment

### Weaknesses
- **Less granular control**: The abstraction hides execution details
- **Prompt sensitivity**: Role descriptions and backstories significantly affect output quality
- **Token-heavy**: Autonomous collaboration burns more tokens than deterministic workflows
- **Debugging difficulty**: Autonomous agent interactions can be hard to trace without paid tools
- **Python-only**: No JavaScript/TypeScript support

### Pricing
- **Framework**: Free, open source (MIT)
- **CrewAI Cloud** (Control Plane): Free trial available, enterprise pricing on request
- **CrewAI AMP Suite**: Enterprise licensing (contact sales)
- **LLM costs**: Pass-through

---

## 3. AutoGen (Microsoft)

**Repository:** [github.com/microsoft/autogen](https://github.com/microsoft/autogen)  
**Language:** Python  
**License:** MIT  
**Note:** Microsoft has announced the newer [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) as the recommended starting point for new projects. AutoGen continues to receive maintenance and security patches.

### Architecture
AutoGen uses a **layered, extensible design**:

- **Core API**: Message-passing, event-driven agents with local and distributed runtime
- **AgentChat** (high-level): Pre-built agent types and team patterns for common scenarios
- **Extensions**: Model clients, tool integrations (MCP, etc.)
- **AutoGen Studio**: No-code GUI for prototyping multi-agent workflows

Key concepts:
- **AssistantAgent**: LLM-powered agent with tools
- **AgentTool**: Wraps an agent as a callable tool for another agent
- **Teams**: Round-robin, selector-based, swarm, and custom orchestration patterns

### Multi-Agent Orchestration
- **AgentTool pattern**: Agents wrapped as tools for a coordinator agent
- **RoundRobinGroupChat**: Agents take turns
- **SelectorGroupChat**: An LLM selects which agent speaks next
- **Swarm**: Agents hand off based on context
- **MagenticOneGroupChat**: A lead agent orchestrates specialists
- Supports distributed runtime for scaling across processes/machines

### Strengths
- **Microsoft backing**: Enterprise credibility, active development
- **AutoGen Studio**: No-code prototyping is excellent for non-developers
- **Distributed runtime**: Production-ready scaling across machines
- **Flexible orchestration**: Multiple built-in team patterns
- **MCP integration**: First-class Model Context Protocol support
- **Research pedigree**: Strong academic foundation from Microsoft Research

### Weaknesses
- **Transition confusion**: AutoGen v0.2 → v0.4 was a major rewrite; documentation can be confusing
- **Microsoft Agent Framework overlap**: Unclear long-term positioning vs the newer framework
- **Complexity**: The layered architecture has a learning curve
- **Verbose API**: More ceremony than lighter alternatives
- **Ecosystem fragmentation**: Multiple packages to install and configure

### Pricing
- **Framework**: Free, open source (MIT)
- **Azure integration**: Optional, usage-based Azure pricing
- **LLM costs**: Pass-through

---

## 4. OpenAI Agents SDK

**Repository:** [github.com/openai/openai-agents-python](https://github.com/openai/openai-agents-python)  
**Language:** Python  
**License:** MIT  

### Architecture
The Agents SDK (successor to Swarm) is intentionally **minimal** with very few primitives:

- **Agent**: An LLM with instructions, tools, and optional output schema
- **Handoffs**: Agents can delegate to other agents (the core multi-agent mechanism)
- **Guardrails**: Input/output validation that runs in parallel with agent execution
- **Runner**: Executes the agent loop (tool calls → LLM → repeat until done)
- **Sessions**: Persistent memory across agent runs

Built on top of the **Responses API**, which combines Chat Completions simplicity with Assistants API tool-use capabilities. Comes with built-in tools: web search, file search, computer use.

### Multi-Agent Orchestration
- **Handoffs**: An agent can hand control to another agent with context
- **Agents-as-tools**: Wrap an agent as a tool callable by another agent
- Orchestration is Python-native — use if/else, loops, etc.
- No complex graph or team abstractions — just function composition

### Strengths
- **Extremely simple**: Minimal abstractions, quick to learn
- **Built-in tracing**: Visualize and debug workflows in OpenAI dashboard
- **Built-in tools**: Web search, file search, computer use out of the box
- **Guardrails**: First-class input/output validation
- **Realtime agents**: Voice agent support with interruption detection
- **Fine-tuning pipeline**: Traces can be used to fine-tune models
- **Production successor to Swarm**: Proven patterns, production-ready

### Weaknesses
- **OpenAI lock-in**: Designed primarily for OpenAI models (though extensible)
- **Limited orchestration patterns**: No built-in supervisor, round-robin, or complex team patterns
- **Python-only**: No TypeScript SDK
- **New/evolving**: API surface may change
- **No built-in persistence**: Sessions are basic compared to LangGraph's checkpointing

### Pricing
- **SDK**: Free, open source (MIT)
- **API costs**: Standard OpenAI pricing (tokens + tool usage)
  - GPT-4o: ~$2.50/$10 per 1M input/output tokens
  - Web search: Additional per-search cost
  - File search: $0.10/GB/day storage + retrieval costs
- **Tracing/Evals**: Included in OpenAI platform (usage-based)

---

## 5. Anthropic Claude (Agentic Tool Use)

**Language:** API-based (Python/TypeScript SDKs)  
**License:** Proprietary API  

### Architecture
Anthropic does not offer a standalone "agent framework" in the same way as others. Instead, they provide **agentic patterns** via the Claude API:

- **Tool use**: Define tools as JSON schemas; Claude decides when to call them
- **Agentic loops**: Your code manages the loop — call Claude, execute tools, feed results back
- **Computer use**: Claude can interact with desktop environments via screenshots
- **Extended thinking**: Claude can reason through complex multi-step problems
- **MCP support**: Model Context Protocol for standardized tool integration

Multi-agent orchestration is DIY — Anthropic provides the model capabilities but not the orchestration framework.

### Multi-Agent Orchestration
- **No built-in multi-agent framework** (as of early 2026)
- Patterns suggested in documentation: orchestrator-workers, evaluator-optimizer
- Third-party frameworks (LangGraph, CrewAI, etc.) commonly used to orchestrate Claude agents
- Claude's strong instruction-following makes it well-suited for agent roles

### Strengths
- **Best-in-class models**: Claude Sonnet/Opus excel at agentic tasks, tool use, and instruction-following
- **Computer use**: Unique capability for GUI automation
- **Extended thinking**: Superior reasoning for complex tasks
- **MCP ecosystem**: Anthropic created the MCP standard
- **No framework overhead**: Direct API access, maximum flexibility
- **Strong safety**: Constitutional AI approach

### Weaknesses
- **No framework**: You build everything yourself or use third-party
- **No multi-agent orchestration**: Must implement from scratch
- **Vendor lock-in**: Tightly coupled to Claude models
- **No built-in tracing/observability**: Requires third-party tools
- **Higher latency**: Extended thinking adds response time

### Pricing
- **SDK**: Free (open source Python/TypeScript SDKs)
- **API costs**:
  - Claude Sonnet 4.5: ~$3/$15 per 1M input/output tokens
  - Claude Opus 4: ~$15/$75 per 1M input/output tokens
  - Claude Haiku 3.5: ~$0.80/$4 per 1M input/output tokens
- **Batch API**: 50% discount for async processing

---

## 6. Mastra

**Repository:** [github.com/mastra-ai/mastra](https://github.com/mastra-ai/mastra)  
**Language:** TypeScript  
**License:** MIT  
**Backing:** YC W25, from the team behind Gatsby  

### Architecture
Mastra is a **TypeScript-first** framework with a comprehensive feature set:

- **Agents**: Autonomous LLM-powered agents with tools, memory, and stopping conditions
- **Workflows**: Graph-based workflow engine with intuitive syntax (`.then()`, `.branch()`, `.parallel()`)
- **Model routing**: Connect to 40+ providers through one interface
- **Memory**: Conversation history, working memory, semantic recall
- **RAG**: Built-in retrieval-augmented generation
- **MCP servers**: Author and consume MCP servers natively
- **Evals & Observability**: Built-in evaluation and tracing

### Multi-Agent Orchestration
- Agents can be composed within workflows
- Workflows provide deterministic orchestration with branching and parallelism
- Human-in-the-loop via suspend/resume with persistent state
- Integrates with Vercel AI SDK UI and CopilotKit for frontend

### Strengths
- **TypeScript-native**: First-class TS experience, great for web developers
- **Comprehensive**: Agents, workflows, RAG, memory, evals — all in one package
- **Modern DX**: `npm create mastra@latest` scaffolding, intuitive API design
- **Framework integration**: Works with React, Next.js, Node.js
- **MCP support**: Both server and client
- **Active development**: YC-backed, fast iteration

### Weaknesses
- **Young project**: Less battle-tested than LangGraph or CrewAI
- **TypeScript only**: No Python support
- **Smaller community**: Fewer examples, tutorials, and third-party resources
- **Enterprise features maturing**: Not yet at the level of LangGraph Cloud or CrewAI AMP

### Pricing
- **Framework**: Free, open source (MIT)
- **Cloud/hosted**: Not yet available (as of early 2026)
- **LLM costs**: Pass-through to your chosen provider

---

## 7. Vercel AI SDK

**Repository:** [github.com/vercel/ai](https://github.com/vercel/ai)  
**Language:** TypeScript  
**License:** Apache 2.0  

### Architecture
The Vercel AI SDK is a **TypeScript toolkit** with two main libraries:

- **AI SDK Core**: Unified API for `generateText`, `generateObject`, `streamText`, tool calls
- **AI SDK UI**: Framework-agnostic hooks (`useChat`, `useCompletion`) for React, Vue, Svelte, etc.

Agent support is via **tool-calling loops** — the SDK handles the loop of calling an LLM, executing tools, and feeding results back until the model stops calling tools. Not a dedicated agent framework, but a powerful building block.

### Multi-Agent Orchestration
- **Limited built-in support**: No first-class multi-agent primitives
- Can compose agents manually using tool calls and routing logic
- Best used as the **LLM interaction layer** within a larger framework (e.g., Mastra uses Vercel AI SDK under the hood)

### Strengths
- **Excellent DX**: Clean, intuitive TypeScript API
- **Provider-agnostic**: 40+ LLM providers through unified interface
- **Streaming-first**: Built for real-time UX with streaming text and structured objects
- **UI integration**: Best-in-class hooks for building chat interfaces
- **Structured output**: First-class `generateObject` for type-safe LLM responses
- **Wide adoption**: De facto standard for AI in the JS/TS ecosystem
- **Vercel ecosystem**: Seamless deployment on Vercel platform

### Weaknesses
- **Not an agent framework**: Lacks agent abstractions, memory, orchestration
- **No multi-agent support**: You build it yourself
- **No workflow engine**: No built-in graph or state machine
- **Frontend-focused**: Core strength is UI integration, not backend orchestration
- **Vercel deployment bias**: Best experience on Vercel platform

### Pricing
- **SDK**: Free, open source (Apache 2.0)
- **Vercel platform**: Free tier → Pro $20/mo → Enterprise custom
- **LLM costs**: Pass-through

---

## 8. Semantic Kernel (Microsoft)

**Repository:** [github.com/microsoft/semantic-kernel](https://github.com/microsoft/semantic-kernel)  
**Language:** Python, C#/.NET, Java  
**License:** MIT  

### Architecture
Semantic Kernel is Microsoft's **enterprise-grade orchestration SDK** for building AI agents:

- **Kernel**: Central orchestrator that manages services, plugins, and memory
- **Plugins**: Modular capabilities (native functions, prompt templates, OpenAPI specs, MCP)
- **Agents**: `ChatCompletionAgent`, `OpenAIAssistantAgent`, and custom agent types
- **Process Framework**: Structured workflow modeling for complex business processes
- **Memory/Vector DB**: Integration with Azure AI Search, Elasticsearch, Chroma, etc.

### Multi-Agent Orchestration
- **AgentGroupChat**: Multiple agents collaborate in a chat-based workflow
- **Agent selection strategies**: Round-robin, function-based selection
- **Termination strategies**: Configurable conditions for ending group chats
- Agents can have different models, plugins, and instructions
- Process Framework for complex multi-step business workflows

### Strengths
- **Multi-language**: Python, C#, Java — unmatched language breadth
- **Enterprise-ready**: Built for Azure ecosystem, production-grade stability
- **Plugin system**: Extremely flexible — native code, prompts, OpenAPI, MCP
- **Structured output**: Pydantic/BaseModel support for typed responses
- **Azure integration**: First-class Azure OpenAI, Azure AI Search, etc.
- **Mature**: Stable APIs, comprehensive documentation
- **Process Framework**: Unique capability for modeling business processes

### Weaknesses
- **Azure-centric**: Best experience with Azure services
- **Verbose**: Enterprise-grade means more boilerplate
- **Complex abstractions**: Kernel, plugins, planners — many concepts to learn
- **.NET bias**: C# gets features first, Python/Java lag slightly
- **Less community buzz**: Fewer tutorials/examples than LangChain ecosystem

### Pricing
- **SDK**: Free, open source (MIT)
- **Azure services**: Pay-as-you-go Azure pricing
- **LLM costs**: Pass-through (Azure OpenAI pricing slightly different from direct OpenAI)

---

## 9. DSPy (Stanford NLP)

**Repository:** [github.com/stanfordnlp/dspy](https://github.com/stanfordnlp/dspy)  
**Language:** Python  
**License:** MIT  

### Architecture
DSPy takes a radically different approach: **programming, not prompting**. Instead of writing prompts, you:

1. Define **Signatures** (input/output specs): `"question -> answer"`
2. Compose **Modules** (behavior patterns): `ChainOfThought`, `ReAct`, `ProgramOfThought`
3. **Compile/optimize**: DSPy automatically generates optimal prompts or fine-tunes weights

Core concepts:
- **Modules**: Composable units like `dspy.ChainOfThought("question -> answer")`
- **Optimizers**: Algorithms that improve prompts/weights based on metrics (e.g., `BootstrapFewShotWithRandomSearch`)
- **Assertions**: Constraints that guide LM behavior
- **Evaluators**: Metric-based evaluation of pipeline quality

### Multi-Agent Orchestration
- **Not a primary focus**: DSPy is about optimizing LM programs, not multi-agent coordination
- Can compose modules into pipelines that resemble agent workflows
- `ReAct` module provides agent-like tool-use loops
- Multi-agent patterns must be built manually on top of modules

### Strengths
- **Paradigm shift**: Eliminates prompt engineering — the compiler does it for you
- **Optimization**: Automatically finds better prompts through compilation
- **Reproducibility**: Declarative programs are more reproducible than prompt strings
- **Model-agnostic**: Works with any LLM via LiteLLM
- **Academic rigor**: Research-backed from Stanford NLP
- **Portability**: Programs transfer across models better than hand-crafted prompts

### Weaknesses
- **Steep learning curve**: The programming-not-prompting paradigm is unfamiliar
- **Not agent-focused**: Limited agent/multi-agent primitives
- **Optimization overhead**: Compilation requires examples and evaluation runs
- **Debugging difficulty**: Compiled prompts can be opaque
- **Smaller ecosystem**: Fewer integrations than LangChain/CrewAI
- **Documentation**: Academic style, can be dense

### Pricing
- **Framework**: Free, open source (MIT)
- **LLM costs**: Pass-through (optimization runs consume additional tokens)

---

## 10. Pi SDK (Inflection AI)

**Language:** N/A (no public SDK)  
**Status:** Not a developer framework  

### Overview
**Pi is a consumer AI assistant by Inflection AI**, not a developer SDK/framework. Despite the name similarity, there is no "Pi SDK" for building agents. Inflection AI pivoted in 2024 to enterprise (partnering with Microsoft), and Pi remains a conversational AI product.

- No public API or SDK for developers
- No multi-agent capabilities
- No framework for building custom agents
- Primarily a chatbot interface at pi.ai

**Verdict**: Not applicable to this comparison. If the intent was a different "Pi SDK," it may refer to a niche or internal tool not widely available.

---

## Multi-Agent Orchestration Patterns Compared

| Pattern | LangGraph | CrewAI | AutoGen | OpenAI SDK | Mastra | Semantic Kernel |
|---------|-----------|--------|---------|------------|--------|----------------|
| Supervisor/Manager | ✅ | ✅ | ✅ | ⚠️ Manual | ✅ | ✅ |
| Peer-to-peer | ✅ | ✅ | ✅ | ✅ Handoffs | ⚠️ | ⚠️ |
| Round-robin | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Dynamic routing | ✅ | ✅ | ✅ | ⚠️ | ✅ | ✅ |
| Human-in-the-loop | ✅ | ⚠️ | ✅ | ✅ | ✅ | ✅ |
| Persistent state | ✅ | ⚠️ | ✅ | ⚠️ | ✅ | ✅ |
| Distributed runtime | ⚠️ Cloud | ❌ | ✅ | ❌ | ❌ | ⚠️ Azure |

---

## Decision Framework

### Choose LangGraph if:
- You need maximum control over agent execution flow
- Durable, long-running workflows are critical
- You want strong observability (LangSmith)
- You're comfortable with graph-based thinking

### Choose CrewAI if:
- You want fast multi-agent prototyping
- The role-playing metaphor fits your use case
- You need autonomous agent collaboration
- You're building proof-of-concepts before production

### Choose AutoGen if:
- You need distributed multi-agent systems
- Microsoft ecosystem alignment matters
- You want a no-code option (AutoGen Studio)
- Research/experimental multi-agent patterns

### Choose OpenAI Agents SDK if:
- You're already committed to OpenAI models
- You want minimal abstractions and fast setup
- Built-in tools (web search, file search) are valuable
- Voice/realtime agents are a requirement

### Choose Mastra if:
- You're a TypeScript team building full-stack AI apps
- You want agents + workflows + RAG in one package
- Next.js/React integration is important
- You want modern DX with MCP support

### Choose Vercel AI SDK if:
- You need the best AI-powered UI components
- Streaming and real-time UX are priorities
- You're building chat interfaces
- You'll pair it with another framework for orchestration

### Choose Semantic Kernel if:
- You're in a .NET/Java/enterprise environment
- Azure integration is a requirement
- Multi-language support matters
- You need structured business process modeling

### Choose DSPy if:
- You want to eliminate prompt engineering
- Automatic optimization of LM programs appeals to you
- You're building research-grade NLP pipelines
- Reproducibility and portability across models matter

---

## Key Trends (Early 2026)

1. **MCP is becoming standard**: Most frameworks now support Model Context Protocol for tool integration
2. **Convergence on patterns**: Supervisor, handoff, and tool-as-agent patterns appear across frameworks
3. **TypeScript catching up**: Mastra and Vercel AI SDK closing the gap with Python-first frameworks
4. **Enterprise features differentiating**: Observability, tracing, and managed deployment are the new battleground
5. **Framework consolidation**: Microsoft moving from AutoGen → Agent Framework; OpenAI from Swarm → Agents SDK
6. **Human-in-the-loop is table stakes**: Every serious framework now supports pause/resume workflows
7. **Voice/multimodal agents emerging**: OpenAI Agents SDK leading with realtime voice support

---

## Appendix: Links & Resources

| Framework | Docs | GitHub |
|-----------|------|--------|
| LangGraph | [docs.langchain.com](https://docs.langchain.com/oss/python/langgraph/overview) | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) |
| CrewAI | [docs.crewai.com](https://docs.crewai.com) | [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) |
| AutoGen | [microsoft.github.io/autogen](https://microsoft.github.io/autogen/) | [microsoft/autogen](https://github.com/microsoft/autogen) |
| OpenAI Agents SDK | [openai.github.io/openai-agents-python](https://openai.github.io/openai-agents-python/) | [openai/openai-agents-python](https://github.com/openai/openai-agents-python) |
| Anthropic Claude | [docs.anthropic.com](https://docs.anthropic.com) | N/A (API) |
| Mastra | [mastra.ai/docs](https://mastra.ai/docs) | [mastra-ai/mastra](https://github.com/mastra-ai/mastra) |
| Vercel AI SDK | [ai-sdk.dev](https://ai-sdk.dev/docs/introduction) | [vercel/ai](https://github.com/vercel/ai) |
| Semantic Kernel | [learn.microsoft.com](https://learn.microsoft.com/en-us/semantic-kernel/) | [microsoft/semantic-kernel](https://github.com/microsoft/semantic-kernel) |
| DSPy | [dspy.ai](https://dspy.ai/) | [stanfordnlp/dspy](https://github.com/stanfordnlp/dspy) |
