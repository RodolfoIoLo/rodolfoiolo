# Rodolfo Iolo 个人网站文档

本仓库当前处于“从零重建”状态：实现层为空，`docs/` 是施工前唯一保留的资料。

## 唯一执行入口

从 [taskbook/README.md](taskbook/README.md) 开始，严格按章节顺序执行。`taskbook/` 是一套按时间顺序拆分的任务书，不是多套互相竞争的方案。

## 规范资料

- [design/product-and-visual-baseline.md](design/product-and-visual-baseline.md)：产品范围、信息架构和视觉基线。
- [api/openapi.yaml](api/openapi.yaml)：HTTP API 的唯一契约源。
- [adr/](adr/)：已经接受的架构决策。
- [operations/](operations/)：部署、备份、恢复和事故处理手册。

## 文档权威顺序

发生冲突时，按以下优先级判断：

1. 当前任务书中较晚的、明确标注“替换全文”的步骤；
2. `docs/api/openapi.yaml`；
3. 已接受的 ADR；
4. 产品与视觉基线；
5. 运维手册。

旧版 v1/v2 任务书、覆盖台账和未完成工作包已被新任务书取代，不得继续引用。
