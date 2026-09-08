# 阿里云 DataLake 说明与使用

本系列讲解如何使用阿里云服务，把 **SQL Server 数据实时入湖** 并继续分层加工。核心服务组合：**OSS（存储）+ DLF（元数据）+ DTS（同步）+ EMR（计算）**。

## 整体流程一览

1. 规划 OSS 目录结构
2. 开通并授权服务
3. 创建 OSS Bucket 与目录
4. DLF 建元数据库
5. 配置数据源
6. 创建实时入湖任务
7. 验证数据
8. Silver/Gold 分层加工

<div class="grid cards" markdown>

-   :material-swap-horizontal:{ .lg .middle } [**服务与术语映射**](service-mapping.md)

    ---

    OSS / DLF / DTS / EMR 各是什么、对应哪个通用概念。

-   :material-file-tree:{ .lg .middle } [**目录结构规划**](directory-planning.md)

    ---

    在 OSS 里怎么建目录，各层路径长什么样。

-   :material-account-key:{ .lg .middle } [**入湖前置准备**](prerequisites.md)

    ---

    开通服务、网络白名单、RAM 授权。

-   :material-play-circle:{ .lg .middle } [**详细入湖操作步骤**](ingestion-guide.md)

    ---

    从创建 Bucket 到验证数据的完整实操（核心）。

-   :material-chart-box-outline:{ .lg .middle } [**Silver/Gold 分层与优化**](silver-gold-processing.md)

    ---

    入湖之后如何继续清洗、汇总与调优。

</div>
