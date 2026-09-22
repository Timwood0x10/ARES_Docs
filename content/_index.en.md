---
title: "ARES"
description: "A Go-native, multi-agent runtime with an adaptive knowledge graph"
---

ARES is a Go-native multi-agent runtime. It provides a unified SDK for building
LLM-powered agents with built-in memory, tool calling, an adaptive knowledge
graph (AKG), strategy evolution, and MCP integration.

## Key features

- **Unified Runtime** — one entry point (`sdk.NewRuntime`) wires the LLM client,
  tool registry, memory, knowledge fabric, and evolution system.
- **Adaptive Knowledge Graph (AKG)** — distills conversations into fact-level
  KnowledgeObjects with quality gating and hybrid retrieval (vector + lexical),
  no LLM required in the build/retrieve loop.
- **Multi-provider LLM** — OpenAI, Ollama, Anthropic, OpenRouter, plus automatic
  failover across fallback providers.
- **Tool calling** — built-in tools, custom tools, MCP-discovered tools, and a
  capability planner for intent-based tool resolution.
- **Agent OS kernel** — agents are scheduled processes, not workflow nodes.
  Work is durable `Task` intent (checkpoint + lease + epoch fencing); the
  kernel scheduler drives `Schedule → Acquire → RunQuantum → finalize`, so an
  agent dying does not kill the task (lease expiry requeues it).
- **Dynamic graphs** — the live topology is a `MutableDAG`
  (`internal/fabric/task/workflow/engine`): session L2 graphs grow per
  quantum (plan/tool/answer nodes) and compile incrementally into the task
  fabric via `planprojection`; evolution structure patches mutate DAG objects
  in place. See the workflow module page.
- **Strategy evolution** — GA over instruction/LLM-parameter strategies:
  population scoring, lifecycle gates (shadow/eval/rollback) and a
  coordinator applying patches from multiple sources. Verified gate
  semantics documented on the ares_evolution module page.
- **Bilingual documentation** — every module page is available in English and
  Chinese. Pages are audited against the source tree; where a page lags,
  the source is authoritative.

## Explore

Start with the [Architecture overview](../architecture/), then browse the
[Module reference](../modules/) for per-package details, or read the
[Guides](../guides/) for deployment and extension walkthroughs.
