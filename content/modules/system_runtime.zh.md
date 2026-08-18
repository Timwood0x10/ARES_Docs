---
title: "system_runtime"
description: "系统级控制面：组件注册表、依赖感知拓扑排序、生命周期编排与状态快照。"
weight: 107
maturity: "Production"
---

`internal/system_runtime` 包（包名 `system_runtime`）是统一组件装配、依赖
解析、生命周期编排与停机协调的 **System Runtime** 控制面，贯穿所有入口
（`serve`、`start`、SDK）。

System Runtime 不同于 `ares_runtime.Manager`——后者仍是 **Agent 生命周期**
子系统。System Runtime 持有更广的组件图：`EventStore`、`Memory`、`MCP`、
`Flight`、`Evolution`、`Tools`、`HTTP` 等。它与 `taskfabric`（Scheduler）、
`agentfabric`（Lifecycle）、`agentipc`（IPC）共同构成 **ARES Kernel**：
三大支柱加上负责按依赖顺序启动/停机的控制面。

## 职责

- 提供 `Component` 接口及其生命周期伴生接口
  （`Binder`、`Starter`、`ReadinessChecker`、`Stopper`、`Waiter`）：
  身份（`Name`）、依赖元数据（`Dependencies`）、依赖注入
  （`Bind(ctx, deps)`）、活动生命周期（`Start` / `Stop`）、就绪
  （`Ready`）、后台排空（`Wait`）。
- 提供 `Mode` 分类（`ModeRequired` / `ModeOptional` / `ModeDegraded`），
  声明组件失败如何影响整体系统。
- 持有组件 `Registry`：name → `entry{component, mode, status}`，按注册顺序
  迭代，依赖感知 `Resolver`（`Get(name)`），以及状态数据面
  （`GetStatus`、`SetStatus`、`AllStatuses`）。
- 通过 Kahn 算法计算依赖感知的 `TopologicalOrder`；对未注册依赖 fail loud
  （拼写错误或缺失注册不得被静默隐藏），对检测到的环 fail loud（错误信息
  中报告环成员）。
- 通过 `Orchestrator` 编排生命周期状态机：上行 `Constructed → Bound →
  Started → Ready`（逆拓扑 `Start`，依赖先启动）；下行 `Ready → Stopping →
  Stopped`（拓扑 `Stop`，被依赖者先停）；启动中途失败时回滚已启动组件。
- 提供状态快照 API：`Snapshot`（所有组件状态的即时视图，含聚合
  `SnapshotSummary`）、`IsReady`（所有 `ModeRequired` 组件 `Ready`/`Degraded`
  且无 `Failed` 时为真）、`Snapshot.JSON` 用于诊断/监控输出。

## 架构图

```mermaid
flowchart TD
    APP["Application<br/>serve / start / SDK"] --> RG["Registry.Register(c, mode)"]
    RG --> EN["entry{component, mode, status=Constructed}"]
    EN --> ORD["Registry.TopologicalOrder<br/>Kahn's algorithm"]
    ORD -- unregistered dep --> L1["fail loud: typo / missing registration"]
    ORD -- cycle --> L2["fail loud: cycle members reported"]
    ORD --> ORC["NewOrchestrator(reg, rootCtx)"]
    ORC --> UP["Start(ctx) — reverse-topological"]
    UP --> BND["Bind(ctx, deps) per component<br/>Constructed → Bound"]
    BND --> STR["Start(ctx) per component<br/>Bound → Started"]
    STR --> RDY["Ready(ctx) per component<br/>Started → Ready"]
    RDY -- error / degraded --> FL["Failed / Degraded"]
    BND -- error --> RB["rollback already-started components"]
    STR -- error --> RB
    UP --> SHD["Shutdown(ctx) — topological"]
    SHD --> STP["Stop(ctx) per component<br/>Ready → Stopping → Stopped"]
    STP --> WTR["Wait() per component<br/>drain background work"]
    SNAP["Registry.Snapshot()"] --> S["Snapshot{TakenAt, Components, Summary}"]
    S --> J["Snapshot.JSON()"]
    ISR["Registry.IsReady()"] --> CHK{"all Required Ready/Degraded<br/>and none Failed?"}
```

## 生命周期状态机

