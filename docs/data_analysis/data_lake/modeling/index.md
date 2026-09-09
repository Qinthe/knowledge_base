# 数据湖建模说明

企业级数据湖建模方法，以 Medallion 分层模型为主线。

## 文档列表

- [核心概念与术语](concepts.md) — 数据湖、元数据、ACID、Schema Evolution 等基础概念
- [分层模型（Medallion）详解](layered-architecture.md) — Bronze/Silver/Gold/Application 四层职责
- [各层制作流程](layer-build-process.md) — 从原始数据到应用层的完整流程与 SQL 示例
- [各层数据调用](data-access.md) — 不同角色从哪一层取数
- [性能与扩展性设计](performance-tuning.md) — 小文件、分区、分桶等优化手段
