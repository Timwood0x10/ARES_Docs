---
title: "ARES"
description: "Go 原生多智能体运行时，含自适应知识图谱"
---

ARES 是一个 Go 原生的多智能体运行时。它提供统一 SDK，用于构建具备内置记忆、
工具调用、自适应知识图谱（AKG）、策略进化与 MCP 集成的 LLM 智能体。

## 核心特性

- **统一运行时** — 单一入口（`sdk.NewRuntime`）串联 LLM 客户端、工具注册表、
  记忆、知识图谱与进化系统。
- **自适应知识图谱（AKG）** — 将对话蒸馏为事实级 KnowledgeObject，带质量门控
  与混合检索（向量 + 词法），构建/检索路径中无需 LLM。
- **多供应商 LLM** — OpenAI、Ollama、Anthropic、OpenRouter，并支持跨后备供应商
  的自动故障转移。
- **工具调用** — 内置工具、自定义工具、MCP 发现的工具，以及基于意图的能力
  planner 解析。
- **Agent OS 内核** — agent 是被调度的进程，而非工作流节点。工作是持久化的
  `Task` 意图（checkpoint + lease + epoch fencing）；内核调度器驱动
  `Schedule → Acquire → RunQuantum → finalize`，agent 死亡不等于任务死亡
  （lease 过期后任务重新入队）。
- **动态图** — live 拓扑是 `MutableDAG`（`internal/fabric/task/workflow/engine`）：
  session L2 图按量子生长（plan/tool/answer 节点），经 `planprojection`
  增量编译进 task fabric；进化结构补丁原地变更 DAG 对象。详见 workflow 模块页。
- **策略进化** — 基于遗传算法优化智能体指令与 LLM 参数，协调器从多来源应用
  补丁；闸门语义核实记录见 ares_evolution 模块页。
- **双语文档** — 每个模块页面均提供中英文版本；页面经源码树审计，若有滞后以源码为准。

## 探索

从[架构概览](../architecture/)开始，然后浏览[模块参考](../modules/)了解各包细节，
或阅读[指南](../guides/)获取部署与扩展指引。
