# Agent Server 认证与会话：运行样本

固定来源为 [OpenHands/software-agent-sdk 提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98)。样本在 Linux、Python 3.13.16、上游 server/SDK/tools 1.58.2 上运行实际进程，显式监听 `127.0.0.1`，关闭 VS Code、工具预加载和遥测，使用独立临时 HOME、工作区、会话持久化目录、tmux 目录和测试 key。模型是本机 OpenAI 兼容 HTTP fixture；创建的 Agent 不加载工具。上游核心 SDK 依固定锁文件安装，server 和工具是固定源码 editable 安装并补装所需依赖，环境并非完整生产部署。

## 健康检查与认证

- 在 `OH_SESSION_API_KEYS_0` 设置测试 key 的样本中，`GET /ready` 无凭据返回 200 和 `{"status":"ready"}`；`GET /api/conversations/search` 无 key、错误 key 或仅带 `Authorization: Bearer <key>` 均返回 401，带正确 `X-Session-API-Key` 返回 200 和空页面。`GET /api/conversations/count` 返回数字 0。
- 固定提交的上游 README 将会话 key 描述为 Bearer 认证；但本样本和 `dependencies.py` 的受保护常规 API 检查使用 `X-Session-API-Key`。这是固定版本、上述配置和请求的具体冲突，不推广到所有版本、静态工作区 cookie、WebSocket 或其他鉴权路径。
- 源码将 `/ready`、`/health`、`/alive` 等健康路由与常规 `/api` 认证路由分开；初始化和工作区静态路由还有各自的中间件。代理、跨源、会话 cookie 和密钥轮换尚未在此测试；WebSocket 首帧认证的单独样本见 `WEBSOCKET-OBSERVED.md`。

## 创建、消息和运行

- `POST /api/conversations` 接收 Agent/模型和工作区配置，样本中无初始消息的创建返回 201、会话 ID 及 `idle` 状态。`GET /api/conversations/{id}` 返回该会话；`GET /api/conversations/search` 返回 `{items, next_page_id}`；`GET /api/conversations/count` 返回会话数。身份、配置字段和全部错误条件仍需逐项核验。
- `POST /api/conversations/{id}/events` 提交用户消息，样本中 `run:false` 返回 200、记录用户消息但不访问模型。随后 `POST /api/conversations/{id}/run` 返回 200 并在后台执行；轮询会话直到 `finished`，事件包含 Agent 消息，本机模型收到一条 `/v1/chat/completions` 请求。不要将触发执行的 HTTP 200 当作任务同步完成。
- 创建请求中的 `initial_message` 是不同路径：样本把初始消息设为 `run:false`，创建后仍启动 Agent 并请求模型。源码在有初始消息时先存消息再运行；不要把单独发送消息时的 `run:false` 行为套用到创建请求。此结论仅针对该版本的这一配置。

## 重启与删除

- 在服务正常退出、保留同一临时会话目录后重启，样本可按原 ID 获取会话，事件 ID 列表与重启前一致，计数保留。删除临时会话返回 200，随后读取为 404；两条测试会话均删除后计数为 0。该样本不覆盖进程崩溃、并发写入、恢复正在执行的工具或存储损坏。
- 路由源码还提供暂停、中断、确认策略、事件筛选和 WebSocket 等功能；其参数、认证、重连、并发及错误语义未完整核验，不能因上述结果声称远程协议或服务端已完整重建。

可运行测试见 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `tests/test_openhands_server.py`。该测试只启动和关闭自己的本机进程与 fixture；其临时环境不是安全沙箱。上游样本通过不等于独立重建通过。
