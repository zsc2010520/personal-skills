---
name: aep-ingestion-troubleshooting
description: 排查 Adobe Experience Platform (AEP) 数据摄入问题：批次失败或部分失败、数据进了数据湖但没进 Profile、流式数据丢失、Data Prep 映射后字段为空、行数对不上。当用户提到 batch 失败、ingestion 报错、数据没进来、dataflow 失败、映射问题时使用。
---

# AEP 数据摄入排查

摄入问题先分清数据停在哪一站：源端、映射（Data Prep）、数据湖、身份图、Profile。每一站的失败原因和查看位置都不同，"批次成功"只代表进了数据湖。

## 先收集信息

- 摄入方式：源连接器（云存储、数据库、CRM 等）、批量 API、流式（HTTP API、Web SDK、Edge）。
- 数据集名称、对应的 schema、是否启用 Profile。
- 失败的批次 ID 或 dataflow run ID、发生时间、界面上显示的错误代码和信息。
- 现象：整批失败、部分行失败、成功但数据不对、成功但 Profile 里没有。
- 最近有没有改过 schema、映射或源端文件格式。

## 排查顺序

### 1. 找到确切的错误

- 在 Monitoring 或数据集的活动页面找到批次，看状态和错误摘要。
- 批次状态的含义：
  - Loading、Loaded、Staging：还在处理，不是失败。
  - Retrying：平台在自动重试临时问题，不需要操作，会有延迟。
  - Stalled：重试用尽，摄入卡住。
  - Failed：终态，配置或数据有问题，修正后重新提交。
  - Abandoned：批次没有在预期时间内完成。
  - Active 或 Success：已进入数据湖，可供下游使用。
- 错误信息过于笼统（例如提示解析、转换或校验数据时出错）时，开启错误诊断后重新摄入，再下载诊断结果。诊断文件在平台上保留的时间有限（约 30 天）。
- 需要具体的失败行时，用 Data Access API 取回失败批次的文件。

### 2. 格式和文件层面

- 批量摄入支持 Parquet 和 JSON。CSV 只支持扁平结构，嵌套数据要先通过映射转成 XDM。
- JSON 默认是单行模式，一行一个对象。多行 JSON 需要在创建批次时声明，并把对象包在数组里。
- 256 MB 及以上的文件用大文件上传 API。
- 文件为空、CSV 表头行为空会直接导致失败。
- 批次要先上传完所有文件再发完成信号，完成之前数据不会出现在数据集里。

### 3. Schema 校验

最常见的失败原因：

- **类型不匹配**：字符串进了整数字段，日期格式不符合要求。日期时间用 ISO 8601 格式。
- **必填字段缺失**。
- **空值**：平台不接受 null 或空字符串。没有值的字段要从 payload 里省略这个键，不要传 null。
- **约束不满足**：枚举值之外的取值、不符合 pattern 或长度限制。
- **字段不存在**：payload 里的路径在 schema 中找不到，常见于租户命名空间写错或层级放错。
- 批量校验只检查 schema 里设了约束的字段，数组内嵌套的字段不校验。没有报错不代表数据是对的。

### 4. Data Prep 映射

- 映射后目标字段为空：源字段名是否完全一致（大小写、空格、特殊字符），计算字段的函数是否返回了 null。
- 源文件结构变了（列名改了、列少了）而映射没更新。
- 用映射界面的预览功能看样例数据的转换结果，再跑整批。
- 类型转换要显式处理，尤其是日期字符串转日期、字符串转数字。

### 5. 数据进了数据湖，但没进 Profile

批次显示成功，Profile 里看不到，按下面检查：

- schema 和数据集是否都启用了 Profile。启用之前摄入的数据不会自动补进 Profile。
- 记录的主身份是否有值。主身份为空的记录进不了 Profile。
- 在 Monitoring 里分别看数据湖、身份图、Profile 三段的记录数，定位是在哪一段减少的。
- Profile 摄入有单条记录大小的限制，超限会导致 Profile 侧失败。
- 身份服务的报错不一定表示摄入失败。例如记录只有一个身份时，身份图没有可建立的关联，会出现身份解析相关的提示，但数据仍然可以进入 Profile。先确认实际结果，再判断是否需要处理。

### 6. 流式摄入

- 流式校验分同步和异步两种。异步模式下请求返回成功，不代表数据通过了校验，失败的记录在 Monitoring 的流式端到端视图里查看。
- 调试时可以用同步校验，让接口直接返回校验错误。
- payload 结构不对（例如缺少 `xdmEntity`）会直接被拒绝。
- 事件数据不支持更新，对已有事件的"更新"会失败。事件的 `_id` 要保证唯一。

### 7. 行数对不上

- 用 Query Service 对比源端行数和数据集行数。写 SQL 时遵循 `aep-query-service-sql` 技能的规则。
- 差异的常见来源：部分行校验失败被丢弃；同一批数据被重复摄入；源端的增量条件漏掉或重复覆盖了时间边界。
- 成功的批次可以查看输入记录数和输出记录数，两者的差值就是被丢弃的行数。

## 修复时的规则

- 数据湖只能追加。数据错了不能改单行，要处理掉错误的批次并重新摄入正确的数据。
- 替换已摄入的批次用 batch replay；撤掉已进入数据湖的批次用 revert；处理中的批次用 abort。不要直接从 Catalog 里删除批次。
- 删除批次、删除数据集、对已启用 Profile 的数据集做操作之前，先向用户确认影响范围。这些操作会影响 Profile、Audience 和下游的激活。
- 完成批次的调用失败后重试是安全的。

## 交付要求

1. 说明数据停在哪一站，给出错误代码和对应的失败样例行。
2. 给出根因和修复步骤，区分"修源数据""改映射""改 schema"三类做法。改 schema 只能增加，不能删除或修改已有字段。
3. 给出重新摄入后的验证方法：批次状态、行数对比、样例 profile 检查。
4. 无法从诊断信息定位原因时，建议用户带上批次 ID 和 dataflow run ID 联系 Adobe 支持。

## 参考

- 批量摄入排错指南 https://experienceleague.adobe.com/en/docs/experience-platform/ingestion/batch/troubleshooting
- 流式摄入排错指南 https://experienceleague.adobe.com/en/docs/experience-platform/ingestion/streaming/troubleshooting
- 获取摄入错误诊断 https://experienceleague.adobe.com/en/docs/experience-platform/ingestion/quality/error-diagnostics
- 取回失败批次 https://experienceleague.adobe.com/en/docs/experience-platform/ingestion/quality/retrieve-failed-batches

产品行为会更新。本文件与官方文档或用户环境的实际表现不一致时，以后者为准，并提醒用户更新本技能。
