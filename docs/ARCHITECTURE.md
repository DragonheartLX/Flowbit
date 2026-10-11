# 项目结构

Flowbit 是基于 Rust 的 AI agent 框架。用 JSON DAG 声明式编排多 agent 工作流。

## 设计决策

- 工作流形式：DAG 图。
- 序列化格式：JSON。
- 并发模型：tokio 全异步。
- 节点模型：多输入多输出，参数可配，Blender shader 式原语。
- 数据流：全局 JSON 上下文 + `{{path.to.field}}` 变量引用。
- 变量能力：纯路径引用，不支持运算符或函数。
- 变量失败时机：混合模式。必需变量解析期失败，可选变量带默认值。
- Agent 编排：workflow 编排多个 agent。
- Rig 集成：直接使用 `rig::Agent`，agent 定义完全转发 rig 配置字段。
- Composite 定义：内嵌 sub_workflow 与独立文件引用，两者都支持。
- Composite 映射：显式 port_map，内部节点前缀式命名空间引用。
- 端口声明：内置节点 kind 预设 ports，custom 节点显式声明。
- Params schema：JSON Schema 标准格式校验。
- Tool 定义：Rust 注册表 + MCP 服务集成。
- 失败策略：fail-fast，节点级可配置 retry。
- Retry 配置：节点级完整配置，含 max / delay / backoff / retryable。
- 运行时能力：per-run 有界 worker pool、LLM 流式输出、checkpoint 与恢复。
- Checkpoint 恢复：严格跳过，跳过已完成节点。
- 错误模型：分层枚举 RunError。
- 错误码命名：`E_模块_详情` 分层枚举。
- SSE 事件：固定 9 类事件枚举。
- 交付形态：CLI + 库 + HTTP server。
- Web UI：Vite + Vue + TDesign，五页完整方案。
- 后端：Axum，SQLite 持久化。
- Server 认证：API token 中间件。
- Agent 定义：内嵌 workflow JSON。
- Provider 配置：`.flowbit.toml` + 环境变量。
- 敏感字段脱敏：自动模式匹配。

## 现状

Bootstrap 阶段。仅 workspace 骨架。

