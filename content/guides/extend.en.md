---
title: "Extension Guide"
description: "How to extend ARES with custom LLM providers, tools, knowledge stores, and strategy sources."
weight: 2
---

ARES is designed for extension at every layer. Each section below references
the exact SDK option or interface to implement.

## Add an LLM provider

The SDK ships with built-in support for OpenAI, Ollama, Anthropic, and
OpenRouter. To use a custom provider:

1. Implement the `llm.Client` interface (`Generate`, `Chat`,
   `GenerateEmbedding`, `GenerateStream`).
2. Construct a `*core.LLMConfig` with your provider name and base URL.
3. Pass it via `sdk.WithLLMConfig(cfg)` or add failover with
   `sdk.WithFallbackLLM(cfg)`.

## Add a custom tool

1. Implement the `tools.Tool` interface (`Name`, `Description`, `Execute`,
   `Parameters`, `Capabilities`), or wrap a function with `tools.ToolFunc`.
2. Register it: `runtime.RegisterTool(myTool)`, or pass it as an
   `sdk.WithTool` agent option.
3. For idempotent (retry-safe) tools in the sub-agent, type-assert the
   `ToolBinder` to `*toolBinder` and call `BindIdempotentTool`.

## Add a knowledge store backend

1. Implement the `knowledge.KnowledgeStore` interface (13 methods including
   `Store`, `Query`, `HybridSearch`, `FindDuplicate`).
2. Wire it via `sdk.WithKnowledgeStore(myStore)` or register it as a
   `provider.GraphProvider` with `sdk.WithKnowledgeProvider`.

## Add a strategy source

1. Implement `agents.StrategySource` returning the live `ActiveStrategy`
   (prompt + LLM params).
2. Inject it via `leader.WithStrategySource(src)` or
   `sub.WithStrategySource(src)` when constructing agents.

## Add an MCP server connection

1. Start an MCP server exposing tools over stdio.
2. Connect with `sdk.WithMCP(MCPConn{Command: "...", Args: []string{...}})`.
   Discovered tools are auto-registered into the tool registry.

## Tune the knowledge quality gate

Use `sdk.WithAKGQualityGate(knowledge.QualityGateConfig{...})` to adjust the
extraction / consistency / freshness / usage weights and score thresholds that
gate which candidates become active KnowledgeObjects.
