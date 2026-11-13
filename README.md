<h1 align="center">dsh-better-goal</h1>

<p align="center">给一个 DSH 会话装上长期目标：跨轮次自动推进，随时暂停、恢复，轮数由你控制。</p>

<p align="center"><strong>模型负责推进，你负责方向和边界。</strong></p>

<p align="center">
  <a href="https://www.npmjs.com/package/dsh-better-goal"><img src="https://img.shields.io/npm/v/dsh-better-goal.svg" alt="npm 版本"></a>
  <a href="LICENSE"><img alt="Apache-2.0" src="https://img.shields.io/github/license/omdsh-dev/dsh-better-goal?color=blue"></a>
  <a href="package.json"><img alt="DSH 0.2.2-rc.2" src="https://img.shields.io/badge/DSH-0.2.2--rc.2-4B32C3"></a>
</p>

<p align="center">
  <a href="#它能带来什么">它能带来什么</a> ·
  <a href="#一次目标怎么走">怎么工作</a> ·
  <a href="#快速开始">快速开始</a> ·
  <a href="#命令">命令</a> ·
  <a href="#模型工具">模型工具</a> ·
  <a href="#设置">设置</a> ·
  <a href="#恢复与边界">恢复与边界</a> ·
  <a href="#开发">开发</a>
</p>

## 它能带来什么

- **把「一轮对话」变成「一个目标」**：goal 持久化在会话里，跨轮次保留 objective、已执行轮数和状态，不用每轮重新交代。
- **自动续跑**：会话空闲时，driver 自动排下一轮，直到目标完成、被暂停、达到轮数上限或遇到需要人介入的情况。
- **人始终握着方向盘**：GoalBar 和 `/goal` 命令可以暂停、恢复、就地编辑、完成或清除目标；轮数策略也可以在运行中改。
- **模型有简单统一的入口**：增强实现只暴露一个 `goal` 工具，`get` / `set` / `pause` / `complete` / `resume` 五个动作；模型不需要管理 goal id 或 revision。
- **不是盲目空转**：续跑前会先等子代理结束；从第二轮起，再给上一轮的后台 job 一个有界等待（默认 40 分钟）。job 不会被取消；上一轮结束得太快时，下一轮会提示模型「轮与轮之间没有延迟，要等就在本轮内阻塞等待」。
- **可观测、可恢复**：GoalBar 显示目标、轮数和状态；设置页管理策略；目标历史支持搜索和状态筛选。重开、宿主重启或 fork 后从会话日志恢复目标，再显式恢复续跑。
- **实现可切换**：默认增强实现（本插件的单工具、轮数策略和 driver）；也可以切到原生实现，使用上游的 goal 服务、工具、命令和 driver。两者共享同一份持久格式，切换不丢目标。

适合：一次任务需要很多轮工具调用、跨多轮排查或持续跑测试直到绿；你希望模型自主推进，但自己随时能刹车、改目标或限轮数；会话重开、宿主重启或 fork 后还想接着做。

## 一次目标怎么走

1. **创建**：`/goal <objective>` 或模型调用 `goal set`。目标立即进入 active 并 arm，轮数从 0 开始。
2. **自动续跑**：一轮结束后，driver 在会话安静、没有竞争队列且不在 plan mode 时排下一轮；轮与轮之间没有固定延迟。
3. **等待与提示**：续跑前会先等子代理结束；从第二轮起，再等上一轮的后台 job 最多 `jobWaitSeconds`（默认 2400 秒）。job 不会被取消；第一轮没有上一轮的后台工作，所以不等待。上一轮短于 `spinHintSeconds`（默认 120 秒）时，下一轮会附带空转提示，要求模型把等待放在本轮内完成。
4. **停下来**：模型可以 `pause` 等人，或 `complete`；人会 `/goal pause`、`/goal fail <reason>`、`/goal clear`，或达到有效的轮数上限。暂停/清除/阻塞都会停止自动续跑。
5. **恢复**：`/goal resume` 或模型 `goal resume` 重新 arm，保留 objective 和已执行轮数；在 `agent` 轮数模式下可以在恢复时同时提高上限。
6. **结束与历史**：`complete` 标记完成；`clear` 写 tombstone 并保留历史。已完成或已清除的目标仍可从目标历史查看。

## 快速开始

```sh
dsh plugin --profile <profile> add dsh-better-goal@0.4.4
```

安装、升级或换包后重启对应 profile；只刷新网页不够。

```text
/goal 把 packages/foo 迁移到新 API，验收标准是 pnpm test 全绿
/goal pause
/goal resume
/goal clear
```

也可以随 `dsh-better-dsh` 一起安装；本插件不依赖它，单独安装即可工作。

## 命令