```
Flowbit/
├── Cargo.toml
├── .gitignore
├── docs/
│   ├── TODO.md
│   └── ARCHITECTURE.md
└── flowbit/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

## 规划

### 目录结构

```
Flowbit/
├── Cargo.toml                     # workspace
├── flowbit-core/                  # 核心库
│   ├── src/
│   │   ├── lib.rs
│   │   ├── workflow/              # workflow 解析与验证
│   │   ├── dag/                   # DAG 结构与拓扑调度
│   │   ├── node/                  # 节点 trait 与内置实现
│   │   ├── agent/                 # 基于 rig::Agent 的封装
│   │   ├── context/               # 全局上下文与变量解析
│   │   ├── composite/             # 复合节点加载
│   │   ├── error/                 # RunError 与重试策略
│   │   └── checkpoint/            # 快照与恢复
│   └── tests/
├── flowbit-cli/                   # CLI 二进制
│   └── src/
│       ├── main.rs
│       ├── config.rs
│       └── commands/              # run / validate / dev
├── flowbit-server/                # HTTP server + Web UI
│   ├── src/
│   │   ├── main.rs                # Axum
│   │   ├── api/                   # REST 端点
│   │   ├── stream/                # SSE 推送
│   │   ├── auth/                  # token 中间件
│   │   └── storage/               # SQLite 访问
│   └── web/                       # Vite + Vue + TDesign
│       ├── src/
│       │   ├── editor/            # DAG 画布
│       │   ├── runner/            # 执行面板
│       │   ├── monitor/           # 历史执行
│       │   ├── registry/          # 节点与 tool 列表
│       │   └── settings/          # 配置管理
│       ├── package.json
│       └── vite.config.ts
├── flowbit-examples/              # 示例 workflow
├── docs/
└── target/
```

### 模块职责

- `flowbit-core`：运行时核心。CLI 与 server 都依赖它。
- `flowbit-cli`：命令入口。加载配置、调用 core、输出结果。
- `flowbit-server`：REST + SSE + 托管 Web UI + 认证中间件。
- `flowbit-examples`：开箱可用的示例 workflow。

### Workflow JSON

顶层字段：`name`、`inputs`、`agents`、`nodes`、`edges`、可选 `sub_workflows`。

```json
{
  "name": "triage",
  "inputs": [{"name": "query", "type": "string"}],
  "agents": {
    "planner": {
      "model": {"provider": "openai", "name": "gpt-4"},
      "system": "You plan.",
      "tools": [{"impl": "search"}],
      "temperature": 0.7,
      "max_tokens": 4096,
      "response_format": {"type": "json_object"}
    }
  },
  "nodes": [
    {
      "id": "plan",
      "kind": "agent",
      "params": {"agent": "planner"},
      "inputs": {"prompt": "{{inputs.query}}"}
    },
    {
      "id": "act",
      "kind": "agent",
      "params": {"agent": "planner"},
      "inputs": {
        "prompt": "{{plan.result}}",
        "context": "{{plan.steps | default([])}}"
      },
      "retry": {
        "max": 3,
        "delay_ms": 1000,
        "backoff": "exponential",
        "factor": 2.0,
        "max_delay_ms": 10000,
        "retryable": ["E_LLM_TIMEOUT", "E_LLM_RATE_LIMIT", "E_LLM_TRANSIENT"]
      }
    }
  ],
  "edges": [["plan", "act"]]
}
```

- `nodes[].kind` 决定节点类型。
- `nodes[].inputs` 命名输入，值是变量引用或字面量。
- `{{node_id.output_port}}` 引用上游输出。
- `{{path | default(value)}}` 可选变量带默认值。
- `edges` 定义拓扑顺序，变量解析独立于拓扑。
- 必需变量解析期校验，缺失立即报错。

### 节点 trait

多输入多输出，参数可配。

```rust
#[async_trait]
pub trait Node: Send + Sync {
    fn id(&self) -> &NodeId;
    fn kind(&self) -> NodeKind;
    fn ports(&self) -> NodePorts;
    fn params_schema(&self) -> ParamsSchema;

