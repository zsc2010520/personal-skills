# 常用查询模板

所有模板里的表名、字段路径、命名空间都是占位，使用前替换成用户环境的真实值。`_tenant` 代表租户命名空间。

## 1. 探索数据集

```sql
SHOW TABLES;
```

```sql
SELECT timestamp, _id, web.webPageDetails.name AS page_name
FROM web_events
WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
  AND timestamp <  to_timestamp('2026-01-02 00:00:00')
LIMIT 10;
```

在 psql 类客户端里，`\d web_events` 查看列和类型，`\d web_events_web` 查看复合列 `web` 的子字段。

## 2. 按本地时间统计每日事件数

```sql
SELECT date_format(from_utc_timestamp(timestamp, 'Asia/Shanghai'), 'yyyy-MM-dd') AS local_date,
       COUNT(*) AS events
FROM web_events
WHERE timestamp >= to_utc_timestamp('2026-01-01 00:00:00', 'Asia/Shanghai')
  AND timestamp <  to_utc_timestamp('2026-01-08 00:00:00', 'Asia/Shanghai')
GROUP BY 1
ORDER BY 1;
```

## 3. 事件去重（保留最早一条）

去重键默认是事件的 `_id`。业务上有其他定义时（例如同一用户同一订单号），把 `PARTITION BY` 换成对应字段，并在交付时写明。

```sql
SELECT _id, timestamp, page_name
FROM (
  SELECT _id,
         timestamp,
         web.webPageDetails.name AS page_name,
         ROW_NUMBER() OVER (PARTITION BY _id ORDER BY timestamp ASC) AS rn
  FROM web_events
  WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
    AND timestamp <  to_timestamp('2026-01-02 00:00:00')
) t
WHERE rn = 1;
```

验证：

```sql
SELECT COUNT(*) AS total_rows, COUNT(DISTINCT _id) AS distinct_ids
FROM web_events
WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
  AND timestamp <  to_timestamp('2026-01-02 00:00:00');
```

## 4. 订单去重（同一用户同一订单号）

`order` 是保留字，需要反引号。

```sql
SELECT ecid, purchase_id, timestamp
FROM (
  SELECT identityMap['ECID'][0].id AS ecid,
         commerce.`order`.purchaseID AS purchase_id,
         timestamp,
         ROW_NUMBER() OVER (
           PARTITION BY identityMap['ECID'][0].id, commerce.`order`.purchaseID
           ORDER BY timestamp ASC
         ) AS rn
  FROM web_events
  WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
    AND timestamp <  to_timestamp('2026-02-01 00:00:00')
    AND commerce.`order`.purchaseID IS NOT NULL
) t
WHERE rn = 1;
```

## 5. 展开数组

```sql
SELECT _id, item.SKU AS sku, item.quantity AS quantity
FROM (
  SELECT _id, explode(productListItems) AS item
  FROM web_events
  WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
    AND timestamp <  to_timestamp('2026-01-02 00:00:00')
    AND productListItems IS NOT NULL
) t;
```

## 6. 查看某个身份命名空间的覆盖情况

```sql
SELECT COUNT(*) AS events,
       SUM(CASE WHEN identityMap['ECID'] IS NOT NULL THEN 1 ELSE 0 END) AS with_ecid,
       SUM(CASE WHEN identityMap['Email'] IS NOT NULL THEN 1 ELSE 0 END) AS with_email
FROM web_events
WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
  AND timestamp <  to_timestamp('2026-01-02 00:00:00');
```

## 7. 会话划分（30 分钟无活动切分）

```sql
SELECT ecid, timestamp, session.num AS session_num, session.is_new AS is_new_session
FROM (
  SELECT identityMap['ECID'][0].id AS ecid,
         timestamp,
         SESS_TIMEOUT(timestamp, 60 * 30) OVER (
           PARTITION BY identityMap['ECID'][0].id
           ORDER BY timestamp
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
         ) AS session
  FROM web_events
  WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
    AND timestamp <  to_timestamp('2026-01-02 00:00:00')
) t;
```

## 8. 找出 A 表有、B 表没有的记录

```sql
SELECT a.customer_id
FROM crm_customers a
WHERE NOT EXISTS (
  SELECT 1 FROM loyalty_members b WHERE b.customer_id = a.customer_id
);
```

## 9. 把结果写成新数据集（平铺结构）

```sql
CREATE TABLE daily_page_views AS (
  SELECT date_format(timestamp, 'yyyy-MM-dd') AS event_date,
         web.webPageDetails.name AS page_name,
         COUNT(*) AS page_views
  FROM web_events
  WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
    AND timestamp <  to_timestamp('2026-02-01 00:00:00')
  GROUP BY 1, 2
);
```

## 10. 写入符合 XDM schema 的数据集

目标 schema 的每一层都用 `struct()` 构造，别名与 schema 字段名一致，类型不一致时 `CAST`。

```sql
CREATE TABLE customer_ltv WITH (schema='Customer LTV Schema', rowvalidation='false') AS (
  SELECT struct(
           customer_id AS customerId,
           struct(
             CAST(total_revenue AS double) AS totalRevenue,
             CAST(order_count AS int) AS orderCount
           ) AS ltv
         ) AS _tenant
  FROM (
    SELECT identityMap['Email'][0].id AS customer_id,
           SUM(commerce.`order`.priceTotal) AS total_revenue,
           COUNT(DISTINCT commerce.`order`.purchaseID) AS order_count
    FROM web_events
    WHERE timestamp >= to_timestamp('2026-01-01 00:00:00')
      AND timestamp <  to_timestamp('2026-02-01 00:00:00')
      AND commerce.`order`.purchaseID IS NOT NULL
    GROUP BY 1
  ) s
);
```

之后追加数据用相同结构的 `INSERT INTO customer_ltv SELECT ...`。

## 11. 增量加载骨架

每次只处理源表自上次运行以来新增的数据。需要一张记录处理进度的日志表（这里叫 `checkpoint_log`）。这是骨架，快照 ID 的获取方式和日志表结构按官方"增量加载"文档补全后再使用。

```sql
$$ BEGIN
  SET @from_snapshot_id = SELECT coalesce(last_snapshot_id, 'HEAD')
                          FROM checkpoint_log a
                          JOIN (SELECT MAX(process_timestamp) AS process_timestamp
                                FROM checkpoint_log
                                WHERE process_name = 'my_job' AND process_status = 'SUCCESSFUL') b
                            ON a.process_timestamp = b.process_timestamp;

  SET @to_snapshot_id = SELECT snapshot_id
                        FROM (SELECT history_meta('source_table'))
                        WHERE is_current = true;

  SET @last_updated_timestamp = SELECT CURRENT_TIMESTAMP;

  INSERT INTO target_table
  SELECT _id, timestamp, page_name
  FROM source_table SNAPSHOT BETWEEN @from_snapshot_id AND @to_snapshot_id;

  INSERT INTO checkpoint_log
  SELECT 'my_job' AS process_name,
         'SUCCESSFUL' AS process_status,
         cast(@to_snapshot_id AS string) AS last_snapshot_id,
         cast(@last_updated_timestamp AS timestamp) AS process_timestamp;
EXCEPTION
  WHEN OTHERS THEN SET @ret = SELECT 'ERROR';
END $$;
```
