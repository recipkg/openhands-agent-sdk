# 文件编辑器与终端：隔离样本

来源为 [OpenHands/software-agent-sdk 固定提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98) 的 `openhands-tools`。本样本运行 Python 3.13.16、SDK 和工具 1.58.2、Bash、Linux；SDK 按固定锁文件安装，工具包为固定源码的 editable 安装，单独安装 `binaryornot 0.6.0`、`libtmux 0.53.0`、`func-timeout 4.3.5` 和 `tom-swe 1.0.3`。未安装或测试浏览器工具。工作区及其外部对照文件均在同一临时目录，终端指定 `subprocess` 后端；这不是隔离或安全沙箱测试。

## 文件编辑器

- 动作支持 `view`、`create`、`str_replace`、`insert`、`undo_edit`，使用绝对路径。`view_range` 在源码中是从 1 开始的行范围，结束位置 `-1` 表示到文件结尾；相对路径在样本返回错误。各文件编码、二进制、目录与错误边界仍需进一步验证。
- 样本创建三行文件后，再次 `create` 同路径返回错误且不覆盖。`str_replace` 的旧文本出现两次时返回错误并保留原文件；唯一匹配时替换。`insert_line=1` 在第一行之后插入文本；一次 `undo_edit` 恢复上一次插入之前的内容，而保留前一次替换。跨会话、服务重启和多次撤销仍需验证。
- **`workspace_root` 不构成文件访问边界**：样本的无额外限制编辑器既可查看也可修改工作区外、仍属于临时目录的文件。源码显示它用于编辑器当前目录及路径提示；不能承诺其会限制文件系统访问。
- 构造 `FileEditorExecutor` 时设置 `allowed_edits_files` 为单个目标路径，样本可编辑目标文件，修改列表外文件被拒绝，列表外文件仍可通过 `view` 读取。列表为空、链接、竞争更改和其他工具是否受限未穷尽；此配置不是全局访问控制或沙箱。

## 终端

- `TerminalAction` 的 `command` 执行 shell 命令；样本通过 `TerminalExecutor(terminal_type="subprocess", shell_path="/bin/bash")` 在指定工作区启动。连续调用在同一会话中运行：第一次设置环境变量，下一次能读到它。完成后显式关闭会话。默认平台检测、tmux 后端、并发、环境过滤及终端清理细节仍需单独测试。
- 样本 `pwd` 返回工作区路径、退出码 0；`false` 返回退出码 1，但 `is_error` 为 false。使用方需要结合退出码、输出及状态判定命令是否成功，不能只看工具级错误标志。样本中的 JSON 列表字面量被工具拒绝，返回 `is_error=true` 且没有正常退出码；并非所有 shell 输入格式都已测试。
- 样本执行 `sleep 2; printf done`，以 0.3 秒超时首次返回 `exit_code=-1`，进程仍在运行；再传空命令并等待后返回退出码 0 和 `done`。这是软等待的样本，不是进程已终止或超时失败；取消、重置、输入转发和无输出超时均须进一步对照。

## Agent 集成样本

使用仅本机的模拟模型依次返回真实 `file_editor` 创建文件和 `terminal` 读取文件调用，SDK 记录两组动作及观察，终端输出与磁盘内容一致，最终状态为 `FINISHED`。工具和工作区使用一次性目录；模型 fixture 与 Python 级连接限制不能代替操作系统隔离。测试仅覆盖正常的单会话、顺序操作，不覆盖真实模型、多用户、服务器或重建实现。

可运行样本见 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `tests/test_openhands_tools.py`。这些是固定上游环境的结果，不是整个工具包或独立重建验收。
