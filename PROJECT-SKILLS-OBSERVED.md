# 项目技能发现与调用：上游样本

来源为 [software-agent-sdk 固定提交 `46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98`](https://github.com/OpenHands/software-agent-sdk/tree/46b1fa5cb91ada17e6ec938c7e8624a4fa19fa98) 的 `skills/skill.py`、`context/agent_context.py`、`tool/builtins/invoke_skill.py` 与 `skills/execute.py`。实测 Linux、Python 3.13.16、该提交的 `uv.lock` 和 SDK 1.58.2，使用独立 Git 项目、一个本地 Git 子模块与 SDK 内置 `TestLLM`；没有调用外部模型或网络。测试进程的 Python socket 被禁用，但它不是 OS 级沙箱。

## 发现、合并和禁用

`load_project_skills(work_dir)` 的样本使用工作区中的 `.agents/skills/<name>/SKILL.md`，也能在工作区属于 Git 子目录时找到仓库根目录的技能。样本中，工作区和仓库根目录存在同名 `shared` 时，以工作区版本为准；仓库根目录的 `.agents/skills/shared` 与 `.openhands/skills/shared` 同名时，前者优先。源码还列出 `.openhands/microagents` 及其他第三方指导文件，但这些位置、更多冲突和错误条件未在本样本完整验证。

将独立仓库作为固定提交 Git 子模块放在根目录 `.agents/skills/subskill` 后，SDK 能发现其中的 `SKILL.md`，同时子模块中的 `scripts/fixture.sh` 文件仍在原处；加载样本没有运行此脚本。这只说明本例发现与文件保留，不证明任意嵌套子模块、符号链接或插件都支持。

本地会话构造时，`AgentContext(load_project_skills=True)` 的样本在初始化时将项目技能合并到 Agent 上下文。项目技能与显式提供的同名技能冲突时，样本使用项目版本，保留其他显式技能；`disabled_skills` 中的名字被排除。设置 `load_project_skills=False` 时，样本不加载项目技能，原有显式技能仍保留。工作区 `AGENTS.md` 的正文进入本样本系统提示词；这不说明任何输入有权覆盖用户或宿主规则。

## 渐进披露和调用

没有触发器的 AgentSkills 格式技能，在本样本系统提示词的可用技能列表中展示名称与描述；正文和 `SKILL.md` 文件路径不在列表中。提示词要求通过 `invoke_skill` 调用。该固定提交中 `Skill` 的类注释仍声称列表包含位置，但实际 `to_prompt` 实现明确省略位置；以本次生成提示词和该实现为准，不能把注释中的位置当作已观察输出。

对样本加载的 `subskill`，直接调用 SDK 的 `InvokeSkillExecutor` 返回技能正文，并在结果中附带该技能目录信息；会话的 `invoked_skills` 包含该名称。此测试直接调用执行器，未通过真实模型发起工具调用，也未验证全部工具注册或远端协议。

**调用时有执行行为：**本样本技能正文中的 `!` 后接反引号命令在调用时由 shell 执行，输出被嵌入返回正文，工作目录为会话工作区；代码围栏中的相同写法没有执行。样本中的内联命令只输出固定文本并创建临时标记；Git 子模块中的 `scripts/fixture.sh` 没有自动执行。源码显示该渲染使用进程权限，不能把技能来源当作安全沙箱；未测试所有转义、失败、超时或其他触发器。安装、发现、列出和调用是不同阶段，不能假设调用只读取 Markdown。

可用 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `test_openhands_sdk.py` 指定 `RECIPKG_TEST_OPENHANDS_PYTHON` 执行；测试建立临时 Git 仓库、本地子模块、模拟 LLM 和工作区。恢复、并发、不同工具集、网络权限、远端工作区、技能文件变化后的刷新以及独立重建仍未验证。

## 模型实际发起调用

同一固定源码和锁文件在 Linux / Python 3.13.16 上运行；临时工作区含可调用的 `alpha` 与 frontmatter 设置 `disable-model-invocation: true` 的 `hidden`，通过只允许本机端口的模拟 OpenAI 兼容 HTTP 服务返回工具调用。Python socket 被限制到该端口，并非操作系统安全沙箱，也没有使用真实模型。

在 `AgentContext(load_project_skills=True)` 和无显式工具的样本中，首个模型请求包含 `invoke_skill` 工具；提示上下文包含 `alpha` 简介但不含正文，也不含禁用技能的简介或正文。模拟模型依次请求两次 `alpha`、一个未知名称和 `hidden`：前两次工具结果包含正文与内联命令输出，未知和禁用名称分别返回错误观察。两次相同调用让内联命令各执行一次；会话 `invoked_skills` 仍只有一个 `alpha`，该列表的去重不表示副作用被去重。真实模型、远端执行、会话恢复后的行为及其他触发条件仍未验证。

源码中 `InvokeSkillTool.create()` 的工具注解同时标记 `readOnlyHint=true` 和 `idempotentHint=true`，而本机样本的内联命令有写文件副作用且重复运行。`tool.py` 的安全风险预测路径会检查 `readOnlyHint`；这是源码观察，不等于本样本测试了确认策略或全部安全机制。不能仅依据这些注解认定技能调用只读、幂等或处于沙箱。
