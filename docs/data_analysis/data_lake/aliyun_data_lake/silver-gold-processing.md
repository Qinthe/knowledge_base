# Silver/Gold 分层调用与优化

入湖完成后，Bronze 层有了原始数据。下面继续加工出 Silver（清洗）、Gold（汇总）。

## 1. 从 Bronze 读取并处理

使用 **EMR** 或 **DataWorks** 创建 Spark 作业，直接读 DLF Catalog 中的表。

### 1.1 创建使用 DLF 元数据的 EMR 集群

- 创建 EMR 集群时，在"元数据"选项中选择 **"DLF 统一元数据"**。
- 这样 EMR 里的 Spark 就能直接 `SELECT` DLF 里的表，无需自建 Hive Metastore。

### 1.2 编写 Spark SQL 做 ETL

示例：从 Bronze 表 `ods_sqlserver_db.employees` 读取，清洗后写入 Silver 表，并按业务日期分区：

```sql
CREATE TABLE dwd_sales_db.employees_cleaned
USING delta
PARTITIONED BY (dt)
AS
SELECT
  employee_id,
  TRIM(name)                   AS name,
  CAST(salary AS DECIMAL(18,2)) AS salary,
  hire_date                    AS dt
FROM ods_sqlserver_db.employees
WHERE employee_id IS NOT NULL;
```

> 要点：创建 Silver 层表时，**使用 `PARTITIONED BY` 子句按业务日期分区**。

## 2. 性能优化

### 2.1 合并小文件（Optimize）

- **问题**：DTS 频繁微批写入会在 Bronze 层产生大量小文件。
- **对策**：在 Silver ETL 前/后，对 Delta 表执行 `OPTIMIZE`；或在 DLF"湖格式管理"中配置自动优化策略。

```sql
OPTIMIZE ods_sqlserver_db.employees;
```

### 2.2 生命周期管理

- 在 DLF"湖管理"中，对 `/lakehouse/bronze/` 目录的 **Location** 配置**生命周期规则**：如 30 天后转低频存储（省钱）、1 年后删除。
- 有效控制存储成本膨胀。

## 3. 通用性保证

- **元数据统一**：所有计算引擎（EMR、Flink、MaxCompute）都通过 **DLF 统一 Catalog** 读写元数据，而非各自维护 Hive Metastore。
- **Schema Evolution**：源表新增字段时，开启入湖任务的 **"自动 Schema Evolution"** 选项，或用 Spark SQL `ALTER TABLE ... ADD COLUMN`，避免任务因结构变化中断。
