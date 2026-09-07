# 生产部署与回滚手册

只部署 `main` 上通过全部 required checks 的 40 位 Git SHA。发布前确认磁盘余量大于 20%、最近备份和隔离恢复有效，并记录当前 release、API 镜像和迁移版本。

在 ECS 的 `/opt/personal-site/repository` 执行 `./scripts/deploy.sh <40位SHA>`。脚本验证 SHA 属于 `origin/main`、持有发布锁、备份数据库、构建 SHA 镜像、执行向前迁移、等待 readiness、从真实 API 构建静态站点、原子切换 `releases/current` 并执行外部冒烟。

发布后从外部运行 `./scripts/smoke-production.ps1 -Domain <生产域名>`。失败时选择同时存在完整静态目录和镜像的前一健康版本，执行 `./scripts/rollback.sh <12位release>`。数据库不执行 down；schema 必须兼容上一版 API。

每次发布记录 UTC 时间、操作者、Git SHA、镜像 ID、迁移版本、旧/新 release、冒烟结果和回滚目标，不记录密钥、Cookie 或数据库 URL。
