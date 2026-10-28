# 本机会话、工具与确认：运行样本

固定来源为 [OpenHands/software-agent-sdk 提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98)。本样本在 Linux、Python 3.13.16 上，以该提交的 `uv.lock` 安装 `openhands-sdk` 1.58.2，使用临时工作区、临时持久化目录、临时测试凭据及仅本机的 OpenAI 兼容 HTTP fixture。没有调用真实模型或访问用户项目；fixture 连接限制和临时目录不是容器或安全沙箱证明。

## 创建和事件

- 样本通过 `register_tool` 注册自定义 `ToolDefinition`，提供动作参数类型、观察结果类型及执行器；Agent 接收 LLM 配置和工具声明，`LocalConversation` 关联工作区和持久化目录。先 `send_message`，再 `run`。不同语言实现不必采用相同内部类型；若声称兼容此 Python SDK，则需独立验证公开接口。
- 本机模型 fixture 的第一次 `POST /v1/chat/completions` 返回一个工具调用，指定字符串参数；SDK 执行器将该字符串写入临时工作区的固定文件。第二次请求获得最终文本后，样本会话状态为 `FINISHED`。模型服务、HTTP 协议和工具是独立参与者，配方不包含真实模型或凭据。
- 该样本的持久化事件顺序为：`SystemPromptEvent`（`agent`）、`MessageEvent`（`user`）、`ActionEvent`（`agent`，含工具名）、`ObservationEvent`（`environment`，对应工具名）、`MessageEvent`（`agent`）。系统可产生其他状态、事件、多个调用和并发路径，不能把这五个事件视为所有运行的通用顺序。
- 基础事件源码定义 `id`、`timestamp`、`source` 和可选 `parent_id`；动作与观察关联工具调用及动作。字段序列化、树分支、旧格式兼容、回调时序和重复投递仍需进一步提取与实测。

## 参数错误和确认

- 在相同流程中，模型给工具的必填 `text` 参数缺失时，样本出现 `ActionEvent` 后接 `AgentErrorEvent`，执行器未写文件；模型随后返回正常最终文本，会话最终仍为 `FINISHED`。因此 `FINISHED` 不自动等于所有工具调用成功，也不能据此声称用户任务已完成。
- 设置 `AlwaysConfirm()` 后，第一次 `run` 在模型提出动作后进入 `WAITING_FOR_CONFIRMATION`，没有执行器副作用；本样本再次调用 `run` 相当于批准，随后产生工具结果和文件。若调用 `reject_pending_actions` 后继续运行，出现 `UserRejectObservation`，文件不存在；模型仍可继续并结束。此样本仅覆盖单个自定义工具动作，不代表所有内建工具、多动作、超时或并发确认行为。

## 恢复

- 使用相同 agent、工作区、持久化目录及会话 ID 创建新 `LocalConversation` 后，样本恢复原有五个事件；发送后续消息并运行会新增用户及 Agent 消息，第三次模型请求包含后续消息。无论动作正常、参数无效、批准还是拒绝，上述样本均可恢复。
- 源码允许没有持久化目录时使用内存存储；有文件存储时恢复基础状态与事件。存储格式、迁移、损坏、跨进程并发、变更工具集或工作区时的兼容规则尚未在此样本完成验证，不应据此推断。

可运行样本见 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `tests/test_openhands_sdk.py` 及配套 fixture。该样本检查所选 Python SDK 的公开行为和本机模拟模型，既不证明上游所有能力，也不证明从本配方独立重建成功。