```mermaid
stateDiagram-v2
    [*] --> Declared: Register (declared type)
    [*] --> Constructed: Register (constructed instance)
    [*] --> Disabled: config gate false
    Constructed --> Bound: Bind() ok
    Constructed --> Failed: Bind() error
    Bound --> Started: Start() ok
    Bound --> Failed: Start() error
    Started --> Ready: Ready() ok
    Started --> Degraded: Ready() ok (reduced capacity)
    Started --> Failed: Ready() error
    Ready --> Stopping: Stop() called
    Degraded --> Stopping: Stop() called
    Stopping --> Stopped: Stop() ok + Wait() ok
    note right of Stopped
        Stopped is terminal;
        idempotent Stop is safe.
    end note
    note right of Failed
        Failed is terminal;
        rollback already-started
        components on mid-boot failure.
    end note
```

## 外部接口

```go
package system_runtime

// --- Component contracts ---

type Component interface {
    Name() string
    Dependencies() []string  // names of components that must be Ready first
}

type Binder interface {
    Bind(ctx context.Context, deps Resolver) error  // inject live references
}
type Starter interface {
    Start(ctx context.Context) error  // goroutines, connections, tickers
}
type ReadinessChecker interface {
    Ready(ctx context.Context) error  // verifies fully operational
}
type Stopper interface {
    Stop(ctx context.Context) error  // idempotent; reverse-topological order
}
type Waiter interface {
    Wait() error  // blocks until background work owned by the component completes
}

type Resolver interface {
    Get(name string) any  // returns the component instance, or nil if not found
}

// --- Mode ---

type Mode int
const (
    ModeRequired Mode = iota  // must reach Ready for system Ready
    ModeOptional              // not constructed when disabled; Required when enabled
    ModeDegraded              // may operate in reduced capacity; must report degraded state
)
func (m Mode) String() string

// --- Registry ---

type Registry struct {
    // mu sync.RWMutex
    // entries map[string]*entry
    // order []string  — registration order for deterministic iteration
}
func NewRegistry() *Registry
func (r *Registry) Register(c Component, mode Mode) error  // rejects nil / typed-nil / empty name / duplicate
func (r *Registry) Get(name string) any                    // implements Resolver
func (r *Registry) GetComponent(name string) Component
func (r *Registry) GetMode(name string) (Mode, bool)
func (r *Registry) GetStatus(name string) (ComponentStatus, bool)
func (r *Registry) SetStatus(name string, status ComponentStatus)
func (r *Registry) AllStatuses() []ComponentStatus
func (r *Registry) Names() []string
func (r *Registry) TopologicalOrder() ([]string, error)  // Kahn's algorithm; fail loud on cycle/unregistered dep
func (r *Registry) IsReady() bool                          // all Required Ready/Degraded and none Failed
func (r *Registry) Snapshot() Snapshot

// --- Orchestrator ---

type Orchestrator struct {
    // reg *Registry
    // rootCtx context.Context
    // mu sync.Mutex
    // started []string
    // statuses map[string]ComponentStatus
}
func NewOrchestrator(reg *Registry, rootCtx context.Context) *Orchestrator
func (o *Orchestrator) Start(ctx context.Context) error          // reverse-topological Bind → Start → Ready
func (o *Orchestrator) Shutdown(ctx context.Context) error       // topological Stop → Wait
func (o *Orchestrator) Go(fn func() error)                       // managed goroutine with error capture
func (o *Orchestrator) RootContext() context.Context
func (o *Orchestrator) Cancel()

// --- State machine ---

type State int
const (
    StateDeclared State = iota
    StateConstructed
    StateBound
    StateStarted
    StateReady
    StateDegraded
    StateFailed
    StateStopping
    StateStopped
    StateDisabled
)
func (s State) String() string
func (s State) IsTerminal() bool   // Stopped | Disabled | Failed
func (s State) IsHealthy() bool    // Ready | Degraded

type ComponentStatus struct {
    Name       string    `json:"name"`
    Mode       Mode      `json:"mode"`
    State      State     `json:"state"`
    Reason     string    `json:"reason,omitempty"`
    StartedAt  time.Time `json:"started_at,omitempty"`
    InstanceID string    `json:"instance_id,omitempty"`
}
func (s ComponentStatus) String() string

// --- Snapshot API ---

type Snapshot struct {
    TakenAt    time.Time         `json:"taken_at"`
    Components []ComponentStatus `json:"components"`
    Summary    SnapshotSummary   `json:"summary"`
}
type SnapshotSummary struct {
    Total    int `json:"total"`
    Ready    int `json:"ready"`
    Degraded int `json:"degraded"`
    Failed   int `json:"failed"`
    Disabled int `json:"disabled"`
    Stopped  int `json:"stopped"`
}
func (s Snapshot) JSON() ([]byte, error)
```

