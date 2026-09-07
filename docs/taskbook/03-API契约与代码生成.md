# 第 03 章：API 契约与代码生成

本章不实现业务。目标是学会先定义 HTTP 边界，再让前后端从同一份契约获得类型。`docs/api/openapi.yaml` 已作为任务书资料完整保留，不复制第二份，避免两个“真相源”。

## 03-01 创建契约分支

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main
PS> git pull --ff-only
PS> git status --short --branch
PS> git switch -c feat/openapi-contract
```

## 03-02 阅读契约而不是直接生成

在 VS Code 打开 `docs/api/openapi.yaml`。依次折叠并展开 `info`、`servers`、`paths`、`components/schemas`、`components/securitySchemes`。

终端执行：

```powershell
PS> (Select-String -LiteralPath 'docs\api\openapi.yaml' -Pattern '^  /').Count
PS> (Select-String -LiteralPath 'docs\api\openapi.yaml' -Pattern '^      operationId:').Count
PS> Select-String -LiteralPath 'docs\api\openapi.yaml' -Pattern '^  /','^      operationId:' | ForEach-Object { $_.Line.Trim() }
```

预期：33 条 path、47 个 operationId。数量变化不一定错误，但必须能在 PR 中解释新增或删除的接口。

在施工记录画出登录调用链：`LoginRequest -> POST /auth/login -> AccessToken response + Set-Cookie -> LoginError`。

## 03-03 理解契约中的四种访问级别

逐项核对：

1. 公开读：文章、项目、分类、标签、公开设置、RSS、Sitemap、healthz。
2. 管理写：创建、更新、发布、删除、后台统计和审核；要求 Bearer Access Token。
3. 刷新会话：只使用 HttpOnly Refresh Cookie，JavaScript 不能读取。
4. 匿名互动：签名访客 Cookie + 限流；不能借此访问管理接口。

发现写接口没有 `security` 时停止。不要用“前端不显示按钮”代替服务端授权。

## 03-04 使用固定容器 lint OpenAPI

先拉取固定工具镜像：

```powershell
PS> docker pull dshanley/vacuum:0.30.0
PS> $repoPath = (Get-Location).Path
PS> docker run --rm --mount "type=bind,source=$repoPath,target=/work,readonly" dshanley/vacuum:0.30.0 lint /work/docs/api/openapi.yaml
```

预期：命令退出码为 0。Warning 可以记录并评估，Error 必须修复。不要把 `$repoPath` 命名为 `$HOME`，也不要把整个磁盘挂进容器。

再次执行，确保结果可重复：

```powershell
PS> docker run --rm --mount "type=bind,source=$repoPath,target=/work,readonly" dshanley/vacuum:0.30.0 lint /work/docs/api/openapi.yaml
PS> $LASTEXITCODE
```

预期最后输出 `0`。

## 03-05 建立 Go 模块和生成配置

创建目录：

```powershell
PS> New-Item -ItemType Directory -Force -Path 'server\generated\oapi' | Out-Null
PS> Set-Location 'server'
PS> go mod init 'github.com/<GITHUB_USER>/<REPOSITORY>/server'
PS> go mod edit -go=1.25.0
PS> go mod edit -toolchain=go1.25.3
PS> Set-Location '..'
```

必须把 `<GITHUB_USER>` 和 `<REPOSITORY>` 替换为准备表真实值。检查：

```powershell
PS> Get-Content -LiteralPath 'server\go.mod'
```

预期结构为：

```go
module github.com/<GITHUB_USER>/<REPOSITORY>/server

go 1.25.0

toolchain go1.25.3
```

`go.mod` 由 Go 命令生成；不要为了和示例空行完全一致而手工改写。

创建 `server/oapi-codegen.yaml`，文件全文：

```yaml
package: oapi
output: generated/oapi/api.gen.go
generate:
  echo-server: true
  strict-server: true
  models: true
  embedded-spec: true
output-options:
  skip-prune: true
  name-normalizer: ToCamelCaseWithInitialisms
```

创建 `server/generate.go`，文件全文：

```go
package server

//go:generate go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.4.1 --config oapi-codegen.yaml ../docs/api/openapi.yaml
```

为什么放在根 package：`go generate ./...` 会发现唯一生成入口；配置参数不复制到 CI 和个人脚本。

## 03-06 首次生成 Go 契约

```powershell
PS> Set-Location 'server'
PS> go generate ./...
PS> Set-Location '..'
PS> Get-Item -LiteralPath 'server\generated\oapi\api.gen.go' | Select-Object FullName,Length
PS> Get-Content -LiteralPath 'server\generated\oapi\api.gen.go' -TotalCount 8
```

预期：文件非空，头部含 `Code generated`。这是命令生成文件：提交但禁止手改。

生成代码会引用 Echo 和 oapi runtime。固定运行时依赖并整理模块：

```powershell
PS> Set-Location 'server'
PS> go get github.com/labstack/echo/v4@v4.13.4
PS> go get github.com/oapi-codegen/runtime@v1.1.2
PS> go mod tidy
PS> go test ./generated/...
PS> Set-Location '..'
```

如果精确版本在执行时已因安全撤回，停止并为升级写 ADR；不能偷偷去掉版本号。

## 03-07 生成 TypeScript 契约

在根 workspace 安装固定生成器：

```powershell
PS> pnpm add --save-dev --workspace-root openapi-typescript@7.10.1
PS> New-Item -ItemType Directory -Force -Path 'web\types' | Out-Null
PS> pnpm exec openapi-typescript 'docs/api/openapi.yaml' --output 'web/types/openapi.d.ts'
PS> Get-Item -LiteralPath 'web\types\openapi.d.ts' | Select-Object FullName,Length
PS> Get-Content -LiteralPath 'web\types\openapi.d.ts' -TotalCount 8
```

预期：头部含 `This file was auto-generated`。提交但不手工编辑。

## 03-08 检查重复 operationId 和生成漂移

PowerShell 检查 operationId 唯一：

```powershell
PS> $operationIds = Select-String -LiteralPath 'docs\api\openapi.yaml' -Pattern '^      operationId:' | ForEach-Object { ($_.Line -split ':',2)[1].Trim() }
PS> $duplicates = $operationIds | Group-Object | Where-Object Count -gt 1
PS> $duplicates
PS> $operationIds.Count
```

预期：`$duplicates` 无输出，数量为 47。

连续生成两次并确认无变化：

```powershell
PS> Set-Location 'server'
PS> go generate ./...
PS> Set-Location '..'
PS> pnpm exec openapi-typescript 'docs/api/openapi.yaml' --output 'web/types/openapi.d.ts'
PS> git diff --exit-code -- server/generated/oapi/api.gen.go web/types/openapi.d.ts
PS> $LASTEXITCODE
```

预期退出码 0。非 0 表示生成不稳定、配置不同或有人手改生成物。

## 03-09 创建统一生成检查脚本

创建 `scripts/generate.ps1`，文件全文：

```powershell
[CmdletBinding()]
param()

