---
name: aep-query-service-sql
description: 编写、审查和修复 Adobe Experience Platform (AEP) Query Service 的 SQL。适用于查询 XDM 数据集（ExperienceEvent、Profile 快照等）、访问嵌套字段和 identityMap、去重、用 CTAS/INSERT INTO 写回数据集、增量加载，以及排查 Query Service 报错或超时。当用户提到 AEP 查询、Query Service、Data Distiller、XDM 字段查询时使用。
---

# AEP Query Service SQL

Query Service 对外是 PostgreSQL 协议，执行引擎是 Spark SQL，数据是嵌套的 XDM 结构，数据湖只能追加。按普通 PostgreSQL 或 MySQL 的习惯写出来的 SQL 经常跑不通。写之前先读完本文件；需要完整模板时读 [常用查询模板](references/patterns.md)。

## 开始前先确认

缺少下面任何一项就先问用户，不要猜表名和字段路径：

- 表名：数据集在 Query Service 中的表名（不是数据集的显示名称）。可用 `SHOW TABLES` 查看。
- 字段路径：完整的 XDM 路径，包括租户命名空间（形如 `_tenantid`，下文统一写作 `_tenant`）。可用 `\d 表名` 逐层查看。
- 时间范围：事件类数据集必须有时间范围。
- 用途：只是查看结果，还是要把结果写成数据集。两者的写法和限制不同。
- 写回时的目标 schema 名称，以及是否要进入 Profile。

## 硬性规则

1. **只能追加**。不支持 `UPDATE`、`DELETE`、`MERGE INTO`、`INSERT OVERWRITE`。需要"修改"数据时，写新数据集或追加新记录来表达最新状态。
2. **事件表必须按 `timestamp` 过滤**，用半开区间，日期带上时间部分：
   ```sql
   WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
     AND timestamp <  to_timestamp('2026-01-02 00:00:00')
   ```
3. **不要 `SELECT *`**。按列存储对待，只选需要的字段。
4. **所有时间都是 UTC**。按本地时间筛选或展示时显式转换：`from_utc_timestamp(timestamp, 'Asia/Shanghai')`、`to_utc_timestamp('2026-01-01 00:00:00', 'Asia/Shanghai')`。
5. **交互式查询有上限**：10 分钟超时，最多返回 50,000 行，最多 5 个并发。探索时加 `LIMIT`；数据量大就改成 CTAS 批处理，不要靠去掉限制硬跑。
6. **函数用 Spark SQL 的**，不是 PostgreSQL 的。例如用 `date_format`、`explode`、`collect_list`、`struct`，不要用 `to_char`、`unnest`、`array_agg`、`jsonb_*`。不确定某个函数是否支持时明说，并让用户先小范围试跑。
7. **用 `NOT EXISTS` 或 `LEFT JOIN ... IS NULL` 代替 `NOT IN`**，后者遇到 NULL 结果不可靠且慢。
8. **不支持自定义函数（UDF）**。

## 嵌套字段与引号

| 场景 | 写法 |
| --- | --- |
| 访问嵌套字段 | 点号 `web.webPageDetails.name`，或方括号 `web['webPageDetails']['name']` |
| 字段名是保留字（order、group 等） | 点号写法加反引号：``commerce.`order`.purchaseID``；方括号写法不需要：`commerce['order']` |
| 字段名以数字开头 | 反引号：``commerce.`30_day_value` `` |
| 字符串值 | 单引号 `'value'` |
| 带空格的别名 | 双引号 `"my column"`。双引号不能和点号访问混用 |
| 租户自定义字段 | 从租户命名空间开始写全路径：`_tenant.loyalty.tier` |

- 只选到对象一层会返回整个对象（显示为字符串）。要具体值就写到叶子字段，例如 `endUserIds._experience.mcid.id`。
- 数组用 `explode()` 展开成多行；结构体数组也可用 `inline()`。展开会放大行数，先过滤再展开。
- `identityMap` 是 map，键是身份命名空间：`identityMap['ECID']` 得到该命名空间下的身份数组，取第一个 ID 写 `identityMap['ECID'][0].id`。命名空间代码区分大小写，用用户环境里的实际代码。

## 把结果写成数据集

- 新建：`CREATE TABLE 表名 AS (SELECT ...)`。表名只能有单个下划线，不能出现连续两个下划线。
- 追加：`INSERT INTO 表名 SELECT ...`。SELECT 结果的结构和类型必须与目标表完全一致，类型不一致先 `CAST`。
- 对齐已有 XDM schema：
  ```sql
  CREATE TABLE 表名 WITH (schema='目标 Schema 标题', rowvalidation='false') AS (SELECT ...)
  ```
  需要进 Profile 时加 `label='PROFILE'`，并且先向用户确认，因为这会影响实时客户档案。
