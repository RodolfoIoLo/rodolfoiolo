# 第 09 章：首次部署与 MVP 验收

本章把第 08 章已经完成的公共站点第一次部署到阿里云 ECS。此时后台尚未制作，所以只验收公共页面、API、数据库、HTTPS、备份入口和回滚入口。第一次上线采用“服务器从已审核源码构建”的透明流程；第 13 章再升级为 CI 构建不可变镜像和制品。

## 09-01 冻结部署参数

回到第 00 章参数表，确认以下值已经填写：`GITHUB_OWNER`、`REPOSITORY_NAME`、`SITE_DOMAIN`、`ECS_PUBLIC_IP`、`ECS_SSH_USER`、`DEPLOY_ROOT`。本文假定 `DEPLOY_ROOT=/opt/personal-site`。若你的值不同，后续所有命令统一替换，不能一半使用旧值。

在阿里云控制台依次完成：

1. ECS 控制台 -> 实例 -> 创建实例；地域选择离主要访客最近的中国大陆或海外地域，系统选择 Ubuntu 24.04 LTS，最低 2 vCPU/2 GiB，系统盘 40 GiB ESSD；
2. 安全组只开放入方向 TCP `22`、`80`、`443`；PostgreSQL 的 `5432` 不开放；
3. 域名控制台 -> DNS 解析 -> 添加 `@` 和 `www` 两条 A 记录，值均为 ECS 公网 IPv4；
4. 若服务器位于中国大陆，先完成 ICP 备案；未完成前不要用 80/443 对公众提供网站；
5. 在 GitHub 仓库 Settings -> Deploy keys -> Add deploy key，稍后粘贴服务器公钥，只勾选只读，不勾选写权限。

在本机验证 DNS；将参数替换为第 00 章真实域名：

```powershell
PS> Resolve-DnsName '<SITE_DOMAIN>' -Type A
PS> Resolve-DnsName "www.<SITE_DOMAIN>" -Type A
```

两次结果必须包含 `ECS_PUBLIC_IP`。DNS 未生效时停在本步骤。

## 09-02 创建部署分支和目录

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main
PS> git pull --ff-only
PS> git status --short --branch
PS> git switch -c feat/production-mvp
PS> New-Item -ItemType Directory -Force -Path 'deploy\nginx\conf.d','deploy\certbot','scripts' | Out-Null
```

## 09-03 创建 API 多阶段镜像

创建 `server/Dockerfile`，文件全文：

```dockerfile
# syntax=docker/dockerfile:1.12
FROM golang:1.25.3-bookworm AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api \
 && CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/admin ./cmd/admin

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build /out/api /app/api
COPY --from=build /out/admin /app/admin
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/api"]
```

创建 `server/.dockerignore`，文件全文：

```dockerignore
.git
coverage.out
*.test
tmp
uploads
secrets
```

这里的运行镜像没有 shell、编译器和包管理器，进程也不是 root。迁移不塞进 API 启动命令，而由独立的一次性容器执行。

## 09-04 创建生产环境变量模板

创建 `.env.production.example`，文件全文：

```dotenv
COMPOSE_PROJECT_NAME=personal-site
SITE_DOMAIN=example.com
DEPLOY_ROOT=/opt/personal-site
POSTGRES_DB=personal_site
POSTGRES_USER=personal_site
POSTGRES_PASSWORD=replace-with-a-random-32-character-value
DATABASE_MAX_CONNS=10
DATABASE_MIN_CONNS=1
DATABASE_TIMEOUT=5s
JWT_ISSUER=personal-site-api
JWT_AUDIENCE=personal-site-admin
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL=720h
REFRESH_COOKIE_NAME=personal_site_refresh
VISITOR_COOKIE_NAME=personal_site_visitor
```

这是键名模板，不含真实秘密。确认根 `.gitignore` 已包含 `.env.production`、`secrets/` 和 `uploads/`；若没有，在对应段落逐行补上。

## 09-05 创建生产 Compose

创建 `compose.prod.yml`，文件全文：

```yaml
name: ${COMPOSE_PROJECT_NAME}