    async fn execute(
        &self,
        ctx: &mut Context,
        inputs: NodeInputs,
        params: NodeParams,
        stream: Option<StreamSink>,
    ) -> Result<NodeOutputs, RunError>;
}
```

- `NodePorts` 声明输入输出端口的名字与类型。
- `NodeInputs` / `NodeOutputs` 是 `HashMap<String, Value>`。
- `ParamsSchema` 是 JSON Schema 对象。
- `StreamSink` 仅在 LLM 节点启用，异步下发生成增量。
- 运行时按 `NodeKind` 分发到具体实现。

### 节点类型

系统提供原语，用户可组合成 composite。

- `agent`：调用 `rig::Agent`。端口 in=`{prompt, system?}`，out=`{result, tokens?}`。
- `tool`：直接调用 tool。端口 in=`{args}`，out=`{result}`。
- `if`：条件分支。端口 in=`{condition}`，出边指向 then/else 目标节点。
- `loop`：循环子 DAG。参数 `until`、`max_iterations`。
- `map`：对数组应用子 DAG。参数 `source`、`body`、`output`。
- `composite`：引用内嵌 `sub_workflows.<name>` 或外部文件路径。
- `custom`：用户注册的自定义节点，需显式声明 ports 与 params_schema。

### 端口声明规则

- 内置节点 kind 在代码里硬编码 ports，JSON 不写。
- custom 节点必须显式声明 ports。
- 端口类型支持 `string`、`number`、`bool`、`object`、`array`、`any`。
- custom 节点需声明 `params_schema`，格式为 JSON Schema。

```json
{
  "id": "extract",
  "kind": "custom",
  "impl": "my_pkg::Extractor",
  "params_schema": {
    "type": "object",
    "required": ["model"],
    "properties": {
      "model": {"type": "string", "enum": ["ner", "sentiment"]},
      "top_k": {"type": "integer", "minimum": 1, "maximum": 10, "default": 3}
    }
  }
}
```

### 复合节点

内嵌 `sub_workflows` 或外部文件。父节点通过 `port_map` 显式映射输入输出。

```json
{
  "id": "sum1",
  "kind": "composite",
  "ref": "summarize",
  "port_map": {
    "in":  {"prompt": "text", "limit": "limit"},
    "out": {"result": "summary"}
  },
  "inputs": {
    "prompt": "{{plan.result}}",
    "limit": "100"
  }
}
```

- `ref` 指向内嵌 `sub_workflows.summarize` 或外部 workflow 文件路径。
- `port_map.in`：父输入名 → 子 workflow inputs 名。
- `port_map.out`：父输出名 → 子 workflow outputs 名。
- 外层可用 `{{sum1.result}}` 引用父输出。
- 前缀式命名空间：内部节点以 `sum1.<node_id>` 暴露，如 `{{sum1.plan.result}}`。
- 内部节点重命名影响外层引用，需谨慎。

### 数据流

- 全局 JSON 上下文贯穿执行全程。
- 变量语法：`{{path.to.field}}`，仅支持纯路径引用。
- 支持字段访问、数组索引、字面量与引用混合。
- 可选变量：`{{path | default(value)}}`，缺失时取默认值。
- 必需变量解析期校验，缺失立即 `E_PARSE_UNDEFINED_VAR`。
- `inputs.*` 来自 workflow 入口参数。
- `<node_id>.<output_port>` 来自节点输出。
- `sum1.<sub_node>.<output>` 来自 composite 内部节点。
- 解析期完成替换，运行时不再动态求值。

### Agent 模型

Agent 内嵌在 workflow 顶层 `agents` 字段定义，完全转发 `rig::Agent` 配置。

```json
"agents": {
  "planner": {
    "model": {"provider": "openai", "name": "gpt-4"},
    "system": "You plan.",
    "tools": [{"impl": "search"}],
    "metadata": {},
    "context": [],
    "completion_options": {
      "temperature": 0.7,
      "max_tokens": 4096,
      "top_p": 1.0,
      "stop": ["\n"],
      "response_format": {"type": "json_object"}
    }
  }
}
```

- 每个 agent 对应一个 `rig::Agent` 实例。
- 字段完全对齐 `rig::Agent` 当前版本。
- rig 升级时 schema 同步更新。
- 同一 workflow 内 agent 可复用。

### Tool 定义

两种入口：Rust 注册表与 MCP 服务。

- Rust 注册表：`#[tool(name = "search")]` 装饰器标注异步函数，workflow 按名引用。
- MCP 服务：`.flowbit.toml` 声明 MCP server 端点，自动导入 tool。

```toml
[mcp.servers.search]
command = "npx"
args = ["@modelcontextprotocol/server-filesystem", "/path"]
```

- workflow `agents[].tools[]` 字段按名引用已注册 tool。
- 参数 schema 通过 `JsonSchema` derive 自动生成。

### 执行运行时

- 每个 run 独立有界 worker pool，默认 4 并发，`[runtime].per_run_concurrency` 可配。
- 全局 run 数量上限，`[runtime].max_parallel_runs` 默认 4。
- LLM 节点流式下发生成增量到 SSE。
- 每节点完成后写 checkpoint 到 SQLite。
- 中断后可从最后一个 checkpoint 恢复，严格跳过已完成节点。
- 恢复入口：`flowbit run --resume <run_id>` 或 `POST /v1/runs/:id/resume`。
- 未完成 checkpoint 的节点会重跑，可能产生副作用重复。

### Retry 配置

节点级完整配置。

```json
"retry": {
  "max": 3,
  "delay_ms": 1000,
  "backoff": "exponential",
  "factor": 2.0,
  "max_delay_ms": 10000,
  "retryable": ["E_LLM_TIMEOUT", "E_LLM_RATE_LIMIT", "E_LLM_TRANSIENT"]
}
```

