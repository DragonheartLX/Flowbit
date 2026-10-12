# TODO

按优先级排期。P0 阻塞项，P1 上线必需，P2 增强，P3 完善。

## P0 — 核心引擎

- [ ] 初始化 workspace 依赖，写入 `[workspace.dependencies]`。
- [ ] 引入 rig 并锁定 openai / anthropic / gemini 三个 provider features。
- [ ] 引入 jsonschema crate，用于 params_schema 校验。
- [ ] 引入 sha2 crate，用于 checkpoint 输入指纹。
- [ ] 初始化 sqlx 与 migrations 目录。
- [ ] 实现初始 migration 001_init.sql，包含四张表与索引。
- [ ] 初始化 tracing 日志，默认 JSON 输出，支持 env-filter 调级别。
- [ ] 实现 RedactionLayer tracing 层，自动脱敏密钥。
- [ ] 定义 RunError 分层枚举，接入 thiserror。
- [ ] 实现错误码枚举 E_模块_详情，完整覆盖所有模块。
- [ ] CLI 层用 anyhow 聚合错误。
- [ ] 定义 workflow JSON schema，产出 JSON Schema 校验文件。
- [ ] 实现 workflow 解析器，文件读取到 DAG 结构。
- [ ] 引入 petgraph crate，用于 DAG 拓扑排序与并行度计算。
- [ ] 实现 DAG 拓扑排序与并行度计算，基于 petgraph。
- [ ] 定义 Node trait，多输入多输出，参数可配。
- [ ] 定义 NodePorts 与 ParamsSchema 类型。
- [ ] 实现 NodeKind 分发与注册。
- [ ] 实现 agent 节点，封装 `rig::Agent::invoke()`，支持流式下发生成。
- [ ] 实现 tool 节点，直接调用 rig tool。
- [ ] 实现 Rust tool 注册表，支持 `#[tool]` 装饰器。
- [ ] 实现全局 Context，JSON 存储加 `{{path.to.field}}` 解析。
- [ ] 实现变量解析失败时混合模式：必需解析期报，可选带 default。
- [ ] 实现 per-run 有界 worker pool，可配置。
- [ ] 实现 fail-fast 与节点级完整 retry 配置，含退避公式与抖动。
- [ ] 实现 `.flowbit.toml` 配置加载，密钥从环境变量读取。
- [ ] 实现 checkpoint 写入与读取，包含 input_fingerprint。
- [ ] 实现从 checkpoint 恢复执行，严格跳过已完成节点。
- [ ] 实现 API token 中间件，Bearer 校验。

## P1 — CLI 与库

- [ ] 提供 `flowbit run` 子命令，端到端跑 workflow。
- [ ] 提供 `flowbit run --resume <run_id>`，从 checkpoint 恢复。
- [ ] 提供 `flowbit run --input <JSON>`，传入口参数。
- [ ] 提供 `flowbit run --output <path>`，结果写入文件。
- [ ] 提供 `flowbit validate` 子命令，静态校验 workflow。
- [ ] 提供 `flowbit validate --schema`，输出校验后的 schema。
- [ ] 提供 `flowbit list-nodes` 子命令，列出可用节点类型与 ports。
- [ ] `flowbit-core` 对外暴露稳定 API，发布到 crates.io。
- [ ] 结构化日志接入 tracing，默认 JSON 输出。
- [ ] 实现 if 控制流节点。
- [ ] 实现 loop 控制流节点。
- [ ] 实现 map 控制流节点。
- [ ] 实现 composite 节点，内嵌 sub_workflow 引用。
- [ ] 实现 composite 节点，外部文件引用。
- [ ] 实现 composite 节点，显式 port_map 映射。
- [ ] 实现 composite 前缀式命名空间引用。
- [ ] 实现 custom 节点 registry，用户可注册自定义节点。
- [ ] 实现 MCP 服务集成，从 .flowbit.toml 自动导入 tool。
- [ ] 集成测试，核心模块覆盖率不低于 70%。

## P2 — Server 与 Web UI

- [ ] 实现 Axum HTTP server 骨架。
- [ ] 实现 API token 认证中间件，拦截无 Bearer 请求。
- [ ] 实现 workflow CRUD REST 端点。
- [ ] 实现执行触发 REST 端点，POST /workflows/:id/run。
- [ ] 实现 run 状态查询与历史列表端点。
- [ ] 实现 SSE 事件推送，9 类固定事件枚举。
- [ ] 实现 SSE 断线重连与序号续传。
- [ ] 实现 cancel 与 resume 端点。
- [ ] 实现节点与 tool registry 端点。
- [ ] 实现 config 读写端点。
- [ ] Web UI 脚手架，Vite + Vue 3 + TDesign。
- [ ] 实现 Editor 页，DAG 画布拖拽连线。
- [ ] 实现 Runner 页，参数输入与实时日志。
- [ ] 实现 Monitor 页，历史执行 trace 查看。
- [ ] 实现 Registry 页，节点与 tool 列表。
- [ ] 实现 Settings 页，`.flowbit.toml` 编辑，含脱敏开关。
- [ ] Web UI 集成 API token 存储与请求拦截。
- [ ] 实现 `flowbit serve` 子命令，启动 server 加 Web UI。
- [ ] Web UI 启动由 `[server].webui` 配置控制。

## P3 — 完善

- [ ] agent 定义外部化，支持独立 agents 文件被 workflow 引用。
- [ ] 示例 workflow 集，覆盖 RAG、多轮对话、批量处理三类场景。
- [ ] OpenTelemetry trace 接入。
- [ ] 文档与教程。
- [ ] crates.io 发布稳定版本号与 changelog。

## 维护约定

- 新增条目按 P0 / P1 / P2 / P3 归入对应分区。
- 完成后直接删除，不留勾。
- 每周五更新一次优先级。
