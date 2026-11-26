---
name: "orch-round"
description: "herdr 多模型编排。用户要求启动、开始编排时，启动外层循环；被外层循环拉起、要求跑一轮编排时，对齐状态、验收 worker、合并、按路由先验派活、写交接后退出。"
---

# orch-round

本 skill 的基目录（`<skill-base-dir>`）是装着这份 SKILL.md 的文件夹，里面有 `routing.yaml`、`scripts/orch-loop` 和 `scripts/agent-quota`。基目录是只读的同步缓存，修改在 claude.ai 的 skill 里做。

两种进入方式，按触发场景二选一：

- 用户在自己的会话里要求启动或开始编排：执行「启动」，然后结束。
- 外层循环用「跑一轮编排」拉起你（你的终端名是 `orch`）：执行下面的本轮流程。

## 启动

1. 确认 `test -n "${HC_TERMINAL_ID:-}"`，并且当前目录是项目根目录。`.orch/GOAL.md` 不存在时，请用户先写目标，然后停下。
2. 检查循环是否已在运行：`.orch/loop-terminal` 存在、且 `hc get "$(cat .orch/loop-terminal)" --json` 显示该终端还在时，告诉用户循环已在运行，然后停下。
3. 启动循环。GOAL.md 里列了验收命令时，把它们写成放行规则放进 `ORCH_ALLOWED_TOOLS`，例如 `'Bash(cargo test:*) Bash(npm test:*)'`：
   ```
   hc run --bg --name loop --cwd "$PWD" --no-notify --label orch=loop -- \
     env ORCH_ALLOWED_TOOLS='<放行规则>' bash <skill-base-dir>/scripts/orch-loop
   ```
   用 `bash` 调用，不依赖脚本的可执行位。
4. `hc get loop --json`，确认终端在运行。告诉用户循环已启动、在 Herdr 里 attach `loop` 可以看到进度，然后停下。你只负责启动，之后的每一轮由循环拉起新的会话。

完成标准：`loop` 终端在运行，并且已向用户报告。

## 本轮流程

你是本轮主控，固定为 Claude Opus 5.5、effort high。你只跑**一轮**：处理本轮的事，写好交接，然后停下。你运行在外层循环（`orch-loop`）用 `hc run` 启动的终端 `orch` 里，停下后它会存档你的屏幕并结束会话。业务代码由 worker 写，你负责验收、调度和记录。等待 worker 结束、启动下一轮，由外层循环负责。

Herdr 应用的命令是 `hc`（herdr-core），不是 `herdr`。开始前确认 `test -n "${HC_TERMINAL_ID:-}"`，运行 `hc --skill` 并按它的规则操作。地址和状态一律从 JSON 输出里读取；只 kill 和 rm 带 `orch=worker` 标签的终端。

## 文件

| 文件 | 内容 | 谁写 |
|---|---|---|
| `.orch/GOAL.md` | 用户的目标和约束 | 用户 |
| `.orch/RFC.md` | 设计、全局验收命令、ticket 定义 | 主控：第 0 轮建立，之后只在拆分或新增 ticket 时修改 |
| `.orch/next-round.md` | 当前完整状态和下一轮要做的事 | 主控，每轮整份覆盖 |
| `.orch/ledger.jsonl` | 每次派活、验收、合并的记录 | 主控，只追加 |
| `.orch/handoff/<ticket>-<role>.md` | worker 交回的结果 | worker |
| `.orch/STOP` | 文件存在时外层循环停止 | 主控 |
| `.orch/next-wake` | 没有 worker 在跑时，外层循环睡到这个 ISO 8601 UTC 时间再开下一轮 | 主控 |
| `.orch/loop-terminal` | 外层循环所在终端的地址，启动 worker 时作为 `--parent` | 外层循环 |
| `<skill-base-dir>/routing.yaml` | 路由配置：硬约束 + 参考先验 | 用户（在 claude.ai 里修改 skill） |

## 第 0 轮（`.orch/RFC.md` 不存在时）