- `backoff` 取值 `linear` 或 `exponential`。
- `factor` 指数退避系数，默认 2.0。
- `max_delay_ms` 退避上限。
- `retryable` 错误码白名单，其他错误不重试。
- 退避公式：`delay_n = min(delay_ms * factor^n, max_delay_ms) + jitter(0.1 * delay)`。
- 抖动 10%，避免同步重试。
- 全局默认值写在 `.flowbit.toml` 的 `[runtime.retry]`，节点可覆盖。

### 错误模型

分层枚举，结构化可匹配。

```rust
pub enum RunError {
    Config(ConfigError),
    Parse(ParseError),
    Exec(ExecError),
    Tool(ToolError),
    Llm(LlmError),
    Timeout(TimeoutError),
    Io(IoError),
    Internal(InternalError),
}

impl RunError {
    pub fn code(&self) -> &'static str;
    pub fn node_id(&self) -> Option<&NodeId>;
    pub fn is_retryable(&self) -> bool;
}
```

错误码完整枚举：

- `E_CONFIG_MISSING`、`E_CONFIG_INVALID`
- `E_PARSE_SYNTAX`、`E_PARSE_UNDEFINED_VAR`
- `E_EXEC_DAG_CYCLE`、`E_EXEC_TIMEOUT`、`E_EXEC_UNEXPECTED_STATE`
- `E_TOOL_NOT_FOUND`、`E_TOOL_SCHEMA_INVALID`、`E_TOOL_EXEC_FAILED`
- `E_LLM_TIMEOUT`、`E_LLM_RATE_LIMIT`、`E_LLM_CONTEXT_OVERFLOW`、`E_LLM_TRANSIENT`、`E_LLM_AUTH_FAILED`
- `E_IO_FILE_NOT_FOUND`、`E_IO_PERMISSION`
- `E_INTERNAL_UNEXPECTED`

- `is_retryable` 默认仅 LLM 类与 Timeout 类。
- 错误对象带 `node_id`，方便定位失败节点。

### 配置

项目根目录 `.flowbit.toml`。

```toml
[project]
name = "my-agent"

[runtime]
max_parallel_runs = 4
per_run_concurrency = 4

[runtime.retry]
max = 3
delay_ms = 1000
backoff = "exponential"
factor = 2.0
max_delay_ms = 10000

[runtime.redaction]
enabled = true

[storage]
path = "~/.flowbit/data.db"

[llm.providers.openai]
api_key_env = "OPENAI_API_KEY"
default_model = "gpt-4"

[mcp.servers.search]
command = "npx"
args = ["@modelcontextprotocol/server-filesystem", "/path"]

[server]
host = "127.0.0.1"
port = 8080
webui = true
api_token_env = "FLOWBIT_API_TOKEN"
```

- 密钥统一走环境变量。
- 配置文件只写 `*_env` 字段。
- Web UI 启动由 `[server].webui` 控制。
- API token 由环境变量注入，缺失时 server 拒绝启动。

### 交付形态

- CLI：`flowbit run <workflow.json>` 端到端跑一次。
- CLI 附加：`flowbit validate`、`flowbit list-nodes`、`flowbit serve`。
- 库：`flowbit-core` 供其他 Rust 项目嵌入。
- Server：`flowbit serve` 启动 HTTP + Web UI，强制启用 API token。

### 认证

API token 中间件。

- `.flowbit.toml` 声明 `api_token_env`，指向环境变量名。
- 启动时读取环境变量，为空则拒绝启动。
- 请求头 `Authorization: Bearer <token>`。
- 校验失败返回 `401 Unauthorized`。
- 静态资源与登录页不校验。

### Web UI 页面

Vue 3 + Vite + TDesign，五页方案。

- `/editor`：DAG 画布，拖拽节点、连线、属性面板。
- `/runner`：执行面板，输入参数、启动、实时日志。
- `/monitor`：历史执行列表，查看完整 trace。
- `/registry`：节点与 tool 列表，查看 ports 与 params schema。
- `/settings`：配置管理，`.flowbit.toml` 编辑。

