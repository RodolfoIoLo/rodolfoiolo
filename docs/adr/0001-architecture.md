# ADR-0001：个人网站总体架构

- 状态：已接受
- 日期：2026-08-26

## 决策

公共站点使用 Next.js App Router 静态导出；API 使用 Go/Echo；数据使用 PostgreSQL；Nginx 作为唯一公网入口；生产部署在阿里云 ECS 的 Docker Compose；GitHub Actions 执行契约检查、测试、构建和发布。

## 调用链

浏览器 → Nginx/TLS → 静态文件或 /api/v1 → Go middleware → Strict Handler → Service → pgx Repository → PostgreSQL。

## 原因

静态导出减少生产运行时和内存占用；Go/Echo 适合 2G ECS；PostgreSQL 同时承担事务、全文检索和 JSONB；OpenAPI-first 保证前后端契约唯一。

## 约束

公共页面不得依赖请求期 Node 服务；Handler 不写 SQL；Repository 不生成 HTTP 响应；生产数据库不暴露公网；所有变更经 PR。