## 关键类型与方法

| 类型 / 方法 | 用途 |
| --- | --- |
| `Component` / `Binder` / `Starter` / `ReadinessChecker` / `Stopper` / `Waiter` | 受管组件可实现的生命周期伴生接口。 |
| `Mode` | 声明组件失败如何影响整体系统（`ModeRequired` / `ModeOptional` / `ModeDegraded`）。 |
| `Registry` | 组件注册表；name → `entry{component, mode, status}`；按注册顺序迭代。 |
| `Registry.Register` | 按指定 mode 声明组件；拒绝 nil/typed-nil/空名/重复。 |
| `Registry.TopologicalOrder` | Kahn 算法；依赖先于被依赖者；对环或未注册依赖 fail loud。 |
| `Registry.Snapshot` / `Registry.IsReady` | 用于诊断与监控的状态快照 API。 |
| `Orchestrator` | 生命周期编排器；逆拓扑 Start、拓扑 Stop、启动中途失败回滚。 |
| `Orchestrator.Start` | 每组件逆拓扑执行 `Constructed → Bound → Started → Ready`。 |
| `Orchestrator.Shutdown` | 每组件拓扑执行 `Ready → Stopping → Stopped`，随后 `Wait` 排空后台。 |
| `Orchestrator.Go` | 带错误捕获的受管 goroutine，用于组件自有后台工作。 |
| `State` / `ComponentStatus` | 生命周期状态机与每组件状态数据面。 |
| `Snapshot` / `SnapshotSummary` | 所有组件状态的即时视图，含聚合计数。 |

## 模块协作

- `system_runtime` -> `internal/ares_bootstrap`：Bootstrap 是唯一装配根，
  将所有组件注册到 System Runtime registry，并通过 Orchestrator 启停。
- `system_runtime` -> `internal/taskfabric` / `internal/agentfabric` /
  `internal/agentipc`：三大 Kernel 支柱作为组件注册，按依赖顺序启停。
- `system_runtime` -> `internal/ares_events` / `internal/ares_memory` /
  `internal/ares_mcp` / `internal/ares_flight` / `internal/ares_evolution`：
  每个都是受管组件，通过声明 `Dependencies()` 参与拓扑排序。
- `system_runtime` -> `cmd/ares`：serve、start（monitor-live）、SDK 均经
  `ares_bootstrap.Bootstrap` 装配；System Runtime 提供 `ares status` 暴露
  的 `Snapshot()` API。

## 扩展方式

1. **将子系统变为受管组件**：实现 `Component`（`Name`、`Dependencies`）及
   相关伴生接口（`Binder`、`Starter`、`ReadinessChecker`、`Stopper`、
   `Waiter`），通过 `Registry.Register(c, mode)` 按合适 `Mode` 注册。
2. **声明依赖以参与拓扑排序**：通过 `Dependencies()`；编排器让依赖先启动
  （逆拓扑）、被依赖者先停机（拓扑）。未注册依赖在 `TopologicalOrder` 时
   fail loud。
3. **报告降级运行**：实现 `ReadinessChecker`，在缺失写依赖时返回非 nil
   error；编排器将组件转至 `StateDegraded` 并记录原因，`IsReady` 仍返回真。
4. **停机时排空后台**：实现 `Waiter`；编排器在 `Stop()` 之后调用 `Wait()`，
   确保 goroutine 与 ticker 完全退出后再终止进程。
5. **暴露诊断状态**：调用 `Registry.Snapshot()` 并经 `Snapshot.JSON()`
   序列化；`IsSystemReady` 将每组件 `State` 聚合为单一就绪判定。
6. **验证双入口等价**：断言经 `ares_bootstrap.Bootstrap` 构建的组件图在
   `serve`、`start`、SDK 三个入口下等价——锁定入口统一的契约测试。

## 双语状态

本页为英文参考的结构镜像中文版。所有代码标识符、类型名与签名在两份文件中均
保持英文，仅叙述性文字不同。英文版发布为 `system_runtime.en.md`。

## 成熟度

Production。该包由 `orchestrator_test.go`、`registry_test.go` 及闭包契约
测试 `closure_entry_equivalence_test.go` 覆盖。它实现了带依赖感知拓扑
排序的组件生命周期状态机，通过 `ares_bootstrap` 集成所有 ARES 子系统，
且不含任何实验性标记。

{{< maturity "Production" >}}