### HTTP API

REST 端点，`/v1` 前缀，除认证外全部需要 Bearer token。

- `GET /v1/workflows`：列表。
- `POST /v1/workflows`：创建。
- `GET /v1/workflows/:id`：详情。
- `PUT /v1/workflows/:id`：更新。
- `DELETE /v1/workflows/:id`：删除。
- `POST /v1/workflows/:id/run`：触发执行，body 为 input JSON。
- `GET /v1/runs`：历史列表。
- `GET /v1/runs/:id`：状态与结果。
- `GET /v1/runs/:id/stream`：SSE 事件流。
- `POST /v1/runs/:id/cancel`：取消执行。
- `POST /v1/runs/:id/resume`：从 checkpoint 恢复。
- `GET /v1/nodes/registry`：节点列表。
- `POST /v1/nodes/registry`：注册自定义节点。
- `GET /v1/config`：读取配置。
- `PUT /v1/config`：更新配置。

### SSE 事件

固定 9 类，每行一个 JSON。

- `RunStarted { run_id }`
- `RunCompleted { run_id, output }`
- `RunFailed { run_id, error }`
- `RunCancelled { run_id }`
- `NodeStarted { run_id, node_id, kind }`
- `NodeStreamChunk { run_id, node_id, delta }`
- `NodeCompleted { run_id, node_id, output }`
- `NodeFailed { run_id, node_id, error }`
- `Log { run_id, node_id, level, message }`

- 事件按 run 隔离，前端按 `run_id` 归集。
- 前端订阅时自动重连，服务端支持从事件序号续传。

### 敏感字段脱敏

自动模式匹配，tracing 层实现。

- 匹配密钥模式：`sk-` 开头（OpenAI）、`sk-ant-`（Anthropic）、`AIza[0-9A-Za-z_-]{35}`（Gemini）。
- 匹配字段名：`api_key`、`token`、`secret`、`password`、`credential`。
- 匹配环境变量赋值：`*_KEY=***`、`*_TOKEN=***`。
- 命中替换为 `[REDACTED]`。
- 可通过 `[runtime].redaction.enabled = false` 关闭。

### 存储

SQLite 持久化，四张表。

- `workflows`：id、name、content、version、created_at、updated_at。
- `runs`：id、workflow_id、status、input、output、error、started_at、ended_at。
- `checkpoints`：run_id、node_id、snapshot、created_at。
- `logs`：run_id、seq、node_id、level、message、payload、created_at。

- 外键：`runs.workflow_id` 指向 `workflows.id`。
- 主键：`checkpoints(run_id, node_id)`、`logs(run_id, seq)`。
- 存储路径：`~/.flowbit/data.db` 或 `[storage].path` 指定。

## 数据契约

### SQLite Migrations

初始 migration 文件 `migrations/001_init.sql`。

```sql
CREATE TABLE workflows (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  content JSON NOT NULL,
  version INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE runs (
  id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL REFERENCES workflows(id),
  status TEXT NOT NULL,
  input JSON,
  output JSON,
  error JSON,
  started_at TEXT NOT NULL,
  ended_at TEXT
);

CREATE TABLE checkpoints (
  run_id TEXT NOT NULL REFERENCES runs(id),
  node_id TEXT NOT NULL,
  snapshot JSON NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (run_id, node_id)
);

CREATE TABLE logs (
  run_id TEXT NOT NULL REFERENCES runs(id),
  seq INTEGER NOT NULL,
  node_id TEXT,
  level TEXT NOT NULL,
  message TEXT NOT NULL,
  payload JSON,
  created_at TEXT NOT NULL,
  PRIMARY KEY (run_id, seq)
);

CREATE INDEX idx_runs_workflow ON runs(workflow_id);
CREATE INDEX idx_runs_status ON runs(status);
CREATE INDEX idx_logs_run ON logs(run_id);
```

### Checkpoint Snapshot 结构

节点执行完成后写入。

