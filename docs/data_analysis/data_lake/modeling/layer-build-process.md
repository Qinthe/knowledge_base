# 各层制作流程详解

各层关系像一条**数据管道**，从下往上逐层构建：**Bronze → Silver → Gold → Application**。

## 1. Bronze 层制作

**目标**：把源库数据"原样"搬进湖里，不做任何加工。

**工具**：数据同步工具直接完成，例如**阿里云 DTS**。

**步骤**：

1. 在 OSS（或 HDFS）建好 Bronze 层目录（如 `lakehouse/bronze/sqlserver/db_orders/`）。
2. 用同步工具创建"全量 + 增量（CDC）"同步任务，目标格式选 **Delta Lake**。
3. 启动任务，源库全量数据先落盘，之后变更实时追加。

**产出**：与源库一一对应的 Delta Lake 表，带 `_delta_log` 目录。

## 2. Silver 层制作

**目标**：对 Bronze 数据清洗、标准化、建分区，产出干净明细。

**工具**：Spark SQL（EMR / DataWorks）。

**典型步骤与示例**：

1. **读取 Bronze 表**：

   ```sql
   SELECT * FROM bronze_db.orders;
   ```

2. **清洗 + 建分区 + 写入 Silver**：

   ```sql
   CREATE TABLE silver_db.orders_cleaned
   USING delta
   PARTITIONED BY (dt)
   AS
   SELECT
     order_id,
     CAST(order_amount AS DECIMAL(18,2)) AS order_amount,  -- 格式统一
     COALESCE(customer_id, 'UNKNOWN')      AS customer_id,  -- 空值处理
     TRIM(status)                          AS status,       -- 去空格
     order_date                            AS dt
   FROM bronze_db.orders
   WHERE order_id IS NOT NULL;  -- 去重/剔除脏数据
   ```

3. **校验**：检查 Silver 表行数、分区数是否符合预期。

## 3. Gold 层制作

**目标**：按主题域聚合，生成宽表/汇总表。

**示例**（按天汇总销售额）：

```sql
CREATE TABLE gold_db.daily_sales_summary
USING delta
PARTITIONED BY (dt)
AS
SELECT
  order_date            AS dt,
  COUNT(DISTINCT customer_id) AS customer_cnt,
  SUM(order_amount)          AS total_amount,
  COUNT(*)                   AS order_cnt
FROM silver_db.orders_cleaned
GROUP BY order_date;
```

## 4. Application 层制作

**目标**：按具体报表裁剪数据，提供给 BI/API。

**常见做法**：

- 从 Gold/Silver 建**视图（View）**，BI 工具直接连视图。
- 或用调度任务把需要的列导出成 BI 引擎能直接读的表。

## 5. 全流程依赖与调度

- 顺序固定：Bronze → Silver → Gold，不能跳层。
- 用调度工具（DataWorks 调度 / Airflow）编排依赖：Silver 任务必须等 Bronze 完成后才能跑。
- 失败重跑：因为 Bronze 有快照，Silver/Gold 失败可直接删除重建，无需动源库。