- 写嵌套结构不能用点号给目标字段赋值，必须用 `struct()` 逐层构造，最外层别名等于目标 schema 的顶层字段：
  ```sql
  SELECT struct(user_id AS userId, struct(tier AS tier, points AS points) AS loyalty) AS _tenant
  ```
- 聚合函数（`COUNT`、`SUM` 等）在 CTAS 中必须起别名。
- 批处理 CTAS/INSERT INTO 不受 10 分钟限制。把结果持久化到数据湖属于 Data Distiller 的能力，用户环境不一定开通，写回前提醒确认。

## 多步骤与增量

- 多条语句按顺序执行用匿名块，变量用 `SET @变量 = SELECT ...`，作用域只在块内：
  ```sql
  $$ BEGIN
    SET @from_id = SELECT ...;
    INSERT INTO target_table SELECT ... FROM source_table SNAPSHOT SINCE @from_id;
  EXCEPTION
    WHEN OTHERS THEN SET @ret = SELECT 'ERROR';
  END $$;
  ```
- `IF / ELSEIF / ELSE` 只能在匿名块内使用。
- `SNAPSHOT SINCE id`、`SNAPSHOT AS OF id`、`SNAPSHOT BETWEEN a AND b` 只能跟在表名或表别名后面，不能用于子查询和视图。`'HEAD'`、`'TAIL'` 要加单引号。
- 首次建表、之后追加的场景，用 `table_exists('表名')` 在匿名块里分支。
- 第三方客户端（如 DbVisualizer）执行匿名块可能需要额外的分隔符，报错时查该客户端的文档。

## Adobe 自定义函数

会话和归因优先用内置函数，不要手写等价逻辑。它们都是窗口函数，必须带 `OVER (PARTITION BY ... ORDER BY timestamp ...)`：

- 会话：`SESS_TIMEOUT(timestamp, 秒数)`、`SESS_START_IF`、`SESS_END_IF`
- 归因：`ATTRIBUTION_FIRST_TOUCH`、`ATTRIBUTION_LAST_TOUCH` 及带过期条件的变体
- 路径：`NEXT`、`PREVIOUS`
- 间隔：`TIME_BETWEEN_NEXT_MATCH`、`TIME_BETWEEN_PREVIOUS_MATCH`

返回值是结构体，用 `.字段` 取值（例如会话结果的 `.num`、`.is_new`）。具体参数以官方文档为准，不确定时让用户核对。

## 交付要求

1. 给出完整可运行的 SQL，用户环境特有的名称用清晰的占位符，并列出需要替换的内容。
2. 说明假设：时间范围按 UTC 还是本地时间、去重键是什么、用了哪个身份命名空间。
3. 附一条验证 SQL，例如行数对比、重复数检查、抽样 10 行。
4. 涉及写入（CTAS、INSERT INTO、DROP）时先给只读的 SELECT 版本让用户确认结果，再给写入版本。不要主动建议 `DROP TABLE`。
5. 没有在用户环境验证过的字段名和函数，明确标注"需核验"。

## 排错速查

| 现象 | 常见原因与处理 |
| --- | --- |
| 42601 语法错误 | 保留字未加反引号；双引号与点号混用；用了 PostgreSQL 专有语法 |
| 42P01 表不存在 | 用了数据集显示名而不是表名；沙箱不对。用 `SHOW TABLES` 核对 |
| 42P07 表已存在 | CTAS 目标表重名，换名或改用 `INSERT INTO` |
| Error validating schema | SELECT 的字段在目标 schema 中找不到，检查 `struct()` 的层级和别名 |
| Row failed XDM schema validation | 值违反 schema 约束，例如必填字段为空 |
| Function [...] not defined | 该函数 Spark SQL 不支持，换成等价的 Spark 函数 |
| 超过 10 分钟超时 | 缩小时间范围、只选必要列、改成批处理 CTAS、用 `EXPLAIN` 看执行计划 |
| 结果只有 50,000 行 | 交互式上限。需要全量时写成数据集，不要在界面里取全量 |

## 参考

- 官方文档：SQL 语法 https://experienceleague.adobe.com/en/docs/experience-platform/query/sql/syntax
- 官方文档：查询执行通用指南 https://experienceleague.adobe.com/en/docs/experience-platform/query/best-practices/writing-queries
- 官方文档：常见问题与排错 https://experienceleague.adobe.com/en/docs/experience-platform/query/troubleshooting-guide

产品行为会更新。本文件与官方文档或用户环境的实际表现不一致时，以后者为准，并提醒用户更新本技能。