$ErrorActionPreference = 'Stop'
$repositoryRoot = Split-Path -Parent $PSScriptRoot

Push-Location (Join-Path $repositoryRoot 'server')
try {
    go generate ./...
    if ($LASTEXITCODE -ne 0) { throw 'Go contract generation failed.' }
}
finally {
    Pop-Location
}

Push-Location $repositoryRoot
try {
    pnpm exec openapi-typescript 'docs/api/openapi.yaml' --output 'web/types/openapi.d.ts'
    if ($LASTEXITCODE -ne 0) { throw 'TypeScript contract generation failed.' }
}
finally {
    Pop-Location
}
```

创建 `scripts/check-generated.ps1`，文件全文：

```powershell
[CmdletBinding()]
param()

$ErrorActionPreference = 'Stop'
$repositoryRoot = Split-Path -Parent $PSScriptRoot

& (Join-Path $PSScriptRoot 'generate.ps1')

Push-Location $repositoryRoot
try {
    git diff --exit-code -- 'server/generated/oapi/api.gen.go' 'web/types/openapi.d.ts'
    if ($LASTEXITCODE -ne 0) {
        throw 'Generated contract files are stale. Run scripts/generate.ps1 and commit the result.'
    }
}
finally {
    Pop-Location
}
```

执行：

```powershell
PS> .\scripts\generate.ps1
PS> .\scripts\check-generated.ps1
```

## 03-10 扩展 CI 契约 Job

打开 `.github/workflows/ci.yml`，在 `jobs:` 下、现有 `repository:` Job 同级追加以下完整 Job。注意 `contract:` 前有两个空格：

```yaml
  contract:
    name: Contract checks
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Check out repository
        uses: actions/checkout@v5

      - name: Lint OpenAPI
        uses: docker://dshanley/vacuum:0.30.0
        with:
          args: lint /github/workspace/docs/api/openapi.yaml

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: server/go.mod
          cache-dependency-path: server/go.sum

      - name: Set up Node.js
        uses: actions/setup-node@v5
        with:
          node-version-file: .node-version

      - name: Install pnpm
        run: corepack install --global pnpm@11.19.0

      - name: Install workspace dependencies
        run: pnpm install --frozen-lockfile

      - name: Download Go modules
        working-directory: server
        run: go mod download

      - name: Generate Go contract
        working-directory: server
        run: go generate ./...

      - name: Generate TypeScript contract
        run: pnpm exec openapi-typescript docs/api/openapi.yaml --output web/types/openapi.d.ts

      - name: Fail on generated drift
        run: git diff --exit-code -- server/generated/oapi/api.gen.go web/types/openapi.d.ts
```

YAML 不允许 Tab。保存后运行 Prettier：

```powershell
PS> pnpm format
PS> pnpm format:check
```

## 03-11 提交与 PR

先确认未残留示例域名：

```powershell
PS> Select-String -LiteralPath 'docs\api\openapi.yaml' -Pattern 'yourdomain|TODO|FIXME'
PS> git diff --check
PS> git status --short
```

预期第一条无输出。按职责拆成两个提交：

```powershell
PS> git add -- docs/api/openapi.yaml
PS> git commit -m 'feat(api): establish complete HTTP contract'
PS> git add -- server/go.mod server/go.sum server/generate.go server/oapi-codegen.yaml server/generated/oapi/api.gen.go web/types/openapi.d.ts package.json pnpm-lock.yaml scripts/generate.ps1 scripts/check-generated.ps1 .github/workflows/ci.yml
PS> git diff --cached --check
PS> git commit -m 'chore(api): generate typed contract clients'
PS> git push --set-upstream origin feat/openapi-contract
```

PR 中写清调用链：`OpenAPI -> oapi-codegen/openapi-typescript -> Handler interface and web types`。等待 Repository checks 和 Contract checks 通过后合并。

到 Settings -> Rulesets -> protect-main，把 `Contract checks` 加入 required status checks。

本地更新和删除分支采用第 01 章固定动作。

## 03-12 本章停止点

- [ ] OpenAPI lint 退出码 0；
- [ ] 47 个 operationId 无重复；
- [ ] Go/TypeScript 文件由命令生成；
- [ ] 连续生成无 diff；
- [ ] 两个 CI Job 通过；
- [ ] main 分支规则要求 Contract checks；
- [ ] 没有 `yourdomain`、TODO 或手改生成文件。
