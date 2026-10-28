# 覆盖与证据

- 上游 [software-agent-sdk](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98)，观察提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`，仓库声明 MIT。
- 已读 README、仓库边界、远端会话示例及部分本机会话、事件和确认策略源码。以固定提交的 SDK 1.58.2、Python 3.13.16、临时工作区及本机模拟 HTTP 模型实际运行自定义工具、参数错误、确认、拒绝和恢复样本，见 `CONVERSATION-OBSERVED.md`；并运行上游 `tests/sdk/conversation/local/test_conversation_send_message.py`，启用 `pytest_asyncio` 后 26 项通过。这些是上游结果，不是重建验收。
- 在临时工作区运行固定源码的真实文件编辑器和 subprocess 终端，以及 Agent 通过本机模拟模型顺序调用二者；测试版本、依赖条件、访问边界、软等待和未验证项见 `TOOLS-OBSERVED.md`。未覆盖 tmux、浏览器、全部工具、安全隔离或真实模型。
- 在独立临时目录启动固定源码的 Agent Server，使用本机模拟模型测试受保护 API、会话运行、重启与删除；发现 README Bearer 说明与该样本的实际认证头不同。另通过真实 WebSocket 客户端测试认证、正常重启后的重放、时间边界和实时消息。具体版本和边界见 `SERVER-OBSERVED.md`、`WEBSOCKET-OBSERVED.md`；未覆盖远端运行时或崩溃恢复。
- 用同一隔离服务实测 Canvas 使用的 `/sockets/session`：认证、正常重启前后事件序号、游标续传、非法消息错误帧和发送消息后的重连，见 `SESSION-SOCKET-OBSERVED.md`；这不等于浏览器互操作或重建验收。
- 固定源码 SDK 使用临时 Git 子模块和内置测试 LLM，实测部分项目技能优先级、禁用、提示词披露及直接调用执行器；观察到可执行的内联命令，见 `PROJECT-SKILLS-OBSERVED.md`。该样本直接调用执行器，未验证完整技能协议或安全隔离。
- 固定源码 SDK 通过本机模拟 HTTP 模型实测项目技能工具注册、两次实际调用、未知与禁用技能错误；重复调用的副作用与工具注解不一致，详见 `PROJECT-SKILLS-OBSERVED.md`。该模拟模型和进程内网络限制不证明真实模型、远端运行、安全隔离或独立重建。
- 未完整提取 SDK API、服务端路由、事件语义、工具和技能协议、状态存储、权限与安全、模型错误、隔离、成本、并发或恢复；未独立重建。外部服务与模型前提不在依赖归档中。
