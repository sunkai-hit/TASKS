# AGENT.md

你是 TASKS 仓库的持续执行 Agent。每次被唤起时都按本文件执行。

## 1. 启动顺序

1. 读取本文件。
2. 读取 `queue/tasks.json`。
3. 读取 `state/runtime.json`。
4. 检查是否存在未过期的运行锁。
5. 选择第一个“可执行任务”。
6. 执行任务，并持续保存断点。
7. 在本轮剩余能力允许时继续下一个可执行任务。
8. 更新 `TASKS.md` 为人类可读快照。

## 2. 可执行任务判定

任务必须同时满足：
- status 为 `ready`，或 `in_progress` 且属于可恢复状态；
- 所有 `depends_on` 已为 `done` 或 `skipped`；
- 不处于 `waiting_user`；
- 未超过 `max_attempts`；
- 没有仍有效的外部阻塞条件。

选择顺序：
1. 已经 `in_progress` 的任务优先恢复；
2. priority 按 P0 > P1 > P2 > P3；
3. 同优先级按 order 从小到大；
4. 最后按 created_at 从早到晚。

## 3. 运行锁

开始执行前，将 `state/runtime.json` 写为：
- running = true
- current_task_id = 当前任务
- run_started_at = 当前时间
- heartbeat_at = 当前时间

正常结束时：
- running = false
- current_task_id = null
- last_finished_at = 当前时间

若发现 running=true，但 heartbeat_at 已超过 90 分钟，则视为陈旧锁，可以恢复执行。

## 4. 断点策略

每个任务必须维护：
- `checkpoint.summary`：已完成到哪里；
- `checkpoint.next_action`：下一步具体做什么；
- `checkpoint.last_success_at`；
- `artifacts`：代码提交、文件、PR、页面等成果引用；
- `attempts`：失败重试次数；
- `last_error`：最近一次失败原因。

不要只写“进行中”。断点必须让下一次执行无需聊天上下文即可继续。

## 5. 状态迁移

- backlog：尚未批准进入执行队列。
- ready：可以执行。
- in_progress：已经开始且未结束。
- done：验收条件已满足。
- blocked：由于可自动恢复的外部/技术条件暂时无法继续。
- waiting_user：必须等待用户明确决策、授权或关键输入。
- failed：达到 max_attempts 或确认不可完成。
- skipped：任务已取消或无须执行。

不得因为一次工具超时就标记 failed；应优先保存 checkpoint 并保留 in_progress。

## 6. 自动继续原则

- 一轮不要只做一步；有余力就继续下一步或下一个独立任务。
- 一个任务 blocked/waiting_user 后，继续查找不依赖它的其他 ready 任务。
- 每完成一个明显阶段就落盘，不要等整轮结束再保存。
- 对代码仓库修改，尽量形成可识别的 commit，并把 commit URL/SHA 写入 artifacts。
- 对文档/原型等文件，同样记录最终位置和版本。
- 禁止在没有确认目标仓库/文件时盲目修改。

## 7. 需要用户确认的情况

以下情况默认进入 waiting_user：
- 需求方向存在多个实质不同方案且无法从任务描述推断；
- 需要新账号、权限、密钥或人工验证码；
- 删除大量数据、覆盖正式版本、强推分支等高影响操作；
- 对外发送邮件、公开发布等用户未明确授权的动作；
- 用户明确要求“先确认后执行”的步骤。

普通代码实现细节、样式微调、非破坏性重构，不应频繁打断用户。

## 8. 完成标准

任务只有在 acceptance_criteria 全部满足后才能 done。
如果没有 acceptance_criteria，则先根据 task.description 推导最低可验证完成条件，并写入任务后再执行。

## 9. 输出给用户

只有以下情况主动提醒用户：
- 一个重要任务或里程碑完成；
- 进入 waiting_user；
- 达到 failed；
- 发现会显著改变范围的风险。

没有可执行任务时，不发送无意义提醒。