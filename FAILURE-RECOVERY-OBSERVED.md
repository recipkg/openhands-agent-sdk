# 模拟模型认证失败与会话恢复

观察来源为 [software-agent-sdk 固定提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98)，Linux、Python 3.13.16，按该提交 `uv.lock` 安装的 SDK 1.58.2。临时工作区和独立持久化目录使用本机 OpenAI 兼容 HTTP 模拟模型，凭据与错误消息为测试字符串；进程内 socket 限制只允许连接该回环服务，不构成操作系统沙箱。`num_retries=0`，不测试自动重试。

通过 [standard-library-verification](https://github.com/recipkg/standard-library-verification/blob/main/tests/openhands_failure_probe.py) 的真实 `LocalConversation` 执行两种场景：

1. 第一次模型请求返回 HTTP 401 和 `authentication_error`。`run()` 抛 `ConversationRunError`，其原因是 `LLMAuthenticationError`；状态为 `ERROR`，事件中有一条同代码的 `ConversationErrorEvent`，用户可见详情说明密钥无效或过期，未包含测试密钥或原始测试错误消息。此时工具没有执行、标记文件不存在。关闭并以同一会话 ID 从磁盘恢复后，事件类型和 `ERROR` 状态仍在；模拟模型恢复并返回最终消息，调用 `run()` 转为 `FINISHED`，没有产生工具副作用。
2. 第一次模型请求返回一次工具调用；自定义工具将 `once` 追加至隔离文件并返回观察，第二次请求收到同样的 HTTP 401。错误出现时文件只有一行，事件序列已有一个动作、一个工具观察和认证错误。关闭并从同一 ID 恢复后再次运行，模拟模型返回最终消息；模型恢复请求中有该工具调用 ID 对应的观察内容，会话结束时仍只有一个动作事件和一行文件内容。该用例共发生三次模型 HTTP 请求，恢复步骤没有新增用户消息。

以上仅说明在测试模型按上述固定顺序响应、进程正常关闭后重载的行为。测试不会让模型在恢复后重新发出相同工具调用，因此不能证明工具调用全局去重或恰好执行一次；未测失败发生在动作与观察持久化之间、进程异常终止、其他 HTTP 错误、重试、并发、真实模型、服务端 UI 或全部日志中的凭据保护。不得将用户可见错误字段的样本推广为完整秘密泄露审计。
