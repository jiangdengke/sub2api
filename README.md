# Sub2API Fork

这是一个基于 `Wei-Shaw/sub2api` 的自用分支。当前 README 只保留部署、数据备份、已关注功能和后续计划，避免维护过长的上游介绍内容。

## 项目定位

Sub2API 是一个 AI API 网关和配额分发平台，核心用途是把多个上游 AI 账号或 API Key 统一接入，然后对外提供兼容接口、用户 API Key、用量统计和后台管理。

当前主要能力：

- OpenAI、Claude、Gemini、Antigravity 等网关兼容接口。
- 用户、分组、账号、API Key 管理。
- 用量统计、余额/订阅、支付和兑换码。
- 管理后台、渠道监控、运维监控。
- PostgreSQL 数据库备份到 S3 兼容对象存储。

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

当前内置备份功能：

- 后台可配置 S3/R2/OSS 等 S3 兼容存储。
- 手动或定时执行 PostgreSQL 全量备份。
- 备份文件格式为 `.sql.gz`。
- 可从已登记的备份记录恢复数据库。

当前限制：

- 备份主要覆盖 PostgreSQL，不覆盖 Redis 和完整 `deploy/data`。
- 只能恢复系统自己创建并登记过的备份记录。
- 暂时没有“导入已有备份文件”按钮，这是后续优先改造项。

## 当前要实现的功能

详见 [plan.md](./plan.md)。当前优先级：

1. 增加“导入已有 S3 备份”功能：填写已有 `s3_key`，校验对象存在后登记为可恢复备份。
2. 增加备份导入 UI：在数据库备份页面新增“导入备份”按钮和弹窗。
3. 保持现有恢复安全校验：恢复仍需要管理员重新输入密码。
4. 补齐后端服务测试和前端交互测试。

## 预计实现的功能

后续计划按风险从低到高推进：

1. 本地 `.sql.gz` 文件上传并登记为备份。
2. 恢复前自动创建一次当前数据库备份。
3. 恢复审计：记录操作人、开始时间、结束时间和失败原因。
4. 备份上传改为真正流式或分片上传，避免大备份占用过多内存。
5. Redis 和应用数据目录备份。
6. 一键迁移包：PostgreSQL、Redis、应用数据统一导出和导入。

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