| 输入 | 行为 |
| --- | --- |
| `/goal` | 查看当前目标 |
| `/goal <objective>` | 创建或整体替换目标 |
| `/goal set <objective>` | 整体替换：新 objective、轮数归零、立即启动 |
| `/goal create <objective>` | `set` 的别名 |
| `/goal edit <objective>` | 在原目标上就地编辑；已完成目标会新建一个 |
| `/goal pause` / `/goal off` | 暂停并停止自动续跑 |
| `/goal resume` / `/goal on` | 恢复并重新启动自动续跑 |
| `/goal complete` | 标记完成 |
| `/goal fail <reason>` | 仅人类：记录原因并阻塞目标 |
| `/goal clear` | 清除当前目标，历史保留 |

GoalBar 固定在输入区，显示当前目标、轮数和状态，提供暂停、恢复、编辑、清除和历史入口。

## 模型工具

增强实现只注册一个 `goal` 工具：

| 动作 | 语义 |
| --- | --- |
| `get` | 读取当前目标，没有则返回 null |
| `set` | 全量替换为新目标：新 objective、轮数归零、立即 arm |
| `pause` | 暂停并停止续跑；不存原因，模型在回复里说明需要什么 |
| `complete` | 标记完成；只在当前证据确实覆盖整个目标时使用 |
| `resume` | 恢复现有目标，保留 objective 和已执行轮数 |

工具不接受 `goal_id` 或 `revision`；内部从当前目标派生引用，服务层仍做 revision 校验。`max_goal_rounds` 只在 `agent` 轮数模式下暴露。

原生实现则使用 DSH 上游的 goal 服务、工具、命令和 driver；两种实现不会同时运行。

## 自动续跑与轮数

- 续跑发生在目标轮结束后的安静边界；不会在 plan mode、竞争队列或非空闲 Agent 上抢跑。
- 会等待运行中的子代理结束；从第二轮起，后台 job 最多等待 `jobWaitSeconds`，超时后继续下一轮，但不会取消 job，并在下一轮提示里列出仍在运行的 job。第一轮不等待。
- 轮数上限有三种策略：

| 模式 | 行为 |
| --- | --- |
| `agent`（默认） | 模型可在 `set` / `resume` 时传 `max_goal_rounds`；不传时创建不设策略上限，恢复沿用当前上限 |
| `fixed` | 使用设置里的 `roundLimit`（默认 256） |
| `unlimited` | 不设策略上限（兼容哨兵 `Number.MAX_SAFE_INTEGER`，UI 显示为「无上限」） |

- 达到有效上限、运行时错误、续跑被拒绝或人类 `/goal fail` 都会让目标进入 blocked 并停止续跑。
- `unlimited` 不是数学无限：它是原生 goal/change v1 能接受的最大安全整数；切回原生实现时可能显示为一个很大的有限数。

## 设置

在 Plugins 页的 `dsh-better-goal` 行设置：

| 字段 | 默认 | 作用 | 生效 |
| --- | --- | --- | --- |
| `implementation` | `enhanced` | 增强实现或原生实现 | 保存后重载 profile |
| `nativeDefaultMaxGoalRounds` | `256` | 原生实现的默认轮数上限 | 重载 profile |
| `nativeBlockedAfterConsecutiveRounds` | `3` | 原生实现的阻塞阈值 | 重载 profile |
| `roundLimitMode` | `agent` | 轮数策略：`agent` / `unlimited` / `fixed` | 热更新 |
| `roundLimit` | `256` | `fixed` 模式使用 | 热更新 |
| `spinHintSeconds` | `120` | 上一轮短于该秒数时提示空转；`0` 关闭 | 热更新（仅增强实现） |
| `jobWaitSeconds` | `2400` | 下一轮前等待后台 job 的上限；`0` 表示不等 | 热更新（仅增强实现） |
| `questionTimeoutSeconds` | `120` | 已弃用，仅保留旧配置读取 | 只读 |

`roundLimitMode` / `roundLimit` / `spinHintSeconds` / `jobWaitSeconds` 会在下一次准入时生效，不会重置已执行轮数；`implementation` 和 `native*` 需要重载 profile 才会换装配。

## 恢复与边界

- 持久格式沿用 DSH 原生 `goal/change` v1 与 projection v6；写入走 revision 校验，旧 revision 的修改会被拒绝。
- 重开、宿主重启或 fork 后，从会话日志前缀恢复目标与已执行轮数；activation 是进程内状态，恢复后为 disarmed，必须显式 `resume` 才继续自动轮次。
- 每个会话最多一个当前目标。`clear` 写 tombstone 并保留历史；`complete` 只标记完成，不删除记录。
- fork 只影响新会话，不会改变父会话的目标 activation。
- 它不是 cron：续跑由目标轮结束和会话空闲触发，没有按墙钟时间的调度。
- 它不会取消后台 job，也不会替你决定需要人回答的问题；需要人时应该 `pause` 并说明。
- 暂停不持久化原因；需要可读原因时用人类 `/goal fail <reason>`。
- 提问等待不归本插件管；`questionTimeoutSeconds` 仅用于旧配置读取。

## 开发

```sh
npm test        # 单元 / 组件测试
npm run smoke   # 运行时冒烟（如环境具备真实 DSH host）
```

实现、数据格式和测试说明见 [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)。

## 许可

[Apache License 2.0](LICENSE)。