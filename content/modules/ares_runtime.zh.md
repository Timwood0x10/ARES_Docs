---
title: "ares_runtime"
description: "智能体生命周期管理器：注册、监督式启停、崩溃复活、快照与工作流插件。"
weight: 120
maturity: "Production"
---

`internal/ares_runtime` 包（包名 `ares_runtime`）是智能体的进程级监督者。
智能体被视为可丢弃的执行器，runtime 负责其诞生、死亡与复活。`Manager` 实现
`Runtime` 接口，在受管 goroutine 中运行每个智能体，提供 panic 恢复、周期性健康
检查，以及由 `AgentFactory` 驱动的指数退避复活。它还暴露了插件总线、检查点存储
与混沌工程 arena。

## 职责

- 将智能体连同 `AgentFactory` 一起注册，以便死亡后可重建。
- 在受管 errgroup goroutine 中启动每个智能体，提供 `panic` 恢复，并将失败汇入
  `NotifyAgentDead`。
- 运行后台健康检查循环，使用 `base.Heartbeater.IsAlive()`（退化为 `Status()`）
  探测死亡智能体并触发复活。
- 通过 `RestoreAgent` 复活死亡智能体：从工厂创建、回放 `EventStore` 事件、恢复
  快照状态、补充认知记忆状态后重新启动；指数退避（1s 到 30s，最多 5 次）与每智能体
  重启上限共同约束重试。
- 通过 `SnapshotStore` 对有状态智能体（`base.StatefulAgent`）做快照与恢复，并在
  关闭时捕获最终快照。
- 通过 `CheckpointPlugin` 持久化执行检查点（`ExperienceCheckpoint`），用于工作流
  崩溃恢复。
- 提供插件契约（`RuntimePlugin`、`WorkflowHook`、`MemoryPlugin`、
  `EvolutionPlugin`、`RecoveryPlugin`）与 `EventBus` 以供扩展。
- 暴露混沌工程故障注入（`PauseAgent`、`SlowAgent`、`ToolTimeout`、
  `PartitionNetwork` 等）供 arena 使用。

## 架构图

```mermaid
flowchart TD
    APP["Application"] --> RG["Manager.RegisterAgent<br/>agent + AgentFactory"]
    RG --> M["Manager.agents map"]
    ST["Manager.Start"] --> L["launchAgentGoroutine<br/>panic recover"]
    L --> AG["agent.Start(ctx)"]
    HC["healthCheck ticker"] --> AL{"IsAlive / Status?"}
    AL -- dead --> NAD["NotifyAgentDead"]
    AG -- panic / start fail --> NAD
    NAD --> SR["scheduleResurrection<br/>backoff 1s->30s, 5 tries"]
    SR --> RA["RestoreAgent"]
    RA --> RC["recoverAgentState"]
    RC --> RPL["replayEvents<br/>EventStore.Read"]
    RC --> SNAP["RecoverSnapshotOrEvents<br/>SnapshotStore"]
    RC --> COG["buildCognitiveState<br/>MemoryManager"]
    RC --> RS["StatefulAgent.RestoreState<br/>+ ReplayEvents"]
    RA --> L
    STOP["Manager.Stop"] --> FS["final Snapshot save"]
    STOP --> CST["cancel + agent.Stop"]
    CP["CheckpointPlugin"] --> CK["CheckpointStore.Save<br/>ExperienceCheckpoint"]
    PLG["RuntimePlugin / WorkflowHook"] --> BUS["EventBus"]
```

## 外部接口