1. 读 `.orch/GOAL.md` 和仓库。GOAL.md 不存在时：写 `.orch/STOP`，在 next-round.md 里说明缺少目标，停下。
2. 写 `.orch/RFC.md`：目标、全局验收命令（可执行，退出码 0 即通过）、ticket 列表。每张 ticket 包含：
   - `id`：小写字母、数字和 `-`
   - `title`
   - `domain`：routing.yaml 里的领域之一
   - `tier`：`normal` 或 `hard`
   - `depends_on`
   - `owned_paths`：这张 ticket 允许修改的路径；没有依赖关系的 ticket 之间不能重叠
   - `acceptance`：可执行命令，退出码 0 即通过
   - `needs_tests`：`true` 或 `false`
3. 写 next-round.md，所有 ticket 状态为 `pending`。
4. 把当前 UTC 时间写进 `.orch/next-wake`，让外层循环立刻用新会话开始第 1 轮。停下。

完成标准：每张 ticket 都有可执行的 `acceptance`，并且 `owned_paths` 互不冲突。

## 常规轮

### 1. 对齐状态

读 next-round.md 和 RFC.md，运行 `hc ls --all --label orch=worker --json` 和 `bash <skill-base-dir>/scripts/agent-quota --json`。把 next-round.md 里记录的每个 worker 和 hc 的实际状态（`state`、`status.state`、`labels`）逐一核对，不一致时以 hc 为准，并把差异写进本轮交接。

完成标准：每个仍在运行的 worker，都能对应到它的 ticket、角色和当前状态。

### 2. 处理已结束的 worker

对每个 `status.state` 为 `idle` 或 `done`、或已经退出（`state: finished`）的 worker：

1. 读它的 handoff 文件，再在它的 worktree 里亲自运行 ticket 的 `acceptance`。是否通过以退出码为准，不以 worker 的自述为准。
2. 往 ledger 追加一行。
3. 按结果推进：
   - 实现通过：`needs_tests` 为 true 时派 `tests` 角色，否则派 `review` 角色。
   - tests 通过：派 `review` 角色。
   - review 没有提出问题：状态改为 `ready-to-merge`。
   - 同一 ticket 第 1 次失败：把失败原因和 `file:line` 锚点写进新的 prompt，交还原来的 worker 修复。
   - 同一 ticket 第 2 次失败：在 RFC.md 里把它拆成更小的 ticket，改派给不同的候选；拆不开就 park。
4. 这个角色结束后，先 `hc kill <t>` 再 `hc rm <t>`。

对每个处于 `blocked` 或 `error` 的 worker：`hc read <t> --lines 100` 读屏幕，把它在问什么写进交接，然后 park。审批和提问由用户在 Herdr 里 attach 后回答。

**park** 的意思是：`hc label <t> parked=1`，不再驱动这个 worker，保留它的终端和 worktree，在 next-round.md 的「需要用户」一节写明原因。用户处理完后会去掉标签（`hc label <t> parked=`），之后的轮次照常验收它。

完成标准：每个已结束或 blocked 的 worker 都有了明确去向。

### 3. 合并

`ready-to-merge` 的 ticket 一次合并一个到主分支，每合并一个就跑一次全局验收。出现冲突或验收失败时，把 ticket 退回给实现 worker，状态改回 `implementing`。合并成功后删除它的 worktree。

### 4. 派活

对依赖已满足、尚未分配的 ticket，以及上一步产生的 tests、review 角色，按「选模型」选出候选，然后：

1. `git worktree add .orch/wt/<ticket> -b orch/<ticket>`。tests 和 review 角色复用实现的 worktree。
2. 启动 worker。`<command>` 从 routing.yaml 的 `harnesses` 取，代入 `{model}`、`{effort}` 和 `{prompt}`（worker prompt）：
   ```
   hc run --bg --name <ticket>-<role> --cwd <worktree> \
     --parent "$(cat .orch/loop-terminal)" --outlive-parent --no-notify \
     --label orch=worker --label ticket=<id> --label role=<role> -- <command>
   ```
   `--parent` 把 worker 挂在外层循环下面，循环才能等它；`--outlive-parent` 让它在你退出后继续运行；`--no-notify` 让 hc 不往循环所在的 shell 里打字。
3. `hc get <ticket>-<role> --json`，确认 `first_message.sent` 为 true。为 false 时先 `hc read` 看屏幕（可能卡在信任对话框），读过再决定是否重发；处理不了就 park。
4. 往 ledger 追加一行，`event` 为 `dispatched`。