```json
{
  "node_id": "plan",
  "node_kind": "agent",
  "input_fingerprint": "sha256:abc123",
  "inputs": {
    "prompt": "请分析这个问题..."
  },
  "outputs": {
    "result": {...},
    "tokens": 128
  },
  "duration_ms": 1523,
  "created_at": "2026-10-11T08:30:00Z"
}
```

- `input_fingerprint` 用于校验恢复时输入未变。
- `outputs` 完整保留节点输出，恢复时直接 apply。

### SSE Payload 示例

```json
data: {"type":"RunStarted","run_id":"r_abc123"}

data: {"type":"NodeStarted","run_id":"r_abc123","node_id":"plan","kind":"agent"}

data: {"type":"NodeStreamChunk","run_id":"r_abc123","node_id":"plan","delta":"计划如下："}

data: {"type":"NodeStreamChunk","run_id":"r_abc123","node_id":"plan","delta":"1. 分析问题..."}

data: {"type":"NodeCompleted","run_id":"r_abc123","node_id":"plan","output":{"result":{...},"tokens":128}}

data: {"type":"Log","run_id":"r_abc123","node_id":"plan","level":"info","message":"LLM call completed"}

data: {"type":"RunCompleted","run_id":"r_abc123","output":{...}}
```

### 日志 JSON 格式

tracing-subscriber 输出格式。

```json
{
  "timestamp": "2026-10-11T08:30:00.123Z",
  "level": "INFO",
  "target": "flowbit_core::node::agent",
  "message": "agent invoke completed",
  "run_id": "r_abc123",
  "node_id": "plan",
  "kind": "agent",
  "duration_ms": 1523,
  "tokens": 128
}
```

- 每行一个 JSON，便于 grep 与 ELK 接入。
- 敏感字段已由 RedactionLayer 脱敏。
- 支持环境变量 `RUST_LOG` 调整级别。

### CLI 命令签名

```
flowbit run <workflow.json> [OPTIONS]
  --input <JSON>        workflow 入口参数
  --resume <run_id>     从 checkpoint 恢复
  --no-stream           禁用流式输出
  --output <path>       结果写入文件

flowbit validate <workflow.json>
  --schema              输出校验后的 JSON Schema

flowbit list-nodes
  --kind <kind>         按类型过滤

flowbit serve [OPTIONS]
  --config <path>       指定配置文件，默认 .flowbit.toml
  --host <addr>         覆盖配置中的 host
  --port <port>         覆盖配置中的 port
  --no-webui            禁用 Web UI
```

- CLI 参数优先于配置文件。
- 所有命令输出结构化 JSON，便于脚本消费。
- 错误输出到 stderr，正常输出到 stdout。

## 依赖栈

Rust 与前端依赖分两栈。所有版本以 semver 兼容为准则，实际锁版本以 `Cargo.lock` 与 `package-lock.json` 为准。

### Rust 后端

写在 workspace 级 `[workspace.dependencies]`，子 crate 用 `dep.workspace = true` 引用。

```toml
[workspace.dependencies]

# 异步运行时
tokio = { version = "1", features = ["full", "macros"] }
async-trait = "0.1"
tokio-stream = "0.1"
futures = "0.3"

# LLM 与 tool
rig = { version = "0.44", features = ["openai", "anthropic", "gemini"] }

# HTTP server
axum = { version = "0.7", features = ["macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["fs", "trace", "cors"] }
eventsource-stream = "0.2"

# CLI
clap = { version = "4", features = ["derive", "color"] }
clap_complete = "4"

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
toml = "0.8"

# 校验
jsonschema = "0.17"

# 存储
sqlx = {
  version = "0.8",
  features = ["runtime-tokio", "sqlite", "migrate", "json"]
}

# 日志与追踪
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# 错误处理
thiserror = "2"
anyhow = "1"

# 工具
petgraph = "0.7"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
regex = "1"
sha2 = "0.10"
dirs = "6"
```

### Rust 测试依赖

仅在 `dev-dependencies` 中引入。

