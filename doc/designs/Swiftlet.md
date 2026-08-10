# Swiftlet

> **A lightweight, high-concurrency Agent Runtime for Agentic Judging**

**Status:** Draft  
**Version:** 0.1  
**Primary Language:** Rust  
**Runtime:** Tokio  
**Primary Use Cases:** Agentic Judge / Agentic Labeling / Evaluation / Regression / Validation

---

# 1. Overview

Swiftlet 是一个面向 **Agentic Judge / Agentic Evaluation** 场景设计的极轻量 Agent Runtime。

它不试图成为 Claude Code、Codex 或通用 Agent Framework 的替代品，而是针对以下 workload 进行专门优化：

- 大批量独立任务；
- 单任务生命周期较短；
- Tool 数量极少；
- Agent 状态简单；
- 需要高并发运行；
- 需要严格控制上下文、内存和运行资源；
- 需要快速创建和销毁 Agent Run；
- 需要强隔离、可重复、可审计。

典型任务包括：

- 数据自动打标；
- 模型输出评分；
- Agent trajectory 评价；
- benchmark 自动验证；
- coding task correctness 检查；
- regression testing；
- reward generation；
- teacher model 数据生成；
- 数据过滤与质量检查。

Swiftlet 的核心设计目标不是：

> Build a smaller coding agent.

而是：

> Build the smallest practical agent execution primitive for massive concurrent evaluation workloads.

---

# 2. Naming

## Recommended Name: Swiftlet

Swiftlet 的含义与项目特征比较匹配：

| 特征 | Swiftlet |
|---|---|
| Small | 小型鸟类 |
| Lightweight | 极轻 |
| Fast | 高速飞行 |
| Concurrent | 群体活动 |
| Short-lived Agent Run | 快速进入/退出任务 |
| Technical branding | 简短、容易形成 Runtime / Worker 等后缀 |

推荐正式命名：

```text
Swiftlet
```

CLI：

```bash
swiftlet run tasks.jsonl
swiftlet judge benchmark.jsonl
swiftlet serve
```

Rust crate 可以采用：

```text
swiftlet-runtime
swiftlet-core
swiftlet-sandbox
```


# 3. Problem Definition

当前很多模型评测、数据处理和 Agent Benchmark 工作正在从传统 deterministic evaluator：

```text
input
  ↓
rule / script
  ↓
score
```

转变为：

```text
input
  ↓
LLM Agent
  ↓
inspect environment
  ↓
possibly execute tools
  ↓
reason
  ↓
structured judgment
```

即：

```text
Agentic Judge
```

例如判断一个 Coding Agent 是否真正解决了问题：

```text
Agent trajectory
      │
      ▼
Agentic Judge
      │
      ├── Read repository
      ├── Inspect changes
      ├── Bash test
      └── Analyze result
      │
      ▼
PASS / FAIL / SCORE
```

相比单次 LLM Judge，Agentic Judge 可以主动检查环境，因此可靠性更高。

但其代价是运行时复杂度显著增加。

传统 Coding Agent Runtime 通常包含：

- 大量工具；
- MCP；
- RAG；
- long-term memory；
- checkpoint；
- session persistence；
- complex context manager；
- multi-agent；
- planning；
- sub-agent；
- workflow engine。

这些能力对于 Judge workload 大部分都是：

> unnecessary runtime overhead.

Swiftlet 的目标就是删除这些能力。

---

# 4. Target Workload

Swiftlet 优先针对以下 workload：

```text
10³ – 10⁶ independent tasks
```

单任务典型生命周期：

```text
1 – 120 seconds
```

典型 Agent Loop：

```text
1 – 8 model calls
```

典型 Tool Calls：

```text
0 – 10 calls
```

典型工具：

```text
Read
Write
Bash
Skill
```

典型 Context：

```text
5K – 64K tokens
```

而不是持续数十分钟、数百轮 interaction 的 Agent Session。

