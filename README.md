# Flowbit

Rust AI agent 框架。用 JSON DAG 声明式编排多 agent 工作流。

## 特性

- **DAG 工作流**:JSON 声明,纯路径变量引用 `{{path.to.field}}`。
- **多 agent 编排**:一个 workflow 内多个 agent 协作。
- **Blender shader 式节点**:多输入多输出,原语可组合成 composite。
- **Rig 集成**:直接使用 `rig::Agent`,支持 OpenAI / Anthropic / Gemini。
- **Tool 扩展**:Rust 注册表 + MCP 服务集成。
- **流式输出**:LLM 生成增量通过 SSE 实时下发。
- **Checkpoint 恢复**:每节点完成后写快照,支持断点续跑。
- **Web UI**:Vue 3 + TDesign + vue-flow,五页方案。
- **失败策略**:fail-fast,节点级完整 retry 配置。

## 快速开始

```bash
# 构建
cargo build

# 运行 workflow
cargo run -- run examples/basic.json --input '{"query": "hello"}'

# 启动 server 与 Web UI
cargo run -- serve --port 8080
```

## 项目结构

```
Flowbit/
├── flowbit-core/           # 核心库: DAG、节点、执行、存储、错误
├── flowbit-cli/            # 命令行工具
├── flowbit-server/         # HTTP server + Web UI 托管
│   └── web/                # Vue 3 前端
└── flowbit-examples/       # 示例 workflow
```

## Workflow 示例

```json
{
  "name": "triage",
  "inputs": [{"name": "query", "type": "string"}],
  "agents": {
    "planner": {
      "model": {"provider": "openai", "name": "gpt-4"},
      "system": "You plan.",
      "tools": [{"impl": "search"}]
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
      "inputs": {"prompt": "{{plan.result}}"},
      "retry": {"max": 3, "backoff": "exponential"}
    }
  ],
  "edges": [["plan", "act"]]
}
```

## 文档

- `docs/ARCHITECTURE.md` — 完整架构设计(36 项决策)。
- `AGENTS.md` — 项目约定与编码规范。
- [rig.rs 文档](https://rig.rs/docs) — LLM 与 tool 抽象。
- [Vue Flow 文档](https://vue-flow.dev) — DAG 画布。

## 配置

项目根目录 `flowbit.toml`,密钥通过环境变量注入。

```toml
[project]
name = "my-agent"

[runtime]
max_parallel_runs = 4
per_run_concurrency = 4

[llm.providers.openai]
api_key_env = "OPENAI_API_KEY"
default_model = "gpt-4"

[server]
host = "127.0.0.1"
port = 8080
webui = true
api_token_env = "FLOWBIT_API_TOKEN"
```

## 贡献

遵循 `AGENTS.md` 的编码与写作约定。修改前请阅读 `docs/TODO.md`的优先级清单。

## License

Apache License 2.0。见 [LICENSE](LICENSE)。
