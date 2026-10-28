# Agent Server 事件 WebSocket：运行样本

这是旧的 `/sockets/events` 端点，不是固定版本 Canvas 默认使用的会话端点；后者的序号、帧封装和实测结果见 `SESSION-SOCKET-OBSERVED.md`。

来源和隔离测试环境与 `SERVER-OBSERVED.md` 相同，固定上游提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`。客户端通过回环地址连接服务，使用临时会话、测试密钥和本机模拟模型；先正常停止并重启服务，再订阅已有会话。以下是这组输入的观察，不是完整协议或兼容性声明。

## 连接与认证

- 地址为 `/sockets/events/{conversation_id}`。配置 `OH_SESSION_API_KEYS_0` 且连接未携带旧式认证参数时，连接建立后的第一条客户端 JSON 帧为 `{"type":"auth","session_api_key":"<key>"}`；错误密钥触发 WebSocket 关闭码 `4001`。正确密钥可收到事件。该样本未验证密钥轮换、代理、跨源策略或没有认证配置的部署。
- 固定源码还接受查询参数或请求头中的密钥，但将这些方式标记为过时；它们的优先级和错误分支不由此样本保证。密钥不是配方内容，不要写入日志或持久化为真实凭据。

## 重放与实时消息

- 在会话运行完毕、服务正常重启后，以 `?resend_mode=all` 订阅，首帧认证后能收到此前通过事件搜索读取的全部事件 ID。关闭这个连接后重新连接，并设置 `resend_mode=since` 和 URL 编码的 `after_timestamp`：样本使用最后一条历史时间戳作为边界，收到了边界事件，未在测试观察窗口内收到更早事件。该边界在此固定版本中为包含式；客户端需能够按事件 ID 识别重复事件。
- 在恢复连接上发送用户消息 JSON `{"role":"user","content":"..."}` 后，样本收到对应用户消息事件和模拟模型生成的 Agent 消息事件，模型收到新增请求；因此这条 WebSocket 消息路径会启动 Agent。它不同于 REST 消息中的 `run:false`，不要混用。
- 此样本不证明并发重放时的排序、重复或零遗漏；也未模拟网络中断、正在执行时重启、异常帧、跨主机连接、长时间背压或模型及工具失败。应用需针对这些情况另行设计和测试恢复行为。

可运行样本位于 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `tests/test_openhands_server.py` 和 `tests/openhands_websocket_probe.py`。对上游的局部测试通过，不等于从配方独立重建通过。