```go
// Runtime is the supervisor interface implemented by Manager.
type Runtime interface {
    StartAgent(ctx context.Context, agent base.Agent) error
    StopAgent(ctx context.Context, agentID string) error
    RestartAgent(ctx context.Context, agentID string) error
    RestoreAgent(ctx context.Context, agentID string, factory AgentFactory) error
    NotifyAgentDead(agentID string, reason string)
    RegisterAgent(agent base.Agent, factory AgentFactory)
    Start(ctx context.Context) error
    Stop() error
    Stats() RuntimeStats
}

type AgentFactory func() base.Agent

type Config struct {
    HealthCheckInterval time.Duration
    MaxRestartsPerAgent int  // 0 = unlimited
    MaxReplayEvents     int
    AgentStopTimeout    time.Duration
    OverallStopTimeout  time.Duration
    RestoreTimeout      time.Duration
}
func DefaultConfig() *Config

type RuntimeStats struct {
    ActiveAgents    int
    TotalRestarts   int
    Uptime          time.Duration
    BackgroundTasks map[string]int64
}

func New(config *Config, eventStore ares_events.EventStore, memManager memory.MemoryManager) *Manager

type Manager struct {
    // owns agents, factories, eventStore, memManager, snapshotStore,
    // errgroup g/gctx, config, chaosConfig, dagStore
}
func (m *Manager) WithSnapshotStore(store base.SnapshotStore) *Manager
func (m *Manager) RegisterAgent(agent base.Agent, factory AgentFactory)
func (m *Manager) RegisterAgentDAG(agentID string, dag any)
func (m *Manager) GetAgentDAG(agentID string) (any, bool)
func (m *Manager) StartAgent(ctx context.Context, agent base.Agent) error
func (m *Manager) StopAgent(ctx context.Context, agentID string) error
func (m *Manager) GetAgent(agentID string) base.Agent
func (m *Manager) RestartAgent(ctx context.Context, agentID string) error
func (m *Manager) RestoreAgent(ctx context.Context, agentID string, factory AgentFactory) error
func (m *Manager) NotifyAgentDead(agentID string, reason string)
func (m *Manager) Start(ctx context.Context) error
func (m *Manager) Stop() error
func (m *Manager) Stats() RuntimeStats

// Introspection + chaos (manager_chaos.go)
type AgentInfo struct {
    ID       string
    Type     string
    Status   string
    Restarts int
    Paused   bool
}
func (m *Manager) ListAgents() []AgentInfo
func (m *Manager) GetAgentInfo(agentID string) (*AgentInfo, bool)
func (m *Manager) PauseAgent(ctx context.Context, agentID string) error
func (m *Manager) ResumeAgent(ctx context.Context, agentID string) error
func (m *Manager) SlowAgent(ctx context.Context, agentID string, delay time.Duration) error
func (m *Manager) PartitionNetwork(ctx context.Context, agentID string) error
func (m *Manager) ToolTimeout(ctx context.Context, agentID string, timeout time.Duration) error
func (m *Manager) CorruptMemory(ctx context.Context, agentID string) error
func (m *Manager) DisconnectMCP(ctx context.Context, agentID string) error
func (m *Manager) InjectLLMFailure(ctx context.Context, agentID string, errType string) error

// Snapshot / restore helpers
func RecoverSnapshotOrEvents(ctx context.Context, store base.SnapshotStore, agentID string, eventFn func() map[string]any) map[string]any

// Checkpoints
type CheckpointStore interface {
    Save(ctx context.Context, key string, data []byte) error
    Load(ctx context.Context, key string) ([]byte, error)
}
type ExperienceCheckpoint struct {
    SchemaVersion    int
    ExecutionID      string
    WorkflowID       string
    StateVersion     int64
    Status           string
    CurrentRound     int
    StepStates       []StepStateSnapshot
    Variables        map[string]interface{}
    OutputStore      map[string]string
    DAGNodes         []string
    DAGEdges         []DAGEdge
    RouteHistory     []RouteEntry
    ToolHistory      []ToolEntry
    MemoryHits       []MemoryEntry
    InterruptHistory []InterruptEntry
    LoopHistory      []LoopEntry
    ErrorHistory     []ErrorEntry
    ScoringSignals   []ScoringSignal
    CreatedAt        time.Time
}
func CheckpointKey(executionID string) string
func NewCheckpointPlugin(name string, store CheckpointStore) *CheckpointPlugin
func (p *CheckpointPlugin) WithFlushInterval(n int) *CheckpointPlugin
func (p *CheckpointPlugin) WithCollector(c *ExecutionCollector) *CheckpointPlugin
func (p *CheckpointPlugin) BeforeStep(ctx context.Context, executionID string, step *Step) error
func (p *CheckpointPlugin) AfterStep(ctx context.Context, executionID string, result *StepResult) error
func (p *CheckpointPlugin) Snapshot(executionID string) *ExperienceCheckpoint
func (p *CheckpointPlugin) Flush(ctx context.Context, executionID string) error
func (p *CheckpointPlugin) Cleanup(executionID string)

// Plugins
type Capability string
const (
    CapObserver   Capability = "observer"
    CapCheckpoint Capability = "checkpoint"
    CapRouter     Capability = "router"
    CapLoop       Capability = "loop"
    CapMemory     Capability = "memory"
    CapEvolution  Capability = "evolution"
    CapTool       Capability = "tool"
    CapRecovery   Capability = "recovery"
)
type RuntimePlugin interface {
    Name() string
    Capabilities() []Capability
    Start(ctx context.Context, bus EventBus) error
    Stop(ctx context.Context) error
}
type WorkflowHook interface {
    BeforeStep(ctx context.Context, executionID string, step *Step) error
    AfterStep(ctx context.Context, executionID string, result *StepResult) error
}
type MemoryPlugin interface {
    RuntimePlugin
    AdviseRoute(ctx context.Context, state RouteState) ([]RouteAdvice, error)
}
type EvolutionPlugin interface {
    RuntimePlugin
    Recommend(ctx context.Context, state ExecutionState) (*RuntimeRecommendation, error)
    RecordOutcome(ctx context.Context, outcome ExecutionOutcome) error
}
type RecoveryPlugin interface {
    RuntimePlugin
    ShouldRecover(ctx context.Context, failure StepFailure, state ExecutionState) bool
}
type EventBus interface {
    Emit(ctx context.Context, streamID string, eventType ares_events.EventType, moduleName string, payload map[string]any)
    Subscribe(ctx context.Context, filter ares_events.EventFilter) (<-chan *ares_events.Event, error)
}

// Workflow step mirror types
type StepStatus string
const (
    StepStatusPending   StepStatus = "pending"
    StepStatusRunning   StepStatus = "running"
    StepStatusCompleted StepStatus = "completed"
    StepStatusFailed    StepStatus = "failed"
    StepStatusSkipped   StepStatus = "skipped"
)
type Step struct {
    ID        string
    Name      string
    AgentType string
    Status    StepStatus
    Output    string
    Error     string
    StartedAt time.Time
}
type StepResult struct {
    StepID   string
    Name     string
    Status   StepStatus
    Output   string
    Error    string
    Duration time.Duration
    Metadata map[string]string
}

// Sentinel errors
var (
    ErrAgentNotFound        // wraps apperrors.ErrNotFound
    ErrAgentAlreadyRegistered
    ErrRuntimeStopped
    ErrNilAgent
    ErrNilFactory
)
```

