---
title: "ares_observability"
description: "ARES agent 与进化操作的追踪、指标、结构化日志与成本追踪。"
weight: 180
maturity: "Beta"
---

# ares_observability

## 职责

`ares_observability` 提供 ARES 各子系统记录可观测性所用的原语:用于 LLM 调用、
工具调用、agent 步骤与错误的 `Tracer` 接口;发射 span 与指标的 OpenTelemetry 实现;
在 `/metrics` 暴露的 Prometheus 指标集合;用于开发的 `LogTracer`;只做 trace ID
传播的 `NoopTracer`;以及按模型累计 LLM 成本(美元)的 `CostTracker`。

该包将三大信号支柱(trace、metric、log)统一到一个接口之后,使调用方能在不改动
埋点位置的情况下替换实现。它同时记录运行时遥测(LLM/工具调用计数与时长、agent
步骤时长、活跃 agent gauge)与进化专用遥测(deploy 总数、guardrail 触发、shadow
评估结果、按策略 ID 的分数 gauge)。

## 架构图

```mermaid
flowchart TD
    Caller[Agent / Evolution / MCP] --> T[Tracer interface]
    T --> OTel[OTelTracer<br/>spans + Metrics]
    T --> Log[LogTracer<br/>slog]
    T --> Noop[NoopTracer<br/>trace ID only]
    OTel --> Meter[metric.Meter]
    Meter --> M[Metrics counters/histograms]
    OTel --> SpanExp[SpanExporter]
    Prom[PrometheusMetrics] --> Reg[default registry]
    Reg --> HTTP[/metrics endpoint]
    Cost[CostTracker] --> Pricing[PricingConfig]
    Pricing --> Entries[CostEntry list]
    Entries --> Report[Markdown Report]
```

`Tracer` 是中心抽象。`OTelTracer` 构造一对 tracer 与 meter provider,构建
`Metrics` 仪器集合,并把每次调用记录为 span 加 metric 属性。`PrometheusMetrics`
是并行的、基于注册表的集合,注册到默认 Prometheus 注册表,通过
`MetricsHTTPHandler` 暴露。`CostTracker` 与两者独立,依据按模型名索引的
`PricingConfig` 计算成本。

## 外部接口

```go
// Tracer defines the interface for observability tracking.
type Tracer interface {
    RecordLLMCall(ctx context.Context, call *LLMCall)
    RecordToolCall(ctx context.Context, call *ToolCall)
    RecordAgentStep(ctx context.Context, step *AgentStep)
    RecordError(ctx context.Context, err *AgentError)
    GetTraceID(ctx context.Context) string
    WithTrace(ctx context.Context) context.Context
}

func NewOTelTracer(serviceName string, opts ...OTelOption) (*OTelTracer, error)
func NewLogTracer(cfg *LogTracerConfig) Tracer
func NewNoopTracer() Tracer
func NewMetrics(meter metric.Meter) (*Metrics, error)
func NewPrometheusMetrics() (*PrometheusMetrics, error)
func NewCostTracker(pricing PricingConfig) *CostTracker
func DefaultPricingConfig() PricingConfig
func MetricsHTTPHandler() http.Handler
func RegisterMetricsRouter(mux *http.ServeMux)

// OTelOption functional options.
func WithExporter(exp sdktrace.SpanExporter) OTelOption
func WithSampler(sampler sdktrace.Sampler) OTelOption
func WithMetricReader(reader sdkmetric.Reader) OTelOption
```

## 关键类型与方法

| 类型 / 方法 | 用途 |
|---|---|
| `Tracer` | 统一的 trace/metric/log 记录接口。 |
| `OTelTracer` | OpenTelemetry SDK 实现;发射 span 与 metric。 |
| `LogTracer` | 基于 `slog` 的 tracer,用于开发。 |
| `NoopTracer` | 仅在 context 中传播 trace ID 的最小 tracer。 |
| `Metrics` | LLM、工具、agent、错误事件的 OTel 计数器/直方图。 |
| `PrometheusMetrics` | Prometheus 计数器、直方图、gauge、summary。 |
| `CostTracker` | 按调用累计 LLM 成本(美元),输出 Markdown 报告。 |
| `LLMCall` / `ToolCall` / `AgentStep` / `AgentError` | 传给 `Tracer` 的事件载荷。 |
| `PricingConfig` / `ModelPricing` | 按模型的每 1K token 输入/输出定价。 |
| `NewOTelTracer(name, opts...)` | 构建带 provider 的完整 OTel tracer。 |
| `NewPrometheusMetrics()` | 将 Prometheus 集合注册到默认注册表。 |
| `RecordLLMCall` / `RecordToolCall` | OTel 与 Prometheus 两条路径上的计数器+直方图记录器。 |
| `CostTracker.RecordCall` / `TotalCost` / `Report` | 成本累计与报告。 |
| `OTelTracer.Shutdown(ctx)` | 刷出 span/metric 并关闭 provider。 |

## 模块协作

- 依赖 `go.opentelemetry.io/otel`(trace、metric、sdk、stdouttrace)与
  `github.com/prometheus/client_golang`。
- 使用 `internal/logger` 作为包级结构化日志器。
- 被 agent 运行时、MCP 管理器与进化系统消费,用于记录运行时遥测。
- `PrometheusMetrics` 旨在通过 `RegisterMetricsRouter` 挂载到仪表盘或服务器 mux。

## 扩展方式

1. 实现 `Tracer` 接口以接入自定义后端(如 Jaeger、Datadog);在任何消费
   `Tracer` 的地方传入即可。
2. 用 `WithExporter`、`WithSampler`、`WithMetricReader` 配置 `OTelTracer`,
   指向真实 collector 而非 stdout。
3. 扩展 `PrometheusMetrics`:新增一个 `*prometheus.CounterVec` 字段,在
   `NewPrometheusMetrics` 中注册,并暴露一个 `Record...` 辅助方法。
4. 扩展 `CostTracker` 定价:向传入 `NewCostTracker` 的 `PricingConfig.Models`
   map 添加条目,或调用 `DefaultPricingConfig` 后修改。
5. 新增 metric 仪器:在 `NewMetrics(meter)` 中新增一个计数器/直方图,并加一个
   对应的 `Record...` 方法。

## 双语状态

英文源为权威参考。中文页面保持相同的结构、签名与技术内容;两份页面中所有代码
标识符、类型名与指标名均保持英文。

## 成熟度

`ares_observability` 由 `tracer_test.go`、`otel_tracer_test.go`、
`prometheus_test.go`、`cost_test.go` 与 `cost_dashboard_test.go` 覆盖。公共 API
功能完备且经过测试,但仍在演进(tracer 选项、metric 形状),故标记为 Beta。

{{< maturity "Beta" >}}
