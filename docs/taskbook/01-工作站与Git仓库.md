# 第 01 章：工作站与 Git 仓库

本章从现有的 `docs/` 开始初始化仓库，不写业务代码。命令按 PowerShell 7 编写；代码块开头的 `PS>` 只表示终端，不要输入。

## 01-01 打开正确的项目和终端

预计时间：10 分钟。

1. 打开 VS Code。
2. 点击“文件” -> “打开文件夹”。
3. 选择 `D:\CODEkingdom\rodolfoiolo`。
4. 点击“选择文件夹”。
5. 点击“终端” -> “新建终端”。
6. 查看终端标签，确认使用 PowerShell；如果是 Command Prompt，点击终端右侧下拉箭头，选择 PowerShell。
7. 逐行执行：

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> Get-Location
PS> Get-ChildItem -Force
```

预期：`Get-Location` 显示 `D:\CODEkingdom\rodolfoiolo`；根目录只有 `docs`。如果出现其他源码，停止并核对是否打开了错误目录。

## 01-02 检查工具，不先盲目安装

预计时间：20 分钟。

逐行执行：

```powershell
PS> $PSVersionTable.PSVersion
PS> git --version
PS> node --version
PS> corepack --version
PS> go version
PS> docker --version
PS> docker compose version
PS> code --version
```

基线成功值：

| 工具 | 允许值 |
|---|---|
| PowerShell | `7.x` |
| Git | `2.51.x` 或更高兼容版本 |
| Node.js | `v24.14.0` |
| Corepack | `0.34.x` |
| Go | `go1.25.3 windows/amd64` |
| Docker | Engine 29.x，Compose v5.x |
| VS Code | 1.100 或更新稳定版 |

如果命令存在且版本匹配，不重复安装。某一项缺失时只处理该项：

```powershell
PS> winget install --id Microsoft.PowerShell --exact
PS> winget install --id Git.Git --exact
PS> winget install --id OpenJS.NodeJS.LTS --exact
PS> winget install --id GoLang.Go --exact
PS> winget install --id Docker.DockerDesktop --exact
PS> winget install --id Microsoft.VisualStudioCode --exact
```

一次只执行所需的一条。安装后完全关闭并重新打开 VS Code，再重复版本检查。若 `winget` 提供的 Node 或 Go 已超过任务书固定版本，不降级现有安全版本；在施工记录记下差异，后续以 `.node-version` 和 `go.mod` 为项目声明，以 CI 固定版本验证。

## 01-03 启用精确 pnpm

预计时间：10 分钟。

```powershell
PS> corepack install --global pnpm@11.19.0
PS> pnpm --version
```

预期输出：`11.19.0`。如果 PowerShell 提示脚本执行被禁止，不修改全局执行策略，先运行：

```powershell
PS> pnpm.cmd --version
```

若 `pnpm.cmd` 可用，后续把命令中的 `pnpm` 替换为 `pnpm.cmd`。若 Corepack 下载失败，检查网络后重试，不能改用未固定版本的 `npm install -g pnpm`。

## 01-04 检查 Docker Desktop

预计时间：10 分钟。

1. 从开始菜单打开 Docker Desktop。
2. 等待界面显示 Engine running。
3. 回到 VS Code 终端执行：

```powershell
PS> docker info --format '{{.ServerVersion}}'
PS> docker run --rm hello-world
```

预期：第一条输出服务端版本，第二条包含 `Hello from Docker!`。如果提示无法连接 daemon，确认 Docker Desktop 已启动并等待；不要以管理员身份反复启动不同实例。

## 01-05 配置 Git 身份和换行

预计时间：10 分钟。

先查看现有值：

```powershell
PS> git config --global --get user.name
PS> git config --global --get user.email
PS> git config --global --get init.defaultBranch
```

只有名字或邮箱为空时才执行下面两条，并替换成你的 GitHub 展示名和已验证邮箱：

```powershell
PS> git config --global user.name '<YOUR_NAME>'
PS> git config --global user.email '<YOUR_GITHUB_EMAIL>'
```

然后执行：

```powershell
PS> git config --global init.defaultBranch main
PS> git config --global core.autocrlf false #关闭git自动转换换行符,不自动把LF,CRLF互相转换,文件原样保存
PS> git config --global core.safecrlf warn #当文件存在混合换行符,提交时给警告
PS> git config --global fetch.prune true #拉取远程分支时,自动删除本地已经在远程被删除的分支引用
PS> git config --global --list
```

`.gitattributes` 会统一仓库换行；`core.autocrlf=false` 避免 Windows 在提交时偷偷改写脚本。// `.gitattirbutes` 放在项目根目录,会提交到git仓库,团队所有人共用这套规则,优先级高于本机全局git config

## 01-06 初始化本地 Git 仓库

预计时间：15 分钟。

先验证当前目录尚不是仓库：

```powershell
PS> git rev-parse --is-inside-work-tree
```

预期：报错 `not a git repository`。只有出现这个预期结果才执行：

```powershell
PS> git init --initial-branch=main
PS> git status --short --branch
```

预期包含：

```text
## No commits yet on main
?? docs/
```

如果第一条检查已经输出 `true`，不要再次 `git init`；运行 `git status --short --branch` 并从真实状态继续。

## 01-07 建立第一次基线提交

预计时间：15 分钟。

先检查文档文件：

```powershell
PS> Get-ChildItem -LiteralPath docs -Recurse -File | Select-Object FullName
PS> git status --short
PS> git diff --no-index -- NUL docs/README.md
```

`git diff --no-index` 返回退出码 1 表示存在差异，这是预期，不是失败。随后暂存明确路径：

```powershell
PS> git add -- docs
PS> git diff --cached --stat
PS> git diff --cached --check
PS> git commit -m 'docs: establish rebuild specification baseline'
PS> git status --short --branch
PS> git log --oneline --decorate -1
```

预期：工作区干净；最新提交信息与上面一致。`--check` 有输出时先修复行尾空格再提交。

## 01-08 在 GitHub 创建空远端

预计时间：15 分钟。

1. 浏览器登录 GitHub。
2. 右上角点击 `+` -> `New repository`。
3. `Owner` 选择你的账号。
4. `Repository name` 输入准备表中的仓库名。
5. `Description` 输入 `Personal website built with an enterprise-style delivery workflow.`。
6. 选择 `Public`；如果当前不希望公开，可选 `Private`，不影响流程。
7. 不勾选 Add a README、`.gitignore` 或 License，因为本地已有提交。
8. 点击 `Create repository`。
9. 保留页面，复制 HTTPS 地址，形如 `https://github.com/<GITHUB_USER>/<REPOSITORY>.git`。

