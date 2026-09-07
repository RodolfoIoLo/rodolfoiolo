# ADR-0005：数据库访问统一使用 pgx 与手写 SQL Repository

- 状态：Accepted
- 日期：2026-08-25
- 决策范围：Go API 的 PostgreSQL 访问与迁移

## 背景

总体设计早期选择了 Ent，但已经确定的迁移和服务器基础使用手写 SQL 与 `pgxpool`。如果继续同时引入 Ent，会出现两套实体定义、两套查询方式和额外生成代码。项目内容量很小，而学习目标要求能够看见 SQL、事务边界和错误来源。

## 决策

运行时统一使用：

- `github.com/jackc/pgx/v5/pgxpool` 管理连接池；
- Repository 文件保存参数化 SQL、执行查询并映射领域记录；
- Service 决定业务规则和事务边界，通过 `pgx.Tx` 调用 Repository；
- `golang-migrate` 管理独立的版本化 SQL 迁移；
- OpenAPI 生成的 DTO 只在 Handler 边界使用，不作为数据库实体。

不引入 Ent、GORM 或第二套 ORM。

## 原因

1. 所有关键 SQL 和锁行为对学习者、Reviewer 和性能排查都可见；
2. pgx 原生支持 PostgreSQL 类型、批处理、事务与错误码；
3. 内容规模不需要通用 ORM 的动态查询抽象；
4. 减少生成代码和内存/依赖成本；
5. 参数化 SQL 与服务端排序白名单可以安全防止 SQL 注入。

## 代价与控制

- 行扫描代码更多：通过小型私有扫描函数集中重复字段；
- SQL 重构没有 ORM 自动提示：Repository 集成测试必须连接真实 PostgreSQL；
- 动态筛选容易拼接错误：只拼接程序常量，所有用户值仍使用 `$1...$n` 参数；
- 迁移与查询可能漂移：CI 对空数据库执行全部迁移并运行 Repository 测试。

## 被否决的方案

- Ent：类型安全优秀，但本项目会额外维护 Schema/生成物，且掩盖需要学习的 SQL 与锁细节。
- GORM：上手快，但隐式查询和运行时模型不利于本项目的契约与 SQL 学习目标。
- sqlc：可生成强类型查询，但又增加一个生成链；当前查询数量可控，先保持直接 pgx。查询规模明显增长时再写新 ADR 评估。

## 后果

所有后续工作包和参考代码只使用 pgx。发现 Ent/GORM import 或 ORM Schema 目录时，完整性检查必须失败。