services:
  postgres:
    image: postgres:16.9-bookworm
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks: [backend]
    security_opt: ["no-new-privileges:true"]

  migrate:
    image: migrate/migrate:v4.19.0
    profiles: [tools]
    volumes:
      - ./server/migrations:/migrations:ro
    networks: [backend]
    command:
      - -path=/migrations
      - -database=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
      - up

  api:
    build:
      context: ./server
      dockerfile: Dockerfile
    image: personal-site-api:${RELEASE_SHA:-local}
    restart: unless-stopped
    init: true
    environment:
      APP_ENV: production
      HTTP_ADDRESS: :8080
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
      DATABASE_MAX_CONNS: ${DATABASE_MAX_CONNS}
      DATABASE_MIN_CONNS: ${DATABASE_MIN_CONNS}
      DATABASE_TIMEOUT: ${DATABASE_TIMEOUT}
      PUBLIC_SITE_URL: https://${SITE_DOMAIN}
      ALLOWED_ORIGINS: https://${SITE_DOMAIN},https://www.${SITE_DOMAIN}
      JWT_PRIVATE_KEY_FILE: /run/secrets/jwt_private_key.pem
      JWT_PUBLIC_KEY_FILE: /run/secrets/jwt_public_key.pem
      JWT_ISSUER: ${JWT_ISSUER}
      JWT_AUDIENCE: ${JWT_AUDIENCE}
      ACCESS_TOKEN_TTL: ${ACCESS_TOKEN_TTL}
      REFRESH_TOKEN_TTL: ${REFRESH_TOKEN_TTL}
      REFRESH_COOKIE_NAME: ${REFRESH_COOKIE_NAME}
      VISITOR_COOKIE_NAME: ${VISITOR_COOKIE_NAME}
      VISITOR_SECRET_FILE: /run/secrets/visitor_secret.txt
      COOKIE_SECURE: "true"
      UPLOAD_ROOT: /var/lib/personal-site/uploads
      MEDIA_PUBLIC_URL: https://${SITE_DOMAIN}/uploads
    volumes:
      - ./secrets:/run/secrets:ro
      - ./uploads:/var/lib/personal-site/uploads
    depends_on:
      postgres:
        condition: service_healthy
    expose: ["8080"]
    healthcheck:
      test: ["CMD", "/app/api", "--healthcheck"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
    networks: [backend]
    security_opt: ["no-new-privileges:true"]
    read_only: true
    tmpfs: ["/tmp:rw,noexec,nosuid,size=16m"]

  nginx:
    image: nginx:1.28.0-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./deploy/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./deploy/nginx/conf.d:/etc/nginx/templates:ro
      - ./releases:/srv/releases:ro
      - ./uploads:/srv/uploads:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    environment:
      SITE_DOMAIN: ${SITE_DOMAIN}
    depends_on:
      api:
        condition: service_healthy
    networks: [backend]
    security_opt: ["no-new-privileges:true"]

  certbot:
    image: certbot/certbot:v4.0.0
    profiles: [tools]
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt

volumes:
  postgres_data:

networks:
  backend:
    driver: bridge
```

注意：如果第 05 章最终的 API 没有实现 `--healthcheck` 参数，则上面的 distroless 容器不能用 `curl`。在首次执行本章前，把 `api.healthcheck` 替换成 Compose 从同网络探测的独立 sidecar 并不划算；本任务书统一要求在第 09-06 增加该参数。

## 09-06 给 API 增加容器健康检查参数

将 `server/cmd/api/main.go` 中现有 `main()` 改名为 `runServer()`，保留函数体不变；在 imports 中加入 `net/http` 和 `os`，再在文件末尾加入以下完整函数：

```go
func main() {
	if len(os.Args) == 2 && os.Args[1] == "--healthcheck" {
		response, err := http.Get("http://127.0.0.1:8080/readyz")
		if err != nil {
			os.Exit(1)
		}
		defer response.Body.Close()
		if response.StatusCode != http.StatusOK {
			os.Exit(1)
		}
		return
	}
	runServer()
}
```

运行格式和测试：

```powershell
PS> Set-Location 'server'
PS> gofmt -w cmd/api/main.go
PS> go test ./...
PS> go build ./cmd/api
PS> Set-Location '..'
```

如果原文件已经导入 `net/http` 或 `os`，不要重复写 import。编译器通过才继续。

## 09-07 创建 Nginx 主配置

创建 `deploy/nginx/nginx.conf`，文件全文：

```nginx
user nginx;
worker_processes auto;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    charset utf-8;
    server_tokens off;
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    client_max_body_size 12m;

    log_format json escape=json '{"time":"$time_iso8601","remote":"$remote_addr","request":"$request","status":$status,"bytes":$body_bytes_sent,"request_time":$request_time,"request_id":"$request_id","user_agent":"$http_user_agent"}';
    access_log /var/log/nginx/access.log json;
    error_log /var/log/nginx/error.log warn;

    limit_req_zone $binary_remote_addr zone=api:10m rate=20r/s;
    gzip on;
    gzip_types text/plain text/css application/json application/javascript application/xml image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
}
```

创建 `deploy/nginx/conf.d/site.conf.template`，文件全文：

```nginx
map $sent_http_content_type $cache_control {
    default                         "public, max-age=300";
    "text/html"                     "no-cache";
    "application/json"              "no-store";
}

