# TASKS — AI 持续任务调度中心

这个仓库不是普通项目仓库，而是 ChatGPT / AI Agent 的“长期任务队列 + 断点记录中心”。

目标：把跨多个 GitHub 项目、文档、原型、测试和交付物的长任务拆成可恢复的子任务，让每次执行都能从上一次保存的状态继续，而不是依赖聊天上下文。

## 核心原则

1. **GitHub 是事实来源**：任务、状态、断点、结果必须落盘。
2. **一次执行尽量多做**：每轮从第一个可执行任务开始，在允许范围内连续推进多个任务。
3. **先保存再继续**：每完成一个关键步骤立即写回状态。
4. **遇阻不死锁**：单个任务阻塞时，记录原因并继续其他不依赖它的任务。
5. **需要用户决策才等待**：只有业务方向、账号授权、破坏性操作等确实不能自行判断时，标记 waiting_user。
6. **幂等执行**：重复唤起不能重复创建相同成果、重复提交或重复执行危险操作。

## 仓库结构

- `AGENT.md`：每次自动执行必须首先读取的总规则。
- `TASKS.md`：给人看的任务看板。
- `queue/tasks.json`：机器可读的权威任务队列。
- `state/runtime.json`：当前运行状态、锁和最近断点。
- `templates/task-template.json`：新增任务模板。
- `docs/OPERATING_RULES.md`：完整状态机和执行规则。
- `docs/AUTOMATION_PROMPT.md`：自动唤起时使用的标准 Prompt。
- `logs/`：按日期保存必要的执行日志（仅记录重要事件，不记录冗长思考过程）。

## 推荐工作方式

新增任务时，优先写入 `queue/tasks.json`，并同步刷新 `TASKS.md`。任务可以指向其他 GitHub 仓库，例如：

- `sunkai-hit/ibms-smart-park`
- `sunkai-hit/Edge-Gateway`
- `sunkai-hit/Web-Presentation`

任务之间通过 `depends_on` 建立依赖。

## 状态

`backlog` → `ready` → `in_progress` → `done`

异常分支：
- `blocked`：技术或外部依赖阻塞，但未来可自动重试。
- `waiting_user`：必须由用户做决定、授权或补充输入。
- `failed`：多次重试后仍失败。
- `skipped`：明确取消或不再需要。

## 第一条执行规则

任何自动执行器被唤起后，都必须先读 `AGENT.md`，再读 `queue/tasks.json` 和 `state/runtime.json`，禁止直接凭历史聊天记忆继续。