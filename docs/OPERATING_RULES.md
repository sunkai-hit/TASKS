# TASKS 运行机制

## 一、数据源

`queue/tasks.json` 是唯一权威任务数据源。  
`TASKS.md` 只用于阅读，必须由权威数据同步生成，不可反过来覆盖 JSON。

## 二、任务 ID

格式：`TASK-YYYYMMDD-NNN`。

同一天新任务序号递增，禁止复用已经完成或取消的 ID。

## 三、优先级

- P0：立即处理，阻塞核心交付。
- P1：高优先级，正常应优先执行。
- P2：普通任务。
- P3：优化、低优先级或可延期事项。

## 四、依赖

`depends_on` 保存任务 ID 数组。只有全部依赖任务状态为 `done` 或 `skipped` 时，该任务才可执行。

禁止出现循环依赖。

## 五、断点

断点至少包含三部分：

1. `summary`：已经成功完成的事实。
2. `next_action`：下一次唤起后的第一条具体动作。
3. `last_success_at`：最后一次成功持久化时间。

一个好的 next_action 应类似：
“读取 ibms-smart-park/src/components/TwinScene.vue，继续实现消防设备过滤层，并在完成后运行构建。”

不要写：
“继续开发”。

## 六、成果记录

`artifacts` 可以记录：

```json
{
  "type": "commit",
  "repo": "sunkai-hit/xxx",
  "ref": "abc123",
  "url": "https://github.com/...",
  "description": "完成楼层抽屉交互"
}
```

type 可使用：commit、pr、file、url、document、image、release、note。

## 七、重试

- 工具超时：不计为最终失败；保存断点，保持 in_progress。
- 可重复的技术错误：attempts + 1。
- 达到 max_attempts 后进入 failed。
- 权限、账号、验证码：进入 waiting_user，不消耗 attempts。
- 外部服务暂不可用：进入 blocked，并写清恢复条件。

## 八、任务领取与锁

开始任务前先写 runtime 锁。执行过程中，每次重要落盘都刷新 heartbeat_at。

若锁超过 `stale_lock_minutes` 未更新，可判断上一轮已异常结束并接管。

## 九、一轮执行预算

每轮应优先保证“有可恢复成果”，而不是追求一次做完：

1. 获取任务；
2. 做一个可验证小阶段；
3. 提交代码/文件；
4. 更新 checkpoint；
5. 有余力继续。

如果预判剩余步骤可能超出本轮，应在安全断点提前保存。

## 十、跨项目任务

TASKS 仓库只保存调度状态，不复制其他项目源码。

实际成果写回目标仓库；TASKS 只保存：
- 任务描述；
- 状态；
- commit/PR/文件引用；
- 断点；
- 错误与阻塞。

## 十一、用户指令映射

用户说“加入 TASKS：……”：
- 新增任务；
- 默认 status=ready；
- 默认 priority=P1；
- 自动生成验收条件；
- 若能识别目标仓库则写入 project.repo。

用户说“暂停……”：
- 将任务设为 blocked；
- blocking_reason 写“用户主动暂停”。

用户说“取消……”：
- status=skipped。

用户说“继续……”：
- 不改变优先级；
- 恢复 blocked 中属于用户主动暂停的任务为 ready。

## 十二、完成任务后的动作

一个任务 done 后：
1. 写清最终成果；
2. 清空 blocking_reason / user_input_needed；
3. 更新 TASKS.md；
4. 检查是否有依赖它的 backlog/blocked 任务可以转 ready；
5. 本轮仍有能力则继续下一个任务。