server {
    listen 80;
    listen [::]:80;
    server_name ${SITE_DOMAIN} www.${SITE_DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://${SITE_DOMAIN}$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name ${SITE_DOMAIN};

    ssl_certificate /etc/letsencrypt/live/${SITE_DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${SITE_DOMAIN}/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; script-src 'self'; connect-src 'self'; font-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'; upgrade-insecure-requests" always;

    root /srv/releases/current/web;
    index index.html;

    location /api/v1/ {
        limit_req zone=api burst=40 nodelay;
        proxy_pass http://api:8080/api/v1/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;
    }

    location /uploads/ {
        alias /srv/uploads/;
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;
    }

    location /_next/static/ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;
    }

    location / {
        add_header Cache-Control $cache_control;
        try_files $uri $uri/ $uri.html =404;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.${SITE_DOMAIN};
    ssl_certificate /etc/letsencrypt/live/${SITE_DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${SITE_DOMAIN}/privkey.pem;
    return 301 https://${SITE_DOMAIN}$request_uri;
}
```

## 09-08 首次 TLS 启动文件

正式配置在证书签发前无法启动。创建 `deploy/nginx/conf.d/bootstrap.conf.disabled`，文件全文：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name ${SITE_DOMAIN} www.${SITE_DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 "TLS bootstrap in progress\n";
        add_header Content-Type text/plain;
    }
}
```

扩展名 `.disabled` 使 Nginx 模板机制忽略它。部署时会临时改名，证书签发后恢复。

## 09-09 本地生产形态检查

生产 Compose 展开时需要环境文件。复制模板并只填写本地临时值：

```powershell
PS> Copy-Item '.env.production.example' '.env.production'
PS> $passwordBytes = New-Object byte[] 32
PS> [Security.Cryptography.RandomNumberGenerator]::Fill($passwordBytes)
PS> $password = [Convert]::ToHexString($passwordBytes).ToLowerInvariant()
PS> (Get-Content '.env.production' -Raw).Replace('replace-with-a-random-32-character-value',$password) | Set-Content '.env.production' -Encoding utf8NoBOM
PS> docker compose --env-file .env.production -f compose.prod.yml config --quiet
PS> docker build --pull -t personal-site-api:local ./server
PS> docker image inspect personal-site-api:local --format '{{.Config.User}}'
PS> Remove-Item '.env.production'
```

最后一项必须输出 `nonroot:nonroot`。确认 `.env.production` 删除且 `git status --short` 中没有它。

## 09-10 提交部署基线

```powershell
PS> git diff --check
PS> git add -- server/Dockerfile server/.dockerignore server/cmd/api/main.go .env.production.example compose.prod.yml deploy/nginx
PS> git diff --cached --stat
PS> git commit -m 'ops: add production compose and nginx baseline'
PS> git push --set-upstream origin feat/production-mvp
```

在 GitHub 创建 PR。确认 Go CI、Frontend checks、OpenAPI 检查全绿，自审容器未开放 5432、真实秘密未提交、API 非 root 后合并。然后本机执行：

```powershell
PS> git switch main
PS> git pull --ff-only
PS> git branch -d feat/production-mvp
```

## 09-11 初始化 ECS

从本机连接；首次提示主机指纹时，到 ECS 控制台核对实例指纹后输入 `yes`：

```powershell
PS> ssh <ECS_SSH_USER>@<ECS_PUBLIC_IP>
```

以下命令在 ECS 的 Bash 中执行，不输入 `$`：

```bash
$ sudo apt-get update
$ sudo apt-get install -y ca-certificates curl git openssl ufw
$ curl -fsSL https://get.docker.com | sudo sh
$ sudo usermod -aG docker "$USER"
$ sudo systemctl enable --now docker
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow OpenSSH
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443/tcp
$ sudo ufw --force enable
$ exit
```

重新 SSH 登录，验证：

```bash
$ docker version
$ docker compose version
$ sudo ufw status verbose
```

若 Docker 命令仍提示权限错误，退出 SSH 再登录一次，不使用长期 `sudo docker`。

## 09-12 配置只读仓库访问

在 ECS 执行：

```bash
$ install -d -m 700 ~/.ssh
$ ssh-keygen -t ed25519 -C "personal-site-deploy" -f ~/.ssh/personal_site_deploy -N ""
$ cat ~/.ssh/personal_site_deploy.pub
```

把输出整行添加到本章 09-01 的 GitHub Deploy key。然后创建 SSH 配置：

```bash
$ printf 'Host github.com\n  HostName github.com\n  User git\n  IdentityFile ~/.ssh/personal_site_deploy\n  IdentitiesOnly yes\n' > ~/.ssh/config
$ chmod 600 ~/.ssh/config
$ ssh -T git@github.com
```

GitHub 会返回“successfully authenticated”且说明不提供 shell，这是成功。继续：

```bash
$ sudo install -d -o "$USER" -g "$USER" -m 750 /opt/personal-site
$ git clone git@github.com:<GITHUB_OWNER>/<REPOSITORY_NAME>.git /opt/personal-site/repository
$ cd /opt/personal-site/repository
$ git rev-parse --verify HEAD
```

## 09-13 创建服务器秘密和目录

在 ECS 执行：

```bash
$ cd /opt/personal-site/repository
$ install -d -m 700 secrets
$ install -d -m 750 uploads releases certbot/www certbot/conf
$ docker run --rm -v "$PWD/secrets:/work" -w /work personal-site-api:local --help
```

上面的最后一条只有本地已存在镜像时才可执行；首次服务器尚未构建镜像，因此直接使用 OpenSSL：

```bash
$ openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out secrets/jwt_private_key.pem
$ openssl pkey -in secrets/jwt_private_key.pem -pubout -out secrets/jwt_public_key.pem
$ openssl rand -hex 32 > secrets/visitor_secret.txt
$ chmod 600 secrets/jwt_private_key.pem secrets/jwt_public_key.pem secrets/visitor_secret.txt
$ cp .env.production.example .env.production
$ openssl rand -hex 32
```

复制最后输出，在 `nano .env.production` 中只替换 `POSTGRES_PASSWORD`；再把 `SITE_DOMAIN` 替换成第 00 章域名。保存方式：`Ctrl+O`、Enter、`Ctrl+X`。

```bash
$ chmod 600 .env.production
$ grep -E '^(SITE_DOMAIN|DEPLOY_ROOT|POSTGRES_DB|POSTGRES_USER)=' .env.production
$ docker compose --env-file .env.production -f compose.prod.yml config --quiet
```

不要输出 `POSTGRES_PASSWORD`，不要对 `.env.production` 使用 `cat`。

## 09-14 启动数据库、迁移和 API

```bash
$ cd /opt/personal-site/repository
$ export RELEASE_SHA="$(git rev-parse --short=12 HEAD)"
$ docker compose --env-file .env.production -f compose.prod.yml pull postgres migrate nginx certbot
$ docker compose --env-file .env.production -f compose.prod.yml build --pull api
$ docker compose --env-file .env.production -f compose.prod.yml up -d postgres
$ docker compose --env-file .env.production -f compose.prod.yml run --rm migrate
$ docker compose --env-file .env.production -f compose.prod.yml up -d api
$ docker compose --env-file .env.production -f compose.prod.yml ps
$ docker compose --env-file .env.production -f compose.prod.yml logs --tail=100 api
```

`postgres` 与 `api` 都必须 healthy，迁移容器退出码必须为 0，日志不能出现私钥、密码或完整 Cookie。

## 09-15 在服务器构建静态站点

API 只在 Compose 网络内，不映射宿主端口。先临时启动一次性 Node 构建容器，并把网络内 API 地址传给构建：

```bash
$ cd /opt/personal-site/repository
$ RELEASE_SHA="$(git rev-parse --short=12 HEAD)"
$ docker run --rm --network personal-site_backend \
  -e CONTENT_API_BASE_URL=http://api:8080/api/v1 \
  -e NEXT_PUBLIC_API_BASE_URL=https://$SITE_DOMAIN/api/v1 \
  -e NEXT_PUBLIC_SITE_URL=https://$SITE_DOMAIN \
  -v "$PWD:/workspace" -w /workspace \
  node:24.14.0-bookworm bash -lc 'corepack install --global pnpm@11.19.0 && pnpm install --frozen-lockfile && pnpm --dir web build'
$ install -d -m 755 "releases/$RELEASE_SHA/web"
$ cp -a web/out/. "releases/$RELEASE_SHA/web/"
$ ln -sfn "$RELEASE_SHA" releases/current
$ test -s releases/current/web/index.html
$ test -s releases/current/web/rss.xml
```

这里的 `$SITE_DOMAIN` 来自当前 shell；若 `echo "$SITE_DOMAIN"` 为空，执行 `set -a; source .env.production; set +a` 后再运行构建命令。不要把 `.env.production` source 输出到日志。

## 09-16 首次签发证书

先临时启用 HTTP 配置：

```bash
$ cd /opt/personal-site/repository
$ mv deploy/nginx/conf.d/site.conf.template deploy/nginx/conf.d/site.conf.template.disabled
$ cp deploy/nginx/conf.d/bootstrap.conf.disabled deploy/nginx/conf.d/bootstrap.conf.template
$ docker compose --env-file .env.production -f compose.prod.yml up -d nginx
$ curl -I "http://$SITE_DOMAIN/"
$ docker compose --env-file .env.production -f compose.prod.yml run --rm certbot certonly \
  --webroot -w /var/www/certbot \
  -d "$SITE_DOMAIN" -d "www.$SITE_DOMAIN" \
  --email '<TLS_CONTACT_EMAIL>' --agree-tos --no-eff-email
$ rm deploy/nginx/conf.d/bootstrap.conf.template
$ mv deploy/nginx/conf.d/site.conf.template.disabled deploy/nginx/conf.d/site.conf.template
$ docker compose --env-file .env.production -f compose.prod.yml up -d --force-recreate nginx
```

把 `<TLS_CONTACT_EMAIL>` 替换为第 00 章登记的真实联系邮箱。`rm` 只删除刚复制的临时模板；执行前先运行 `realpath deploy/nginx/conf.d/bootstrap.conf.template`，输出必须位于 `/opt/personal-site/repository/deploy/nginx/conf.d/`。

## 09-17 配置证书自动续期

在 ECS 执行：

```bash
$ sudo systemctl edit --force --full personal-site-certbot.service
```

填入文件全文：

```ini
[Unit]
Description=Renew personal-site TLS certificate
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
WorkingDirectory=/opt/personal-site/repository
ExecStart=/usr/bin/docker compose --env-file .env.production -f compose.prod.yml run --rm certbot renew --webroot -w /var/www/certbot --quiet
ExecStartPost=/usr/bin/docker compose --env-file .env.production -f compose.prod.yml exec -T nginx nginx -s reload
```

再创建 timer：

```bash
$ sudo systemctl edit --force --full personal-site-certbot.timer
```

填入：

```ini
[Unit]
Description=Run personal-site certificate renewal twice daily

[Timer]
OnCalendar=*-*-* 03,15:17:00
RandomizedDelaySec=30m
Persistent=true

[Install]
WantedBy=timers.target
```

启用并做 dry run：

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now personal-site-certbot.timer
$ sudo systemctl list-timers personal-site-certbot.timer
$ docker compose --env-file .env.production -f compose.prod.yml run --rm certbot renew --dry-run
```

## 09-18 创建首个管理员

管理员命令从终端安全读取密码。先检查第 06 章 `server/cmd/admin` 的帮助，再执行：

```bash
$ cd /opt/personal-site/repository
$ set -a; source .env.production; set +a
$ docker run --rm -it --network personal-site_backend \
  --env-file .env.production \
  -e DATABASE_URL="postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable" \
  personal-site-api:$RELEASE_SHA /app/admin --username '<ADMIN_USERNAME>' --nickname '<DISPLAY_NAME>' --email '<ADMIN_EMAIL>' --role admin
```

把三个尖括号参数替换为第 00 章非秘密参数表中的值。命令随后在 TTY 中安全读取密码；不要把密码改成命令行参数。

## 09-19 外部 MVP 验收

回到本机 PowerShell：

```powershell
PS> $domain = '<SITE_DOMAIN>'
PS> curl.exe --fail --silent --show-error "https://$domain/" --output "$env:TEMP\personal-site-home.html"
PS> curl.exe --fail --silent --show-error "https://$domain/api/v1/healthz"
PS> curl.exe --fail --silent --show-error "https://$domain/api/v1/readyz"
PS> curl.exe --fail --silent --show-error "https://$domain/rss.xml" --output "$env:TEMP\personal-site-rss.xml"
PS> curl.exe --fail --silent --show-error "https://$domain/sitemap.xml" --output "$env:TEMP\personal-site-sitemap.xml"
PS> curl.exe -I "http://$domain/"
PS> curl.exe -I "https://www.$domain/"
PS> curl.exe -I "https://$domain/"
```

验收：HTTP 与 `www` 都 301 到唯一 HTTPS 域名；主页 200；healthz/readyz 200；RSS/Sitemap 非空；响应有 HSTS、CSP、nosniff；证书域名匹配且浏览器无警告。再用浏览器检查首页、文章、项目、归档、关于、375px、键盘 Tab 和深色主题。

## 09-20 执行第一次回滚演练

在 ECS 记录当前 release，再创建一个故意不完整但不会切换的候选目录：

```bash
$ cd /opt/personal-site/repository
$ readlink releases/current
$ CURRENT_RELEASE="$(readlink releases/current)"
$ test -s "releases/$CURRENT_RELEASE/web/index.html"
$ docker compose --env-file .env.production -f compose.prod.yml ps
```

本章第一次上线只有一个 release，不能假装已经具备旧版本。真正的双版本回滚在第 14 章执行。此处只证明当前 symlink、镜像标签和数据库版本都可识别，并把输出写入本地学习日志，日志不得含秘密。

## 09-21 本章停止点

- [ ] DNS、ECS 安全组和 UFW 只开放 22/80/443；
- [ ] PostgreSQL 没有公网端口；
- [ ] API 非 root、只读根文件系统并通过健康检查；
- [ ] 数据库迁移成功且 API、PostgreSQL healthy；
- [ ] 静态 release 以 Git SHA 命名，`current` 指向有效版本；
- [ ] HTTPS、重定向、安全头、RSS 和 Sitemap 验收通过；
- [ ] Certbot dry run 和 systemd timer 正常；
- [ ] 真实 `.env.production`、密钥和上传文件未进入 Git；
- [ ] 首个管理员通过安全输入创建；
- [ ] 已记录当前 release、镜像和迁移版本，为最终回滚演练准备证据。
