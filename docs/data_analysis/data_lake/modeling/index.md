# 数据湖建模说明

本系列文档讲解企业级数据湖的建模方法，以业界标准的 **奖牌（Medallion）分层模型** 为主线，从概念到实操逐步展开。

**适用读者**：需要理解数据湖分层、并动手规划数据湖结构的数据工程师与数据分析师。

<div class="grid cards" markdown>

-   :material-book-open-variant:{ .lg .middle } [**核心概念与术语**](concepts.md)

    ---

    数据湖、Catalog、元数据、ACID、Schema Evolution 等基础概念，先打牢地基。

-   :material-layers-triple:{ .lg .middle } [**分层模型（Medallion）详解**](layered-architecture.md)

    ---

    Bronze / Silver / Gold / Application 四层分别干什么、为什么这样分。

-   :material-hammer-wrench:{ .lg .middle } [**各层制作流程**](layer-build-process.md)

    ---

    从原始数据到应用层，逐层怎么"做出来"的完整流程。

-   :material-account-search:{ .lg .middle } [**各层数据调用**](data-access.md)

    ---

    数据工程师、数据科学家、BI 分别从哪里取数。

-   :material-speedometer:{ .lg .middle } [**性能与扩展性设计**](performance-tuning.md)

    ---

    小文件、分区、分桶等常见性能坑与优化手段。

</div>
