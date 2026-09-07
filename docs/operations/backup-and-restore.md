# 备份与恢复手册

RPO 24 小时，RTO 2 小时。PostgreSQL 每日逻辑备份并同步到异地域私有 OSS；素材原件、Krita 工程和应用密钥进入加密离线备份。

systemd timer 调用 `scripts/backup-db.sh`。每份备份必须同时存在 `.sql.gz` 和 `.sha256`，通过 `gzip -t` 与 `sha256sum --check` 后才同步。监控最近成功时间、字节数和磁盘余量。

每月至少执行一次 `scripts/restore-drill.sh <绝对备份路径>`，只恢复到随机临时 PostgreSQL 容器，检查用户、文章和迁移表后销毁；再从异地 OSS 下载一份重复演练。

生产恢复前必须保存现状备份、在隔离容器验证目标备份、开启维护窗口并得到事故负责人批准。禁止把未经验证的 SQL 直接输入生产。