---

# 5. Product Goals

Swiftlet 有六个一级目标。

## G1. Minimal Runtime

最小化：

- runtime abstraction；
- tool abstraction；
- allocations；
- process；
- dependency；
- protocol；
- Agent state。

Agent 本质应是：

```text
State Machine
+
Context
+
Model Client
+
Tools
```

---

## G2. Fast Startup

Agent Run 创建应该接近普通 async task 创建，而不是启动完整 Agent Process。

目标：

```text
AgentRun creation < 1 ms
```

Worker cold start：

```text
Target < 100 ms
Stretch < 30 ms
```

不包含 sandbox image preparation。

---

## G3. Low Memory

Agent Runtime 本身应该几乎不占资源。

目标：

```text
Idle AgentRun runtime state
< 100 KB
```

推荐目标：

```text
20 – 50 KB
```

不包含：

- workspace；
- Bash subprocess；
- LLM HTTP response buffer。

---

## G4. High Concurrency

一个 Worker Process 应支持大量 AgentRun：

```text
100 – 1000+ concurrent runs
```

具体吞吐通常受到以下因素限制：

```text
LLM API QPS
Bash CPU
Sandbox Memory
Network bandwidth
Provider rate limit
```

而不应该由 Swiftlet Runtime 自身限制。

---

## G5. Deterministic Resource Control

任何单个 case 都不能：

- OOM 整个 Worker；
- 无限运行；
- 无限 fork；
- 无限输出；
- 无限占用 context；
- 无限 retry；
- 卡住 batch。

---

## G6. Evaluation-grade Reproducibility

每个 Run 都必须能够回答：

```text
What happened?
Why did it stop?
What tools were executed?
How much resource was used?
Which model/version/prompt was used?
```

---

# 6. Non-Goals

Swiftlet V1 明确不支持：

```text
Multi-Agent
Sub-Agent
MCP
RAG
Vector Database
Long-term Memory
Checkpoint Resume
GUI
Interactive Chat
Planning Graph
Workflow DAG
Browser
Computer Use
Persistent Shell
Distributed Consensus
```

除非后续 benchmark 数据证明存在必要性，否则不得进入 Core Runtime。

---

# 7. Core Design Principle

## 7.1 AgentRun ≠ Process

Swiftlet 的核心原则：

```text
AgentRun = Tokio Task
```

而不是：

```text
AgentRun = OS Process
```

Worker：

```text
             Swiftlet Worker
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   AgentRun      AgentRun      AgentRun
    Future         Future        Future
```

只有执行不可信代码时才建立 OS-level isolation：

```text
AgentRun
   │
   └── Bash
        │
        ▼
      Sandbox
```

---

# 8. High-level Architecture

```text
                    Swiftlet
                       │
                       ▼
              ┌─────────────────┐
              │ Input Adapter   │
              │ JSONL/API/Queue │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │    Scheduler    │
              │                 │
              │ Backpressure    │
              │ Semaphores      │
              │ Priority        │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    AgentRun A     AgentRun B     AgentRun N
        │              │              │
        └──────────────┼──────────────┘
                       │
                ┌──────▼──────┐
                │ Agent Core  │
                │             │
                │ State       │
                │ Context     │
                │ StopPolicy  │
                └──────┬──────┘
                       │
             ┌─────────┼───────────┐
             ▼         ▼           ▼
         ModelClient  Tools      Telemetry
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
            Read      Write      Bash
                                  │
                                  ▼
                              Sandbox
```

---

# 9. Core Technology Stack

## Language

```text
Rust stable
```

原因：

- 无 GC；
- 可预测内存；
- 高效 async runtime；
- cheap abstraction；
- 强类型；
- 更适合长期运行 Worker；
- 更适合资源敏感 Runtime。

---

## Async Runtime

```text
Tokio
```

负责：

- AgentRun task；
- network I/O；
- timers；
- channels；
- semaphore；
- subprocess；
- cancellation。

