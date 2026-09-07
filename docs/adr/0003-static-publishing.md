# ADR-0003：静态发布策略

- 状态：已接受
- 日期：2026-08-26

web 使用 Next.js output export。构建阶段从 CONTENT_API_BASE_URL 拉取已发布文章和 active 项目，生成完整 HTML、RSS、Sitemap。Nginx 通过 releases/<git-sha> 和 releases/current 原子切换静态目录；构建失败保留旧版本。

限制：不能使用依赖请求期 Server Action、动态 API Route 或运行时数据库查询的页面。内容更新通过 CI 手动或定时触发重新构建。
