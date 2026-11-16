---
name: aep-audience-troubleshooting
description: 排查 Adobe Experience Platform (AEP / Real-Time CDP) 中 Audience 人数为 0、人数偏少或偏多、预估值与实际值不一致、某个 profile 该进没进或该出没出的问题。当用户提到 audience 或 segment 人数不对、没有人群、人群没更新、评估方式（batch、streaming、edge）相关疑问时使用。
---

# AEP Audience 人数排查

人数不对通常不是规则写错这一种原因。按下面的顺序逐层排除，每一步都要拿到证据再往下走，不要一上来就改规则。

## 先收集信息

缺少就先问用户：

- Audience 的名称和 ID、评估方式（Batch、Streaming、Edge）、生命周期状态（是否已发布）。
- 创建时间和最近一次修改时间；规则最近有没有改过。
- 期望人数和实际人数分别是多少，期望值的依据是什么。
- 使用的 merge policy。
- 规则用到的字段来自哪些数据集，这些数据集是批量还是流式摄入。
- 一个"应该在人群里"的样例 profile 的身份（命名空间加 ID）。排查时样例比总数有用得多。

## 排查顺序

### 1. 评估有没有跑完

- **Batch**：发布不会立即触发评估，要等下一次计划的批量评估完成，首次结果可能要等到 24 小时。在 Audience 详情或 Monitoring 里看最近一次评估作业的时间和状态。显示从未运行或失败时，问题在评估作业，不在规则。
- **Streaming**：成员资格通常在几分钟内更新，但只有该 profile 收到流式更新时才会触发评估。批量摄入的数据不会触发流式评估。
- **预估值不是实际值**。创建时看到的预估来自 profile 抽样作业，和评估后的实际人数不一致是正常现象，以评估结果为准。

### 2. 评估方式是不是你以为的那种

- 规则被修改后如果不再满足 streaming 的条件，会自动从 Streaming 降为 Batch，界面上不一定有明显提示。
- Streaming 的资格条件在 2025 年 5 月有过调整，大意是：事件类规则需要限制在 24 小时时间窗内；单个事件不带时间窗、或时间窗超过 24 小时的，新建或修改后按 Batch 评估；同一个定义里混用批量数据源和流式数据源的也按 Batch 评估。
- 需要同时用批量数据和流式数据时，把批量部分单独建成一个 batch audience，再在 streaming audience 里引用它（segment of segments）。
- 资格规则会更新，判断前让用户对照当前的官方 streaming segmentation 文档，不要凭记忆下结论。

### 3. 数据有没有进 Profile

- 规则用到的字段所在的 schema 和数据集是否都启用了 Profile。先启用 schema，再启用数据集，两者缺一不可。数据集在启用之前摄入的数据不会自动进入 Profile。
- 数据集最近的批次是否成功，有没有失败或部分失败的批次。
- 事件数据集是否设置了 Experience Event 过期，过期的事件不再参与评估。
- 用样例 profile 在 Profile 查看界面确认：这个人存在吗？规则用到的属性和事件在他身上看得到吗？

### 4. 身份和 merge policy

- 样例记录的主身份是否有值、命名空间是否正确。主身份为空的记录进不了 Profile。
- 期望"同一个人"的多条记录是否真的被合并了。看身份图，确认它们之间有共同身份连接。
- Audience 用的 merge policy 是否和预期一致。不同 merge policy 下属性的合并结果不同，人数也会不同。Streaming 和 Edge 评估对 merge policy 有额外要求（Active-on-Edge），以当前文档为准。
- 多个人共用同一个身份值（例如共用邮箱、测试账号）会把多个 profile 合并成一个，导致人数偏少。

### 5. 规则本身

前四步都没问题再看规则：

- 字段路径是否选对。同名字段可能存在于不同的 field group 或租户命名空间下。
- 数据类型和取值：大小写、前后空格、布尔值与字符串 `'true'`、数字与字符串。
- 时间窗是相对时间还是绝对时间，按 UTC 计算。
- 事件条件的"至少一次"与"全部"、"包含"与"排除"是否写反。排除条件遇到字段缺失时的行为要特别确认。
- AND 与 OR 的分组是否符合本意。把规则拆成单个条件分别建临时 audience 或用预估看人数，定位是哪个条件把人筛掉了。

### 6. 用数据验证

有 Query Service 时，用 profile 快照导出数据集核对。表名和 audience ID 需要替换，快照表名用 `SHOW TABLES` 查找（通常以 `profile_snapshot_export_` 开头，按 merge policy 区分）。写 SQL 时遵循 `aep-query-service-sql` 技能的规则。

```sql
SELECT COUNT(*) AS qualified_profiles
FROM profile_snapshot_export_xxx
WHERE segmentMembership['ups']['<AUDIENCE_ID>'].status = 'realized';
```

- 快照是定时生成的，反映的是上一次快照时的状态，不是实时状态。先确认快照的生成时间。
- 字段结构和状态取值（如 `realized`、`exited`）需要在用户环境里核验后再下结论。

## 交付要求

1. 明确写出结论属于哪一层：评估未完成、评估方式、数据未进 Profile、身份或 merge policy、规则逻辑。
2. 列出支持结论的证据（作业时间、样例 profile 的实际数据、查询结果），以及还没验证的假设。
3. 给出修复建议和验证方法，并说明生效需要多久（例如等下一次批量评估）。
4. 修改已被 Journey 或 Destination 使用的 audience 之前，先提醒用户评估影响。不要建议删除重建正在使用的 audience。
5. 排除了前面所有原因仍无法解释，或评估作业持续失败时，建议用户带上 audience ID 和作业信息联系 Adobe 支持。

## 参考

- Segmentation Service FAQ https://experienceleague.adobe.com/en/docs/experience-platform/segmentation/faq
- Streaming segmentation 指南 https://experienceleague.adobe.com/en/docs/experience-platform/segmentation/api/streaming-segmentation
- 评估资格条件更新 https://experienceleague.adobe.com/en/docs/experience-platform/segmentation/eligibility-criteria-update
- Profile 启用规划 https://experienceleague.adobe.com/en/docs/experience-platform/xdm/schema/profile-enablement-planning

产品行为会更新。本文件与官方文档或用户环境的实际表现不一致时，以后者为准，并提醒用户更新本技能。
