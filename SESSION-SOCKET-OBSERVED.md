# Agent Server 会话 WebSocket：运行样本

来源是 [OpenHands/software-agent-sdk 固定提交](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98) 的 `session_socket.py` 和 `session_protocol.py`。环境、临时目录、本机模拟模型及正常重启方式与 `SERVER-OBSERVED.md` 相同，使用真实 WebSocket 客户端；没有启动 Canvas。固定版本 Canvas 源码使用该端点，而不是 `WEBSOCKET-OBSERVED.md` 的旧事件端点。

## 认证、帧与恢复

- `/sockets/session/{conversation_id}` 在该配置下先接收 JSON 首帧 `{"type":"auth","session_api_key":"<key>"}`；错误 key 关闭码为 `4001`。认证后收到 `sync` 帧；其 `from_seq` 为所请求的恢复游标，`through_seq` 为服务连接时已有日志的上界。值为空时字段可能省略。
- `?after_seq=-1` 请求全部历史：每个持久事件包在 `{"type":"durable","seq":整数,"event":事件}` 中。将正常重启前记录的事件 ID 和序号与重启后全量重放对比，本样本全部一致。以某个已有序号为游标重连，只收到该序号**之后**的历史事件；边界自身不重放。省略 `after_seq` 的短观察窗口没有历史持久帧，但可以收到 `transient` 状态帧。
- 样本通过连接发送普通用户消息，服务产生持久用户消息和模拟模型回复；断开后从消息发送前的游标重连，可再次收到这两条事件，序号与实时收到的一致。发送非法 JSON 会得到 `error` 帧，随后同一连接仍可提交有效消息。测试使用正常断开、正常进程重启和无工具模型，不涵盖崩溃、网络丢包或并发。

## 从源码得到的协议边界

- 除持久帧外，协议还定义不携带持久序号的 `transient`、流式进度的 `item_started`、`delta`、`item_aborted`、以及连接错误 `error`；进度帧不被历史重放。最终持久事件和过程流式文本的关联、丢帧与缓冲行为尚未通过此样本验证。
- 服务器源码说明 `seq` 是持久日志位置；客户端不应把临时帧当成可恢复位置。源码允许无法读取的事件被跳过；有洞、慢消费者、并发实时写入与订阅上限须单独测试，不能据此声称绝不丢事件或完整协议兼容。

可运行测试见 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `tests/test_openhands_server.py` 和 `tests/openhands_session_probe.py`。这是对固定上游的局部观察，不是独立重建或 Canvas UI 验收。
