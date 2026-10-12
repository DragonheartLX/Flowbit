# AGENTS.md

Flowbit 是基于 Rust 的 AI agent 框架。用 JSON DAG 声明式编排多 agent 工作流。

## 项目结构

5 个部分:

- `flowbit-core/` — 核心库, DAG / 节点 / 执行 / 存储 / 错误。
- `flowbit-cli/` — 命令行工具, `run` / `validate` / `list-nodes` / `serve`。
- `flowbit-server/` — HTTP server, Axum REST + SSE + 认证。
- `flowbit-server/web/` — 前端应用, Vue 3 + TDesign + vue-flow。
- `flowbit-examples/` — 示例 workflow 与 custom 节点。

## 关键决策

完整决策见 `docs/ARCHITECTURE.md`。

- Workflow: DAG + JSON, 纯路径变量引用 `{{path.to.field}}`。
- 执行: tokio 全异步, per-run 有界 worker pool, fail-fast + 完整 retry。
- Agent: 内嵌 workflow, 完全转发 `rig::Agent` 字段。
- Composite: 内嵌 sub_workflow + 外部文件, 显式 `port_map`, 前缀式命名空间。
- Tool: Rust 注册表 + MCP 服务集成。
- 节点: 多输入多输出, JSON Schema 校验 params。
- 存储: SQLite (sqlx) + 严格跳过 checkpoint。
- 错误: 分层 `RunError` + `E_模块_详情` 枚举。
- SSE: 9 类固定事件。
- 认证: API token 中间件。
- Web UI: Vue 3 + TDesign + vue-flow, 五页方案。
- 日志: tracing JSON + RedactionLayer 自动脱敏。

## 编码约定

- Rust edition 2024。
- 错误处理: 库层 `thiserror`, CLI 层 `anyhow`。
- 日志: `tracing` JSON 格式, 通过 `RedactionLayer` 脱敏密钥。
- 变量语法: `{{path.to.field}}` 纯路径, 可选 `{{path | default(value)}}`。
- 错误码: `E_模块_详情` 分层枚举, 全部在 `flowbit-core/src/error/codes.rs`。
- 命名: snake_case 函数, PascalCase 类型, SCREAMING_SNAKE_CASE 常量。
- 模块文件与目录同名: `flowbit_core::workflow::parser` ↔ `workflow/parser.rs`。

## 写作约定

- 注释与文档一句话讲清: 一句一个事实, 结论放句首, 不叠分句。
- `//` 单行注释为主, 复杂说明用 `///` 或独立 md。
- 中文注释与英文代码并存, 术语用英文。
- 示例代码必须可编译, 不允许伪代码。

## 测试约定

- 单元测试: 与被测模块同目录, `#[cfg(test)] mod tests`。
- 集成测试: `flowbit-core/tests/` 与 `flowbit-cli/tests/`。
- Workflow 校验: 每个 workflow 文件对应一个测试 fixture。
- LLM mock: 用 `wiremock` 拦 rig 的 HTTP 请求, 不真调 API。
- 覆盖率门槛: P1 结束前不低于 70%。

## 构建与运行

```bash
# 构建
cargo build

# 本地跑 workflow
cargo run -- run examples/basic.json --input '{"query": "hello"}'

# 静态校验
cargo run -- validate path/to/workflow.json

# 启动 server
cargo run -- serve --port 8080

# 前端开发
cd flowbit-server/web && npm install && npm run dev

# 测试
cargo test --workspace

# 覆盖率
cargo coverage --workspace
```

## 待办

见 `docs/TODO.md`。

- P0 核心引擎: 29 项
- P1 CLI 与库: 19 项
- P2 Server 与 Web UI: 19 项
- P3 完善: 5 项

## 参考

- `docs/ARCHITECTURE.md` — 完整架构文档。
- `docs/TODO.md` — 优先级待办。
- <https://rig.rs/docs> — rig 官方文档。
- <https://docs.rs/axum> — axum HTTP server。
- <https://docs.rs/sqlx> — sqlx SQLite。
- <https://vue-flow.dev> — 前端 DAG 画布。

## 注意事项

- IDE 开着工程时 rust-analyzer 会自动跑 `cargo check`, 语义改动一定来自人。
- `edit` 工具做纯 no-op 编辑会吞换行, 注释改写用 `edit` 时 oldText 与 newText 都要带尾随换行。
