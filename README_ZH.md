# zhukong-orchestrator

`zhukong-orchestrator` 是一个给 Codex 使用的技能仓库，用来执行严格的“主控式编排”流程：先规划，再把工作下发给 subagent，安全地等待长任务完成，按验收标准核验结果，并把全过程落到本地审计目录里。

这个仓库不是普通应用项目。它的核心是 `SKILL.md` 里的主控 prompt，以及 `assets/` 目录中随仓库内置的本地 `gxd-subagent-shim`。

## 这个技能解决什么问题

它适合那种“主控只负责调度和验收，真正动手由 subagent 完成”的场景。

核心约束如下：

- 主控不直接改项目文件。
- 主控不直接执行构建、测试或其他有副作用的项目命令。
- 所有执行动作都必须通过仓库内置的 `gxd-subagent-shim`。
- 主控不把 subagent 的原始长日志或中间半成品直接倒给最终用户。
- 每次运行都会写入 `.artifacts/agent_runs/<RUN_ID>/...` 作为追溯依据。

这意味着它不是“多代理随便协作”的松散方案，而是一个强调纪律和证据链的编排器。

## 仓库结构

```text
.
├── README.md
├── README_ZH.md
├── SKILL.md
├── references/
│   ├── agent_v1.1.md
│   └── gxd-subagent-shim.md
└── assets/
    ├── bin/
    │   └── gxd-subagent-shim
    └── gxd-subagent-shim-0.2.3/
        ├── pyproject.toml
        ├── README.md
        └── gxd_subagent_shim/
```

其中最关键的文件是：

- `SKILL.md`：主控技能的真正规则来源。
- `references/agent_v1.1.md`：主控流程和输出契约的参考版文档。
- `references/gxd-subagent-shim.md`：如何正确调用本地 shim 的速查文档。
- `assets/bin/gxd-subagent-shim`：推荐优先使用的 wrapper。
- `assets/gxd-subagent-shim-0.2.3/`：本地打包的 shim 实现，只有 Python 标准库依赖。

## 本地 Shim 是怎么工作的

`assets/bin/gxd-subagent-shim` 这个脚本会先把仓库里自带的 shim 源码目录加入 `PYTHONPATH`，然后执行：

```bash
python -m gxd_subagent_shim ...
```

这样做的目的很明确：

- 避免误用 PATH 里的全局 `gxd-subagent-shim`
- 避免不同机器上的 shim 版本漂移
- 保证这个技能始终绑定仓库内的实现

从代码上看，这个 shim 负责：

- 处理 `create` 和 `resume`
- 从 JSON 输入和 CLI 参数中解析 `task_id`、`step_id`、`run_id`
- 调用后端 CLI，并捕获 stdout/stderr
- 把请求、响应、输出和错误写入 `.artifacts/agent_runs`
- 从输出里提取 `thread_id`、模型名和人类可读文本
- 在需要时压缩 Codex 的事件流输出

根据 `pyproject.toml`，它要求 Python `>=3.9`，运行时没有第三方依赖。

## 编排流程

`SKILL.md` 里定义的是一个强约束的流水线：

1. 从用户任务中提取目标、范围、约束和成功标准。
2. 生成尽量少的串行步骤，默认优先只拆一个 `S1`。
3. 通过本地 shim 创建或续跑 subagent。
4. 按长任务策略等待，不允许因为“等久了”就提前打断。
5. 根据验收标准和 `.artifacts` 里的证据做核验。
6. 如果不达标，就生成结构化 rework JSON 并继续 `resume`。
7. 只有全部完成或明确失败时，主控才对最终用户输出一次结论。

默认设计倾向是：能不拆多步就不拆，把“实现 + 自验证”放进同一个 step 里完成。

## 长任务等待策略

这个仓库最重要的设计点之一，是把“任务很久没结束”和“任务真的卡死”明确区分开。

主控策略是：

- 只要 shim 进程还在运行，就不允许主动中断
- 只有 stdout 和 stderr 连续至少 20 分钟都没有新增内容，才允许判定为卡死
- 单次 `create` 或 `resume` 最长允许跑 60 分钟
- 如果是 runner 或平台自己把进程杀掉，应视为平台超时，而不是直接认定 subagent 失败

同时，给 subagent 的执行模板还要求：

- 超过 5 分钟的命令必须持续输出可见进度
- 安静命令也要加 heartbeat 或开启 verbose/progress

## 审计与追溯目录

shim 会把每次运行写到下面这种结构里：

```text
.artifacts/agent_runs/<RUN_ID>/
├── meta.json
├── events.jsonl
├── index.md
└── steps/
    └── <STEP_ID>/
        └── rounds/
            └── R0/
```

常见文件包括：

- `request_raw.txt`
- `create_request.json`、`resume_request.json` 或 `rework_request.json`
- `shim_response.json`
- `subagent_output.md`
- `shim_stdout.txt`
- `shim_stderr.txt`
- `shim_error.txt`
- `shim_abort.txt`

也就是说，真正可信的执行证据来自 shim 落盘后的文件，而不是某个 agent 口头说“我已经跑过了”。

## 运行要求

要正常使用这个技能，外部环境通常需要具备：

- Python 3.9 或更高版本
- 默认后端所需的 `codex` CLI
- 如果要切到 Claude 后端，则需要 `claude` CLI 和 `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`
- 能够执行仓库里的 `assets/bin/gxd-subagent-shim`

代码里实现的主要环境变量包括：

- `SUBAGENT_BACKEND`
- `SUBAGENT_RUN_ID`
- `SUBAGENT_ARTIFACTS_ROOT`
- `SUBAGENT_SAVE_RAW_IO`
- `SUBAGENT_LEGACY_LOG`
- `SUBAGENT_VALIDATE_INPUT`
- `SUBAGENT_VALIDATE_OUTPUT`
- `SUBAGENT_STDOUT_MODE`
- `SUBAGENT_COMPACT_PROFILE`
- `CODEX_BIN`
- `CLAUDE_BIN`

## 推荐使用方式

这个仓库的定位是“技能”，不是直接给终端用户运行的独立程序。

典型触发场景：

- 用户明确说“主控”“主控模式”“按主控流程”
- 任务需要严格的 plan -> dispatch -> verify
- 任务持续时间长，需要可恢复、可追溯的 subagent 执行

推荐调用方式如下：

```bash
"<SKILL_ROOT>/assets/bin/gxd-subagent-shim" create "<JSON>" --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
"<SKILL_ROOT>/assets/bin/gxd-subagent-shim" resume "<JSON>" <thread_id> --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
```

只有 wrapper 不可用时，才建议直接执行本地源码：

```bash
PYTHONPATH="<SKILL_ROOT>/assets/gxd-subagent-shim-0.2.3" \
python -m gxd_subagent_shim create "<JSON>" --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
```

## 这个仓库的价值在哪里

和“随手起一个 subagent 然后等它回来”相比，这个仓库多了四层硬约束：

- 执行入口被固定到本地 shim
- 每个 step 都有明确的验收标准
- 返工通过结构化 JSON 来表达，而不是口头反馈
- 证据默认写入本地审计目录

如果你需要的是高约束、强追溯、适合长任务的代理编排，这套设计比单纯 prompt 更稳。

## 开发说明

如果要修改这个仓库，建议按职责区分：

- 改主控规则或 prompt：修改 `SKILL.md`
- 改参考说明：修改 `references/`
- 改 wrapper 行为：修改 `assets/bin/gxd-subagent-shim`
- 改 shim 实现：修改 `assets/gxd-subagent-shim-0.2.3/`

## 文档语言

- 英文：`README.md`
- 中文：`README_ZH.md`
