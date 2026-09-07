# ADR-0006：2026 工具链版本基线

- 状态：已接受
- 日期：2026-08-25
- 决策者：项目所有者

## 背景

早期设计锁定 Go 1.22、Node.js 20、pnpm 9、Next.js 15 和 Tailwind CSS 3。到 2026-08-25，这组版本已经不适合作为新项目基线：Node.js 20 已经过了新项目应采用的活跃 LTS 阶段，Go 1.22 也不应继续承载新建工程；同时，实际依赖解析已经证明旧 `golang.org/x/crypto v0.36.0` 与当前 Echo/pgx 依赖图冲突。

本项目尚未开始 M0，没有生产数据和兼容负担，因此现在升级的成本最低。

## 决策

采用以下可重复工具链：

| 工具                      |  固定版本 | 固定位置                                            |
| ------------------------- | --------: | --------------------------------------------------- |
| Node.js                   | `24.14.0` | 根目录 `.node-version`、CI                          |
| pnpm                      | `11.19.0` | `web/package.json#packageManager`、Corepack、CI     |
| Next.js / create-next-app |  `16.3.2` | 脚手架命令和 `web/package.json`                     |
| Tailwind CSS              |   `4.3.3` | `web/package.json`；使用 CSS-first 配置             |
| Go 语言版本               |  `1.25.0` | `server/go.mod` 的 `go` 指令                        |
| Go 工具链                 |  `1.25.3` | `server/go.mod` 的 `toolchain` 指令、CI、Dockerfile |

说明：Go 的 `go` 指令表达代码使用的最低语言语义，`toolchain` 指令表达默认构建工具链。二者分开可以避免把“语言特性版本”和“带安全/缺陷修复的补丁工具链”混为一谈。

前端采用 Next.js 16 App Router 的静态导出 `output: 'export'`。公共页面在构建期生成；后台页面使用 Client Component，但仍被导出为静态入口。Tailwind CSS 4 使用 `app/globals.css` 中的 `@theme` 管理 Design Tokens，不再创建只为沿用 v3 习惯而存在的 `tailwind.config.ts`。

## 版本治理规则

1. 文档中的精确版本是首次可重复搭建基线，不表示永久冻结。
2. Dependabot 每周提出依赖更新 PR；补丁版本通过完整 CI 后合并。
3. Node、Go、Next.js、Tailwind 的 major/minor 升级必须单独建分支和 ADR，禁止混入业务功能 PR。
4. 每季度检查运行时支持状态和安全公告；紧急安全更新不等待季度窗口。
5. `pnpm-lock.yaml`、`go.sum` 必须提交；CI 使用 frozen install 和生成漂移检查。
6. 不在任务书中手写间接依赖版本；由包管理器解析后，以 lockfile/checksum 文件为准。

## 后果

### 正面

- 新项目不从已经过期的运行时起步。
- 本地、CI、容器使用一致工具链，降低“本机能跑、CI 失败”的概率。
- Tailwind 4 的 CSS-first tokens 更适合后续视觉 DIY。
- Go 依赖图已在 `go1.25.3 windows/amd64` 完成普通测试和构建验证，Linux race detector 由 CI 继续验证。

### 代价

- 旧版 Next.js/Tailwind 教程中的配置文件和命令不能直接照搬。
- 精确补丁版本需要定期维护。
- Next.js 16 的静态导出限制必须通过构建测试持续检查，不能使用依赖请求期服务器运行时的 API。

## 被否决方案

- 继续使用旧版本：对尚未开工的新项目没有兼容收益，只会增加安全和迁移债务。
- 永远使用 `latest`：无法复现安装结果，CI 与学习步骤会随时间漂移。
- 立即采用刚发布且未经本项目验证的工具链：稳定性收益不足；先用已安装、已验证、仍受支持的精确版本。
