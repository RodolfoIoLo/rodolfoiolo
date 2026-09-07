# ADR-0002：认证与会话安全

- 状态：已接受
- 日期：2026-08-26

Access Token 使用 RSA RS256 JWT，15 分钟有效，只保存在前端内存；Refresh Token 使用 32 字节随机值，浏览器只通过 HttpOnly/SameSite=Lax Cookie 发送，数据库只保存 SHA-256。刷新在 PostgreSQL 事务中 SELECT FOR UPDATE 锁定旧记录并轮换；检测到重放时吊销该用户全部会话。

写操作同时要求同源 Origin 和 X-Requested-With，登录使用按 HMAC-IP 的 Token Bucket 限流。日志不得记录密码、Token、Cookie、邮箱原文或正文。