---

## HTTP

```text
reqwest
+
rustls
```

整个 Worker 共享一个或少量长期：

```text
ModelClient
```

严禁：

```text
new HTTP client per AgentRun
```

连接必须复用。

---

## Serialization

```text
serde
serde_json
```

稳定模型协议尽量 Deserialize 到 struct。

避免 runtime 内大量：

```text
serde_json::Value
```

---

## HTTP Server

如果需要 daemon：

```text
Axum
```

但 V1 CLI 不强制依赖 server mode。

---

## Observability

```text
tracing
tracing-subscriber
```

---

# 10. Agent Core

Agent Core 必须保持简单。

建议状态：

```rust
enum RunState {
    Created,
    Preparing,
    AwaitingModel,
    ExecutingTool,
    Completed,
    Failed,
    Cancelled,
}
```

主循环：

```text
prepare
   │
   ▼
build context
   │
   ▼
model()
   │
   ├──────── final ───────► validate ─► finish
   │
   ▼
tool call
   │
   ▼
execute
   │
   ▼
append result
   │
   └──────────────────────► loop
```

禁止隐藏复杂 transition。

---

# 11. AgentRun Data Model

```rust
struct AgentRun {
    id: RunId,

    task: TaskSpec,

    state: RunState,

    context: RunContext,

    limits: RunLimits,

    stats: RunStats,
}
```

AgentRun 应尽量：

```text
stack-oriented
+
small heap footprint
+
no global mutable state
```

---

# 12. Context Management

Context Manager 是 Swiftlet 最核心模块之一。

目标不是保存完整 conversation history。

目标是：

> 在最小 token 和内存成本下，为下一次推理保留 sufficient information。

---

## 12.1 Context Structure

推荐：

```rust
struct RunContext {
    task: Arc<TaskSpec>,
    events: Vec<ContextEvent>,
    budget: ContextBudget,
}
```

Event：

```rust
enum ContextEvent {
    AssistantText(...),
    ToolCall(...),
    ToolResult(...),
}
```

---

# 13. Context Budget

每一个 Run 都必须拥有硬 Budget。

例如：

```rust
struct ContextBudget {
    max_tokens: usize,
    max_bytes: usize,

    max_tool_result_bytes: usize,
    max_single_file_bytes: usize,
}
```

推荐默认：

```text
max context:
64K tokens

tool result entering context:
16 KB

single Read:
64 KB
```

不同模型允许 override。

---

# 14. Context Projection

模型每次看到的内容不等于完整 RunContext。

真正执行：

```text
RunContext
    │
    ▼
ContextProjector
    │
    ▼
ModelContext
```

优先级：

```text
P0 Task Objective
P0 Rubric / Judge Policy
P0 Output Schema

P1 Recent Tool Results
P1 Current Evidence

P2 Recent Agent Actions

P3 Old Tool Results
P4 Old Assistant Text
```

Context 超预算时从低优先级开始移除。

---

# 15. No Automatic Summarization by Default

V1 不允许默认：

```text
Context too long
     ↓
another LLM
     ↓
summary
```

因为会增加：

- latency；
- cost；
- nondeterminism；
- failure surface。

默认策略：

```text
projection
+
truncation
+
artifact reference
```

---

# 16. Large Output Handling

大输出必须：

```text
Store externally
+
put excerpt into context
```

例如：

```rust
struct ToolArtifact {
    id: ArtifactId,
    path: PathBuf,
    size: usize,
}
```

Context：

```text
bash produced 782 KB output.

artifact:
  bash://a82f

excerpt:
  first 4 KB
  ...
  last 8 KB
```

Agent 可再次：

```text
Read artifact
```

---

# 17. Built-in Tools

V1 仅支持：

```text
Read
Write
Bash
Skill
```

---

# 18. Read Tool

Interface：

```text
read(
    path,
    offset?,
    limit?
)
```