## 关键类型与方法

| 类型 / 方法 | 用途 |
| --- | --- |
| `Runtime` | 智能体生命周期的监督接口。 |
| `Manager` | 具体监督者，持有 agent map、工厂、errgroup 与各类存储。 |
| `New` | 由 `Config`、`EventStore`、`MemoryManager` 构造 `Manager`。 |
| `AgentFactory` | 无参构造函数，用于重建死亡智能体。 |
| `Manager.RegisterAgent` | 注册智能体及其工厂以纳入生命周期管理。 |
| `Manager.Start` | 启动所有已注册智能体并开启健康检查循环。 |
| `Manager.Stop` | 捕获最终快照、取消 context、并发停止智能体并等待 goroutine。 |
| `Manager.StartAgent` / `StopAgent` | 单智能体启停，注入混沌 context。 |
| `Manager.RestartAgent` | 停止并从工厂重新启动智能体（自增重启计数）。 |
| `Manager.RestoreAgent` | 重建智能体、回放事件、恢复状态并重新启动。 |
| `Manager.NotifyAgentDead` | 触发带退避的异步复活，遵循 `MaxRestartsPerAgent`。 |
| `Manager.healthCheck` | 通过 `Heartbeater` 或 `Status()` 的周期性存活探测。 |
| `Manager.WithSnapshotStore` | 接入 `SnapshotStore` 以支持快照优先恢复。 |
| `RecoverSnapshotOrEvents` | 快照优先、事件兜底的状态恢复。 |
| `CheckpointPlugin` | 在步骤边界持久化 `ExperienceCheckpoint` 以支持崩溃恢复。 |
| `RuntimePlugin` / `WorkflowHook` | 插件总线的扩展契约。 |
| `EventBus` | 暴露给插件的事件扇出系统。 |
| `AgentInfo` / `ListAgents` | 面向 dashboard 的内省。 |
| `PauseAgent` / `SlowAgent` / `ToolTimeout` | arena 混沌故障注入。 |
| `Config` | 健康检查间隔、重启上限、回放上限、停止/恢复超时。 |