没有合格候选时，ticket 保持 `pending`，在 next-round.md 写明原因，例如"agy/gemini 已满员，codex 周额度超速"。

完成标准：每张可派的 ticket 要么已经派出，要么写明了不能派的原因。

### 5. 交接并结束

1. 按下面的结构整份覆盖 next-round.md。
2. 没有在跑、且没有 park 的 worker，但还有 ticket 只是因为配额或满员暂时派不出去时：把这些组 `blocked_until` 里最早的时间写进 `.orch/next-wake`；没有 `blocked_until` 就写 30 分钟后。剩下的 ticket 全部是 `parked` 时，什么都不写，外层循环会停下来等用户。
3. 所有 ticket 都已 `merged` 并且全局验收通过时，写 `.orch/STOP`。
4. 停下，等外层循环关闭会话。

## 选模型

routing.yaml 里有两类内容，约束力不同。

**硬约束：每次都遵守**

- 只用 `allowed_models` 里的模型。
- `agent-quota --json` 里 `available: false` 的组不派。
- 组内正在运行的 worker 数达到 `max_workers_per_group` 时不派。
- Claude 周窗口的 `used_pct` 达到 `claude_worker_cutoff_pct` 后，不再派 Claude worker，剩余额度留给主控。
- effort 在 `effort_floor` 和 `effort_cap` 之间。
- tests 和 review 角色的模型厂商（见 `vendors`），必须与这张 ticket 的实现者不同。
- Astra 的 `only_if_weekly_pace_below` 门槛。超速倍数 = `(used_pct / 100) / (1 - (resets_at - as_of) / window_s)`，用周窗口的数据，在 shell 里算，不要心算。

**参考先验：默认照做，有理由时可以偏离**

`domains` 里的候选顺序、每个候选的 effort、`hard_upgrade` 选的模型，都是参考先验。它们来自 2026-11 的公开基准和用户偏好，基准测的是别人的任务和 harness，不是这个仓库。所以：

- 默认按列表顺序，取第一个通过全部硬约束的候选；所有领域都通不过时（`bulk` 除外），再试 `last_resort`。
- `tier: hard` 的 ticket，选中某个 harness 后，换成该 harness 在 `hard_upgrade` 里的模型和 effort。
- 以下情况应该偏离：ticket 的实际性质和它的 domain 标签不符；ledger 显示该候选最近 5 次在同领域里失败 2 次以上；同一 ticket 第 2 次失败后改派。
- 偏离时，在 ledger 的 `deviation` 字段写明理由。说不出理由，就按顺序来。

## worker prompt

每个 worker 都要拿到一份自包含的 prompt，不依赖任何 harness 自带的 skill：

```
你在 worktree <path> 中处理 ticket <id>：<title>。
角色：<implement | tests：为这张 ticket 的实现写测试，不修改实现代码 | review：审查 <base>..HEAD 的改动，只报告问题，不修改代码>
只允许修改这些路径：<owned_paths>
设计背景：<从 RFC.md 摘出的相关段落>
<如果是返工：上次失败的原因和 file:line 锚点>
完成后运行：<acceptance>
把结果写到 <仓库绝对路径>/.orch/handoff/<id>-<role>.md，格式如下：
## 改了什么
## 没做完的
## 问题（每条附 file:line）
写完后停止，不要开始其他工作。结果只写进上面的文件：不要用 hc send 汇报，也不要启动其他 agent。
```

## next-round.md 结构

```
# Round <n> → <n+1>
## Tickets
| id | status | role | worker | group/model/effort | attempts |
## Live workers
<终端名>, <地址，如 t7>, <worktree>, <ticket>, <role>
## 下一轮要做的事
## 需要用户
```

ticket 状态取值：`pending`、`implementing`、`testing`、`reviewing`、`ready-to-merge`、`merged`、`parked`。

## ledger 每行格式

```json
{"ts":"...","round":3,"ticket":"api-auth","domain":"backend","tier":"normal","role":"implement","harness":"agy","group":"gemini","model":"gemini-4-neon","effort":"high","event":"dispatched|passed|failed|merged|parked","attempt":1,"issues":0,"deviation":null}
```