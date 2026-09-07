# ADR-0004：媒体存储

- 状态：已接受
- 日期：2026-08-26

头像、项目截图和背景图存储在阿里云 OSS 私有 Bucket，公网通过启用私有源站回源鉴权的 CDN 自定义 HTTPS 域名读取；不允许匿名 OSS 直链。管理型临时对象可以使用短期签名 URL。API 只保存对象 key、MIME、字节数、宽高和 hash，不保存 AccessKey。上传先校验 MIME、扩展名、大小和图片解码结果，再写入对象存储；失败不写数据库元数据。

背景图预算：首屏 WebP/AVIF ≤300KB，五张总计 ≤1.5MB。素材授权、来源和 SHA-256 记录在 docs/design/assets/background-sources.md。