## 模块协作

- `ares_runtime` -> `internal/agents/base`：使用 `Agent`、`StatefulAgent`、
  `Heartbeater`、`SnapshotStore`。
- `ares_runtime` -> `internal/ares_events`：使用 `EventStore` 进行事件回放、
  完整性校验与生命周期事件发射。
- `ares_runtime` -> `internal/ares_memory`：用于认知恢复
  （`GetLatestSessionForLeader`、`GetMessages`）与 event store 接线。
- `ares_runtime` -> `internal/ares_ctxutil`：用于 detached/带标签 context 与
  后台任务统计。
- `ares_runtime` -> `internal/core/models`：使用 `AgentStatus` 常量作为基于状态的
  健康检查兜底。
- 插件（`MemoryPlugin`、`EvolutionPlugin`、`RecoveryPlugin`）消费工作流引擎产出的
  执行状态与结果，并向 runtime 反馈路由与恢复决策。

## 扩展方式

1. 实现 `base.Agent`（复活需实现 `StatefulAgent`），在 `Start` 前通过
   `Manager.RegisterAgent(agent, factory)` 注册；工厂会在每次复活时被调用。
2. 实现快照优先恢复：实现 `base.SnapshotStore` 并在 `Start` 前通过
   `Manager.WithSnapshotStore(store)` 接入。
3. 新增工作流插件：实现 `RuntimePlugin`（可选 `WorkflowHook`、`MemoryPlugin`、
   `EvolutionPlugin`、`RecoveryPlugin`），声明其 `Capability` 集合，并在 `Start`
   期间注册到 `EventBus`。
4. 持久化执行检查点：实现 `CheckpointStore`，用 `NewCheckpointPlugin(name, store)`
   构建，通过 `WithFlushInterval` 调优批量写入，并在执行完成时调用 `Flush`。
5. 在测试中通过混沌方法（`PauseAgent`、`SlowAgent`、`ToolTimeout`、
   `PartitionNetwork`、`CorruptMemory`、`DisconnectMCP`、`InjectLLMFailure`）
   注入故障，验证复活与兜底路径。
6. 通过传入 `New` 的自定义 `Config`（重启上限、健康检查间隔、回放上限、
   停止/恢复超时）调优监督行为。
7. 通过 `RegisterAgentDAG` 将工作流 DAG 与智能体关联，使进化系统可对运行中的
   DAG 打补丁；用 `GetAgentDAG` 取回。

## 双语状态

本页为中文参考。结构与技术内容完全相同的英文版本发布为 `ares_runtime.en.md`。两个文件中
所有代码标识符、类型名与签名均保持英文，仅叙述性文字不同。

## 成熟度

Production。该包由 `runtime_test.go`、`runtime_core_test.go`、`recovery_test.go`、
`arena_test.go`、`checkpoint_flush_test.go`、`router_test.go`、
`outcome_recorder_test.go` 以及进化插件测试覆盖；实现 `Runtime` 监督接口，集成到
SDK 与 agents；不含任何实验性标记。

{{< maturity "Production" >}}