要求：

```text
workspace only

default max:
64 KB

must support:
offset
pagination
truncation metadata
```

返回：

```json
{
  "content": "...",
  "offset": 0,
  "bytes": 65536,
  "truncated": true
}
```

---

# 19. Write Tool

Interface：

```text
write(path, content)
```

要求：

```text
workspace only
```

必须防止：

```text
../ traversal
symlink escape
absolute path escape
```

配置：

```text
max write size
max total workspace size
```

---

# 20. Bash Tool

Interface：

```rust
struct BashInput {
    command: String,
    cwd: Option<PathBuf>,
    timeout_ms: Option<u64>,
}
```

Response：

```rust
struct BashOutput {
    exit_code: Option<i32>,

    stdout: OutputRef,
    stderr: OutputRef,

    duration_ms: u64,

    timed_out: bool,
}
```

---

# 21. Bash Limits

必须存在：

```text
timeout
memory
CPU
PID
stdout
stderr
filesystem
network
```

默认建议：

```text
command timeout:
30 s

stdout:
1 MB

stderr:
1 MB

context excerpt:
16 KB
```

禁止 Bash 无限输出。

---

# 22. Persistent Shell

V1 禁止 Persistent Shell。

每次：

```text
Bash Tool Call
       │
       ▼
new isolated process
       │
       ▼
execute
       │
       ▼
kill / cleanup
```

这样避免：

- 环境变量污染；
- cwd 污染；
- zombie process；
- background task；
- shell state leak。

---

# 23. Sandbox

安全模型：

```text
Agent output is untrusted.

Task input is untrusted.

Bash command is untrusted.
```

即使 benchmark 数据来自内部，也必须采用该假设。

---

# 24. Sandbox V1

推荐 Linux：

```text
namespace
+
cgroup v2
+
seccomp
+
ephemeral workspace
```

隔离：

```text
PID
mount
user
network
```

限制：

```text
CPU
RAM
PID
disk
wall time
```

---

# 25. Higher Isolation

需要执行真正 hostile workload 时可增加：

```text
gVisor
```

更高安全级别：

```text
Firecracker
```

但都不属于 Swiftlet Core。

架构应保持：

```text
trait SandboxBackend
```

允许替换。

---

# 26. Skill

Swiftlet 中 Skill 不等于 Tool Plugin Framework。

V1：

```text
Skill = Prompt + Policy + Output Schema
```

例如：

```text
skills/
   code_judge.md
   labeler.md
   regression.md
```

定义：

```rust
struct Skill {
    name: String,

    instructions: String,

    output_schema: Option<JsonSchema>,
}
```

---

# 27. Skill Loading

Skill 在 Worker startup 时加载。

禁止每 Agent：

```text
disk scan
parse config
discover plugin
```

加载后共享：

```text
Arc<SkillRegistry>
```

---

# 28. Model Client

所有 AgentRun 共用：

```text
ModelClientPool
```

负责：

```text
connection reuse
authentication
timeout
rate limiting
retry
usage collection
```

---

# 29. Retry Policy

必须非常保守。

推荐：

```text
network error:
max 2 retries

429:
provider-aware backoff

5xx:
1–2 retries

invalid model output:
1 repair attempt

tool error:
return to Agent
```

禁止：

```text
retry forever
```

---

# 30. Structured Judge Output

Agentic Judge 强烈要求 structured result。

例如：

```json
{
  "label": "PASS",
  "score": 0.87,
  "confidence": 0.91,
  "reason": "Tests pass and implementation satisfies the requested behavior."
}
```

推荐 Rust：

```rust
struct JudgeResult {
    label: JudgeLabel,
    score: Option<f32>,
    confidence: Option<f32>,
    reason: Option<String>,
}
```

---

# 31. Stop Policy

每一个 AgentRun 都必须存在硬 StopPolicy。

