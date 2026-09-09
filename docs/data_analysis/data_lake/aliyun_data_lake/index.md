# 阿里云 DataLake 说明与使用

使用 OSS + DLF + DTS + EMR 把 SQL Server 数据实时入湖。

## 文档列表

- [服务与术语映射](service-mapping.md) — OSS/DLF/DTS/EMR 各是什么、对应哪个通用概念
- [目录结构规划](directory-planning.md) — OSS 里怎么建目录、各层路径长什么样
- [入湖前置准备](prerequisites.md) — 开通服务、网络白名单、RAM 授权
- [详细入湖操作步骤](ingestion-guide.md) — 从创建 Bucket 到验证数据的完整实操
- [Silver/Gold 分层与优化](silver-gold-processing.md) — 入湖后如何清洗、汇总与调优
