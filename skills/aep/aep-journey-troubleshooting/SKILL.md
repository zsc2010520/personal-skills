---
name: aep-journey-troubleshooting
description: 排查 Adobe Journey Optimizer (AJO) 中 Journey 不触发、profile 没有进入、进入后卡在某个节点、提前退出或消息没有发出的问题，包括事件触发、Audience Qualification 和 Read Audience 三种入口。当用户提到 journey 没触发、没人进入、事件被丢弃、测试模式不通、消息没发出时使用。
---

# AJO Journey 排查

Journey 的问题按"有没有进入、进入后走到哪、消息有没有发出"三段来查。先确定卡在哪一段，再往下细查。很多失败是静默的：接口返回成功，界面没有报错，但 profile 没有进入。

## 先收集信息

- Journey 的名称、版本、状态：草稿、测试模式还是已发布（Live），以及开始和结束日期。
- 入口类型：单一事件（unitary event）、业务事件、Audience Qualification、Read Audience。
- 一个"应该进入"的样例 profile 的身份（命名空间加 ID），以及它触发事件的大致时间。
- 现象：完全没人进入、部分人没进入、进入后停在某个节点、进入了但没收到消息。
- 最近是否改过 journey、事件配置、schema 或 audience。

## 第一段：profile 没有进入

### 通用检查

- Journey 是否处于预期的状态。事件发到了测试模式的 journey 还是 Live 的 journey，要对应。
- 当前时间是否在 journey 的生效日期范围内。不在范围内的事件会被静默丢弃，界面没有报错。测试模式下 profile 不前进，最常见的原因是开始日期设在了未来。
- 重入规则：profile 是否已经在这个 journey 里，是否不允许重入或还在重入等待期内。

### 事件触发的 journey

- **身份**：事件 payload 里的身份字段是否存在、有值，并且和事件配置里选择的命名空间完全一致，区分大小写。命名空间不匹配会静默丢弃：接口返回成功，profile 不进入。
- **事件条件**：条件里的字段名、取值、数据类型是否和实际 payload 完全一致（字符串与整数、大小写）。条件不满足时事件被接收但被丢弃，日志里可能出现 `notSuitableInitialEvent`。
- **payload**：是否包含 event ID，结构是否符合流式摄入的要求。配置事件后要先保存，再从 payload 预览里复制示例。
- **发送端**：URL 是否正确；请求头 `x-gw-ims-org-id` 和 `Content-type: application/json` 是否正确；事件是否真的流式送达了平台。只是批量导入到数据集的事件不会触发 journey。
- 接口本身返回错误时，先按 AEP 流式摄入的排查思路处理。

### Audience Qualification 入口

- 不是每个符合条件的 profile 都会进入。发布前已经在 audience 里的 profile、很快又退出 audience 的 profile，都可能不进入。
- Audience 本身是否按预期在更新。人数不对时先用 `aep-audience-troubleshooting` 技能排查 audience。

### Read Audience 入口

- 读取的是 profile 快照里的成员资格，不是实时状态。Batch audience 要等评估作业完成并生成快照之后才读得到。
- Journey 的计划执行时间要晚于 audience 评估和快照生成的时间。排在两者之间会读到旧数据，表现为 audience 有人但 journey 没人进入。
- 手动触发的 audience 评估不一定更新 journey 读取的快照，以计划评估为准。
- Audience 显示从未评估或评估失败时，问题在 audience，不在 journey。

## 第二段：进入了但没走完

- 用 journey 报告看 profile 停在哪个节点、从哪条路径离开。
- **条件节点**：确认 profile 的数据是否真的满足条件，是否落入了没有后续动作的分支。提前退出通常是条件逻辑的问题。
- **等待节点**：是否只是还没到时间。
- **数据源或自定义动作**：外部接口是否超时、报错或没有响应。测试模式下看测试日志；Live 状态下让管理员直接调用该接口验证。没有配置超时或错误路径时，profile 会在这里结束。
- **事件堆积**：profile 的实例被长时间等待、数据富化或自定义动作重试阻塞时，后续事件会进入堆栈。单个 profile 在一个 journey 版本上最多堆积 10 个事件，超出的被丢弃，丢弃原因是 `maxInstanceStackEventsReached`。

## 第三段：消息没有发出

- 报告里该动作节点是否显示已执行、有没有错误。
- 原生渠道：检查同意和退订状态、抑制列表、频次规则，以及 profile 上对应渠道的地址（邮箱、手机号、推送 token）是否存在。在消息反馈数据集里查该 profile 的状态（如 `sent`、`bounce`）。
- 自定义动作：HTTP 调用成功不代表消息送达。journey 侧确认动作已执行后，到外部系统继续查。

## 用数据验证

有 Query Service 时查 Journey Step Events 数据集（表名通常是 `journey_step_events`，用 `SHOW TABLES` 确认）。写 SQL 时遵循 `aep-query-service-sql` 技能的规则。

```sql
SELECT timestamp,
       _experience.journeyOrchestration.stepEvents.nodeName AS node_name,
       _id
FROM journey_step_events
WHERE _experience.journeyOrchestration.stepEvents.journeyVersionID = '<JOURNEY_VERSION_ID>'
  AND _experience.journeyOrchestration.stepEvents.profileID = '<PROFILE_ID>'
  AND timestamp >= to_timestamp('2026-01-01 00:00:00')
  AND timestamp <  to_timestamp('2026-01-02 00:00:00')
ORDER BY timestamp;
```

- 从未进入 journey 的 profile 在这个数据集里没有记录。查不到记录本身就说明问题在第一段。
- 同一步骤可能出现 profile、实例、节点都相同但 `_id` 不同的多行，这是后端重平衡造成的，实际只发送一次。计数用 `COUNT(DISTINCT _id)`。
- 丢弃原因等其他字段的路径，对照官方 "Examples of queries" 文档确认后再使用。

## 交付要求

1. 明确问题在哪一段、哪个节点，以及证据（报告截图描述、测试日志、查询结果）。
2. 区分已确认的原因和仍是推测的原因。
3. 给出修复步骤和验证方法。优先建议用测试 profile 在测试模式下复现，再改 Live 的 journey。
4. 修改已发布的 journey 通常要新建版本，提醒用户对正在 journey 中的 profile 的影响。不要建议直接停止或删除正在运行的 journey。
5. 出现节点转换异常、持续的静默丢弃且配置检查无误时，建议用户带上 journey 版本 ID、profile ID 和时间点联系 Adobe 支持。

## 参考

- 排查 Live journey 的执行 https://experienceleague.adobe.com/en/docs/journey-optimizer/using/monitor/troubleshooting/troubleshoot-journey/troubleshooting-execution
- 测试或发布前的排查 https://experienceleague.adobe.com/en/docs/journey-optimizer/using/orchestrate-journeys/manage-journey/troubleshooting
- 知识库：Profiles not entering journeys https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-27937

产品行为会更新。本文件与官方文档或用户环境的实际表现不一致时，以后者为准，并提醒用户更新本技能。