```rust
struct RunLimits {
    max_steps: u16,
    max_model_calls: u16,
    max_tool_calls: u16,

    wall_timeout_ms: u64,

    model_timeout_ms: u64,
    tool_timeout_ms: u64,

    max_context_tokens: usize,
    max_output_bytes: usize,
}
```

推荐默认：

```text
max steps:
8

max model calls:
8

max tool calls:
12

wall timeout:
120 sec
```

---

# 32. Concurrency Architecture

Swiftlet 不使用：

```text
one concurrency limit
```

而使用 resource-specific concurrency。

```text
             Scheduler
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
 AgentSlots   LLMSlots    BashSlots
                              │
                              ▼
                         SandboxSlots
```

---

# 33. Default Limits

例如：

```text
active AgentRun:
1000

LLM requests:
200

Bash processes:
64

Sandbox initialization:
32
```

具体值必须根据：

```text
machine
provider QPS
RAM
CPU
```

动态配置。

---

# 34. Backpressure

所有 queue 必须：

```text
BOUNDED
```

禁止：

```text
unbounded_channel
```

典型：

```text
producer
   │
   ▼
bounded task queue
   │
   ▼
scheduler
```

达到限制后：

```text
producer waits
```

而不是：

```text
RAM grows
```

---

# 35. Cancellation

每个 Run 必须支持 cancellation。

例如：

```text
batch cancelled
provider unavailable
global shutdown
timeout reached
```

必须传播：

```text
CancellationToken
```

到：

```text
model request
tool
bash
sandbox
```

最终保证：

```text
no orphan process
```

---

# 36. Runtime Memory Strategy

V1 优先原则：

```text
Avoid allocation
>
Optimize allocator
```

建议：

```text
Arc<str>
Bytes
Cow
typed struct
shared immutable config
```

避免：

```text
repeated String clone
huge Vec<Value>
duplicated Bash output
duplicated file contents
```

---

# 37. Global Shared State

以下资源应共享：

```text
HTTP Client
Skill Registry
Model Config
Static System Prompt
Tokenizer metadata
Telemetry handle
```

采用：

```text
Arc<T>
```

避免 AgentRun duplication。

---

# 38. Workspace Lifecycle

每个 task：

```text
create workspace
      │
      ▼
run agent
      │
      ▼
capture artifacts
      │
      ▼
cleanup
```

模式：

```text
ephemeral by default
```

Debug 模式：

```text
preserve on failure
```

---

# 39. Input Specification

V1 推荐 JSONL。

Example：

```json
{
  "task_id": "case-182",
  "objective": "Determine whether the implementation fixes the reported bug.",
  "workspace": "./cases/case-182",
  "skill": "code_judge",
  "model": "default",
  "metadata": {}
}
```

---

# 40. Output Specification

```json
{
  "task_id": "case-182",

  "status": "completed",

  "result": {
    "label": "PASS",
    "score": 1.0
  },

  "usage": {
    "model_calls": 3,
    "tool_calls": 4,
    "input_tokens": 12450,
    "output_tokens": 1132
  },

  "runtime": {
    "latency_ms": 8231
  }
}
```

---

# 41. CLI

基础：

```bash
swiftlet run input.jsonl
```

控制并发：

```bash
swiftlet run input.jsonl \
  --jobs 500 \
  --llm-concurrency 100 \
  --bash-concurrency 32
```

指定 Skill：

```bash
swiftlet run input.jsonl \
  --skill code_judge
```

结果：

```bash
swiftlet run input.jsonl \
  --output result.jsonl
```

---

# 42. Server Mode

后续支持：

```bash
swiftlet serve
```

Interface：

```text
POST /runs
POST /batches
GET /runs/:id
POST /runs/:id/cancel
```

但 Server Mode 不应该改变 Runtime Core。

架构：

```text
CLI ───────┐
           │
HTTP ──────┼─► Swiftlet Runtime
           │
Queue ─────┘
```

---

# 43. Telemetry