回到终端，替换地址后执行：

```powershell
PS> git remote add origin 'https://github.com/<GITHUB_USER>/<REPOSITORY>.git'
PS> git remote -v
PS> git push --set-upstream origin main
```

浏览器刷新仓库，预期能看到 `docs/` 和第一次提交。认证弹窗出现时使用 Git Credential Manager 登录，不把 Token 写入命令。

## 01-09 配置 GitHub 分支保护

预计时间：15 分钟。

1. 仓库页面点击 `Settings`。
2. 左侧点击 `Rules` -> `Rulesets`。
3. 点击 `New ruleset` -> `New branch ruleset`。
4. Name 输入 `protect-main`，Enforcement status 选择 `Active`。
5. Target branches 点击 Add target -> Include default branch。
6. 勾选 Require a pull request before merging。
7. Required approvals 设为 `0`，因为这是单人学习仓库；仍必须走 PR 自审。
8. 勾选 Require conversation resolution before merging。
9. 暂不勾选 Required status checks，因为 CI 尚未创建；第 02 章会回来补。
10. 勾选 Block force pushes 和 Restrict deletions。
11. 点击 Create。

企业思维：单人项目使用 PR，不是模拟多人表演，而是让主线变更有 diff、自动门禁和回滚边界。

## 01-10 演练完整分支生命周期

预计时间：20 分钟。

创建只修改文档的练习分支：

```powershell
PS> git switch -c docs/verify-workflow
PS> git status --short --branch
```

在施工记录写下分支名。不要修改文件，直接验证空分支不能产生提交。然后推送并创建 PR：

```powershell
PS> git push --set-upstream origin docs/verify-workflow
```

GitHub 仓库页面点击 Compare & pull request：

1. 标题输入 `docs: verify pull request workflow`。
2. 正文写 `This empty practice branch verifies repository permissions. No merge is required.`。
3. 点击 Create pull request。
4. 确认 Files changed 为 0。
5. 点击 Close pull request，不合并空 PR。
6. 点击 Delete branch。

本地清理：

```powershell
PS> git switch main
PS> git pull --ff-only
PS> git branch -D docs/verify-workflow
PS> git fetch --prune
PS> git status --short --branch
```

这里允许 `-D`，因为分支没有提交且远端 PR 已关闭。正常功能分支一律在确认合并后用 `git branch -d`。

## 01-11 日常 Git 固定动作

以后每个工作单元开始时执行：

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main #切到main主分支
PS> git pull --ff-only #拉取远程main更新(仅允许快进合并,有冲突拒绝拉取,不自动生成合并提交)
PS> git status --short --branch #精简格式输出文件变更状态,附带当前分支名
PS> git switch -c <任务书指定分支> #从main新建并切换到任务分支
```

每个提交前执行：

```powershell
PS> git status --short #简短现实改动文件列表
PS> git diff -- <任务书列出的路径> #查看指定路径工作区和缓存区之间改动
PS> git diff --check #检查代码末尾空格,换行符异常
PS> git add -- <任务书列出的路径> 
PS> git diff --cached #查看暂存区和上一次提交的变更
PS> git commit -m '<任务书指定信息>'
```

每个分支结束时执行：

```powershell
PS> git push --set-upstream origin <分支名> #推送本地分支到远程仓库origin,绑定上下游追踪关系
```

然后在 GitHub 创建 PR、逐文件自审、等待 CI、Squash and merge。合并后：

```powershell
PS> git switch main 
PS> git pull --ff-only #拉取合并后的远程main分支更新,快进方式更新本地main
PS> git branch -d <分支名> #安全删除本地任务分支
PS> git fetch --prune #拉取远程分支信息,清理本地已经在远程被删除的分支引用
```

禁止命令：`git reset --hard`、`git push --force`、`git checkout -- .`、未经检查的 `git clean -fd`。

## 01-12 本章停止点

- [ ] 工具版本已记录；
- [ ] pnpm 可输出固定版本；
- [ ] Docker hello-world 成功；
- [ ] Git 身份正确；
- [ ] `main` 有第一次文档提交；
- [ ] GitHub 远端可见；
- [ ] `protect-main` ruleset 已启用；
- [ ] 空 PR 演练已关闭并清理；
- [ ] 本地 `git status --short --branch` 干净。

全部满足后进入第 02 章。