```toml
[workspace.dependencies]

# HTTP mock
wiremock = "0.6"

# trait mock
mockall = "0.13"

# 异步测试
tokio-test = "0.4"

# benchmark
criterion = "0.5"
```

### 依赖选择依据

- `tokio` 全特性：`full` 含 IO、timer、signal、rt-multi-thread，覆盖 CLI、server、core 三处场景。
- `rig` 默认启 `openai` + `anthropic` + `gemini`：开箱支持三大厂商，其他 provider 需显式 opt-in。
- `sqlx` 而非 `rusqlite`：异步原生，配合 `migrate` feature 内置迁移。
- `sqlx` 使用 offline 模式：`.sqlx/` 目录缓存查询元数据，无需 CI 拉真实 DB。
- `thiserror` + `anyhow`：库层强类型，CLI 层聚合，标准组合。
- `petgraph`：DAG 拓扑排序、并行度计算的标准库。
- `tracing`：结构化日志，`env-filter` 支持运行时调级别，`json` feature 输出 JSON。
- `jsonschema`：params_schema 校验，遵循 JSON Schema 标准。
- `sha2`：checkpoint 输入指纹计算。
- `clap` derive 模式：CLI 参数声明式定义。
- `dirs`：跨平台取 `~/.flowbit/` 路径。
- `uuid` v4：run_id、node 快照 id 用。
- `chrono`：时间戳与日志时间。
- `regex`：变量引用 `{{...}}` 解析与密钥脱敏用。

### 显式不引入的依赖

- `hyper`：通过 `axum` 间接使用，无需显式声明。
- `reqwest`：`rig` 已包含，避免版本冲突。
- `serde_yaml`：workflow 格式定 JSON，仅预留供 `.flowbit.toml` 之外的场景使用。
- `opentelemetry` / `opentelemetry-otlp`：P3 阶段接入，P0 不加。
- `diesel`：与 `sqlx` 二选一，选 `sqlx`。
- `config` / `figment`：`.flowbit.toml` 结构简单，手写 toml 加载足够。
- `color-eyre`：错误 UI 需求弱，不引入，避免与 tracing JSON 日志冲突。

### 前端依赖

`flowbit-server/web/package.json`。

```json
{
  "dependencies": {
    "vue": "^3.5",
    "vue-router": "^4.5",
    "pinia": "^3",
    "@vue-flow/core": "^1",
    "@vue-flow/controls": "^1",
    "@vue-flow/minimap": "^1",
    "@dagrejs/dagre": "^1",
    "@tdesign-vue-next": "^1.10",
    "@vueuse/core": "^11"
  },
  "devDependencies": {
    "vite": "^6",
    "typescript": "^5",
    "@vitejs/plugin-vue": "^5",
    "sass": "^1.80",
    "eslint": "^9",
    "@typescript-eslint/parser": "^8",
    "@typescript-eslint/eslint-plugin": "^8",
    "eslint-plugin-vue": "^9",
    "prettier": "^3"
  }
}
```

### 前端依赖选择依据

- `vue-flow` 三件套（core + controls + minimap）：DAG 画布核心，拖拽连线开箱即用。
- `@dagrejs/dagre`：图布局算法，自动排列节点位置。
- `@tdesign-vue-next`：UI 组件库，与 Vue 3 深度集成。
- `pinia`：Vue 3 官方推荐状态管理。
- `vue-router`：SPA 路由，五页导航。
- `@vueuse/core`：常用组合式 API 工具，跨浏览器封装。
- `vite`：前端构建工具，HMR 快。
- `sass`：样式预处理器，TDesign 主题定制用。
- TypeScript：类型安全。
- ESLint + Prettier：代码规范。

### 开发工具链

- `rustfmt`：Rust 格式化，`cargo fmt`。
- `clippy`：Rust 静态检查，`cargo clippy`。
- `cargo-watch`：文件变更自动重建。
- `just`：跨平台 task runner，替代 Makefile。
- `prettier`：前端格式化。
- `eslint`：前端 lint。
- GitHub Actions：CI，`P3` 阶段配置。
