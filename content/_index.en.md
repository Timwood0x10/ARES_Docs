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
- **Strategy evolution** — GA-based optimization of agent instructions and LLM
  parameters, with a coordinator that applies patches from multiple sources.
- **Bilingual documentation** — every module page is available in English and
  Chinese, kept in sync against the real source code.

## Explore

Start with the [Architecture overview](../architecture/), then browse the
[Module reference](../modules/) for per-package details, or read the
[Guides](../guides/) for deployment and extension walkthroughs.
