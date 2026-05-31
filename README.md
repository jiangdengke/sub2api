# Sub2API Fork

这是一个基于 `Wei-Shaw/sub2api` 的自用分支。当前 README 只保留部署、数据备份、已关注功能和后续计划，避免维护过长的上游介绍内容。

## 项目定位

Sub2API 是一个 AI API 网关和配额分发平台，核心用途是把多个上游 AI 账号或 API Key 统一接入，然后对外提供兼容接口、用户 API Key、用量统计和后台管理。

## 待实现功能

- [ ] 上传 `.sql.gz` 备份文件并导入为可恢复记录。

## 快速部署

推荐使用 Docker Compose 本地目录版，数据目录都在 `deploy/` 下，方便迁移和打包。

```bash
git clone <your-fork-url> sub2api
cd sub2api/deploy

cp .env.example .env
mkdir -p data postgres_data redis_data
```

编辑 `.env`，至少修改这些值：

```bash
POSTGRES_PASSWORD=<生成一个强密码>
JWT_SECRET=<openssl rand -hex 32>
TOTP_ENCRYPTION_KEY=<openssl rand -hex 32>
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=<管理员密码>
SERVER_PORT=8080
```

启动：

```bash
docker compose -f docker-compose.local.yml up -d
docker compose -f docker-compose.local.yml logs -f sub2api
```

访问：

```text
http://服务器 IP:8080
```

停止：

```bash
docker compose -f docker-compose.local.yml down
```

升级镜像：

```bash
docker compose -f docker-compose.local.yml pull
docker compose -f docker-compose.local.yml up -d
```

## 从源码构建部署

如果要运行本 fork 的本地改动，可以使用开发 Compose：

```bash
cd deploy
cp .env.example .env
mkdir -p data postgres_data redis_data

docker compose -f docker-compose.dev.yml up --build -d
```

本地构建会从当前仓库源码构建镜像，适合验证前后端改动。

## Docker 镜像和 Packages

GitHub 仓库右侧的 `Packages` 通常是仓库发布的包。对这个项目来说，最可能就是 Docker/OCI 镜像，例如发布到 GitHub Container Registry 后会显示为 Package。

本仓库已经带有 release workflow。推送 `v*` tag 或手动运行 `Release` workflow 后，会构建并发布 fork 自己的镜像：

```text
ghcr.io/jiangdengke/sub2api:<version>
ghcr.io/jiangdengke/sub2api:latest
```

发布示例：

```bash
git tag v0.1.133
git push origin v0.1.133
```

镜像发布成功后，GitHub 仓库右侧会出现对应 Package。

当前部署文件默认仍使用上游镜像：

```yaml
image: weishaw/sub2api:latest
```

这意味着：

- 直接用 `docker-compose.local.yml` 部署时，会拉取上游发布的 Docker 镜像。
- 用 `docker-compose.dev.yml` 部署时，会从当前 fork 源码本地构建镜像。
- 如果要部署 fork 发布的镜像，需要把 Compose 里的 `image` 改成 `ghcr.io/jiangdengke/sub2api:latest`。

## 数据和备份

Docker 本地目录版的数据位置：

- 应用数据：`deploy/data`
- PostgreSQL：`deploy/postgres_data`
- Redis：`deploy/redis_data`

整机迁移时，可以先停服务，再打包 `deploy/` 目录：

```bash
cd ..
docker compose -f deploy/docker-compose.local.yml down
tar czf sub2api-deploy.tar.gz deploy/
```

当前备份功能以 PostgreSQL 为主，备份文件格式为 `.sql.gz`。后续重点补齐“上传 `.sql.gz` 备份文件并导入为可恢复记录”，用于迁移或恢复外部备份。

## 当前要实现的功能

详见 [plan.md](./plan.md)。当前只聚焦一个功能：

- [ ] 在管理后台上传 `.sql.gz` 备份文件，上传后登记为可恢复记录，并复用现有恢复流程。

## 开发命令

后端：

```bash
cd backend
go test ./...
go build -o bin/server ./cmd/server
```

前端：

```bash
cd frontend
pnpm install
pnpm run build
pnpm run test:run
```

根目录 Makefile：

```bash
make build
make test
```

## 目录说明

- `backend/`：Go 后端、Ent schema、迁移、网关和管理 API。
- `frontend/`：Vue 管理后台和用户前台。
- `deploy/`：Docker Compose、配置样例和部署脚本。
- `docs/`：补充文档。
- `plan.md`：本 fork 的功能计划。

## 备注

这个分支以自用部署和备份/迁移能力增强为主。如果需要上游完整说明，可以查看 `README_CN.md` 或上游仓库文档。