每一个 Run 至少记录：

```text
run_id
task_id

queue_wait_ms
sandbox_start_ms

model_latency_ms
tool_latency_ms
total_latency_ms

input_tokens
output_tokens

model_calls
tool_calls

context_peak_bytes
context_peak_tokens

workspace_bytes

finish_reason
error_type
```

---

# 44. Finish Reasons

统一：

```rust
enum FinishReason {
    Completed,

    MaxSteps,
    MaxModelCalls,
    MaxToolCalls,

    ContextLimit,

    WallTimeout,
    ModelTimeout,
    ToolTimeout,

    InvalidOutput,

    SandboxFailure,

    Cancelled,

    InternalError,
}
```

---

# 45. Design Performance Targets

## Runtime

| Metric | V1 Target | Stretch |
|---|---:|---:|
| Worker cold start | <100 ms | <30 ms |
| AgentRun creation | <1 ms | <100 µs |
| Runtime-only idle memory / run | <100 KB | <30 KB |
| Runtime CPU overhead | <2% | <1% |
| Orphan process | 0 | 0 |
| Unbounded queue | 0 | 0 |

---

# 46. Concurrency Targets

单 Worker benchmark：

```text
1
10
100
500
1000
```

concurrent AgentRuns。

需要测量：

```text
throughput
RSS
CPU
P50
P95
P99
open FD
process count
queue depth
```

关键判断：

> 当大量 Agent 等待 LLM I/O 时，增加 AgentRun 不应该显著增加 CPU。

---

# 47. Memory Targets

假设：

```text
1000 suspended AgentRuns
```

Swiftlet runtime metadata 目标：

```text
< 100 MB
```

Stretch：

```text
< 50 MB
```

不包含：

```text
Bash subprocess
workspace
HTTP payload buffers
```

---

# 48. Stability Targets

运行：

```text
100K tasks
```

要求：

```text
worker crash = 0
orphan process = 0
memory leak trend = 0
task lost = 0
```

单 task failure 不得影响其它 run。

---

# 49. Security Requirements

MUST：

```text
workspace path isolation
command timeout
process tree cleanup
stdout/stderr limit
memory limit
PID limit
CPU limit
network configurable
symlink escape prevention
```

SHOULD：

```text
seccomp
namespace
read-only base FS
```

Optional：

```text
gVisor
Firecracker
```

---

# 50. Repository Structure

推荐：

```text
swiftlet/
│
├── crates/
│   │
│   ├── swiftlet-core/
│   │
│   │   ├── run.rs
│   │   │
│   │   ├── state.rs
│   │   │
│   │   └── limits.rs
│   │
│   ├── swiftlet-context/
│   │
│   ├── swiftlet-model/
│   │
│   ├── swiftlet-tools/
│   │
│   ├── swiftlet-sandbox/
│   │
│   └── swiftlet-telemetry/
│
├── src/
│   └── main.rs
│
├── skills/
│
├── benchmarks/
│
└── tests/
```

如果初期团队规模很小，建议甚至先：

```text
single crate
```

直到代码超过约：

```text
5K–10K LOC
```

再拆 crate。

避免过早模块化。

---

# 51. V1 Minimal Architecture

第一版实际上只需要：

```text
main.rs

runtime.rs
context.rs
model.rs

tools/
  read.rs
  write.rs
  bash.rs

sandbox.rs
skill.rs
telemetry.rs
```

Agent Core 应争取：

```text
< 1000 LOC
```

理想：

```text
300–800 LOC
```

不包括 HTTP protocol 和 sandbox implementation。

---

# 52. Development Phases

## Phase 0 — Runtime Skeleton

实现：

```text
AgentRun
ModelClient
Read
Bash
StopPolicy
JSONL input/output
```

验证：

```text
100 concurrent runs
```

---

## Phase 1 — Judge MVP

加入：

```text
Write
Skill
Structured Output
Context Budget
Telemetry
```

目标：

```text
真实 benchmark 可用
```

---

## Phase 2 — Resource Runtime

加入：

```text
bounded scheduler
LLM semaphore
Bash semaphore
Cancellation
process cleanup
```

目标：

```text
500–1000 concurrent AgentRuns
```

---

## Phase 3 — Sandbox

加入：

```text
namespace
cgroup
seccomp
```

目标：

```text
untrusted code execution
```

---

## Phase 4 — Optimization

仅根据 profile 优化：

```text
allocation
JSON
buffer copy
context projection
connection management
```

禁止 speculative optimization。

---

## Phase 5 — Distributed Worker

如果 workload 证明需要：

```text
Dispatcher
     │
     ├── Swiftlet Worker
     ├── Swiftlet Worker
     ├── Swiftlet Worker
     └── Swiftlet Worker
```

这时再考虑：

```text
Redis
NATS
Kafka
Kubernetes
```

而不是之前。

---

# 53. Benchmark Suite

Swiftlet 自身至少需要以下 benchmark。

## Bench A — Empty Run

```text
No tools
mock model
```

测：

```text
AgentRun creation
runtime overhead
```

---

## Bench B — LLM-bound

```text
1000 runs
mock 2 sec HTTP latency
```

测：

```text
RSS
CPU
scheduler efficiency
```

---

## Bench C — Tool-bound

大量：

```text
Read
small Bash
```

测：

```text
process throughput
FD
CPU
memory
```

---

## Bench D — Adversarial

执行：

```text
yes
fork bomb
sleep forever
huge file
huge stdout
recursive process
```

验证：

```text
limit
kill
cleanup
```

---

## Bench E — Long Batch

```text
100K AgentRuns
```

检查：

```text
RSS slope
FD slope
process leak
latency degradation
```

---

# 54. Success Criteria

Swiftlet V1 成功不应该定义成：

> Agent 能完成多少不同类型的任务。

而应该定义成：

> 在最少 runtime abstraction 和最低资源成本下，是否能够可靠运行大量 Agentic Judge。

核心验收：

```text
1000 lightweight concurrent AgentRuns
+
bounded memory
+
bounded processes
+
bounded context
+
structured results
+
zero worker crash
```

---

# 55. Architectural Boundary

Swiftlet Core 负责：

```text
Agent execution
Context lifecycle
Tool dispatch
Resource scheduling
Cancellation
Telemetry
```

Swiftlet Core 不负责：

```text
Dataset management
Benchmark definition
Training pipeline
Experiment tracking
Distributed orchestration
Model serving
Container scheduling
```

这些应该属于上层。

---

# 56. Long-term Positioning

Swiftlet 最终不应该只服务：

```text
Agentic Judge
```

更准确的底层定位是：

> **Lightweight Agent Execution Runtime**

其上可以构建：

```text
Agentic Judge
Agentic Labeler
Agentic Grader
Agentic Validator
Agentic Filter
Teacher Runner
Reward Generator
Regression Runner
```

架构关系：

```text
               Applications

 Agentic      Agentic      Reward
 Judge        Labeler      Generator
    │            │             │
    └────────────┼─────────────┘
                 │
                 ▼
             Swiftlet
                 │
        Lightweight Agent
        Execution Runtime
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
     LLM        Tools     Sandbox
```

因此 Swiftlet 最重要的长期价值不是：

> another Agent.

而是：

> **the execution substrate for running enormous numbers of small agents efficiently.**

---

# 57. One-line Definition

对内：

> **Swiftlet is a Rust-native lightweight agent runtime optimized for high-concurrency Agentic Judge workloads.**

更偏产品：

> **Run thousands of tiny agents, fast.**

更偏工程：

> **Minimal runtime. Bounded resources. Massive concurrency.**

推荐作为 README tagline：

> **Swiftlet — A tiny, fast and highly concurrent runtime for Agentic workloads.**