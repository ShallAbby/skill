---
name: docker-compose-fullstack-deploy
description: >-
  架构自适应的全栈便携式 Docker Compose 部署方案生成器。支持根据用户的实际技术栈（如前端 React AntDesign / Vue，
  后端 Python FastAPI / Java SpringBoot / Node.js，中间件 MinIO / Elasticsearch / Redis / rNacos / PostgreSQL / MySQL，
  以及 AI 流式网关 SSE / Dify）动态拼装高内聚、多环境隔离、配置数据全外挂且带前后端秒级热更新与备份回滚脚本的部署包。
---

# 架构自适应全栈 Docker Compose 便携式部署生成指南

本技能将成熟的**容器化工程底座范式**与**可插拔的技术栈组件（积木块）**解耦。无论用户的项目是 `Python FastAPI + React (Ant Design) + MinIO + Elasticsearch`，还是 `Java SpringBoot + Vue + Redis + rNacos`，均使用统一的目录哲学、环境变量隔离机制、网关规范与热更新脚本体系，按用户的实际架构按需拼装生成。

---

## 一、 跨架构通用的 6 大核心工程范式（黄金法则）

无论底层技术栈如何变化，生成的所有部署方案必须遵循以下 6 条核心设计原则：

1. **`.env` 单点收敛与多环境前缀隔离**：
   - 使用 `CONTAINER_PREFIX={{PROJECT_NAME}}-{{ENV_NAME}}` 统一驱动所有容器名（`${CONTAINER_PREFIX}-<service>`）和自定义网络名（`${CONTAINER_PREFIX}-network`）。
   - 所有宿主机暴露端口、中间件密钥（如 `MINIO_ROOT_PASSWORD`、`ES_PASSWORD`、`REDIS_PASSWORD`）、JVM/内存限制参数全部收拢在 `.env` 中。同一台宿主机只需修改 `.env` 即可零冲突并行运行多套环境（`dev` / `test` / `stage` / `prod`）。
   - 在 `docker-compose.yml` 顶部定义 `x-shared-env: &shared-env` 统一注入 `TZ: Asia/Shanghai`。

2. **便携式自包含目录结构（配置、数据、日志全外挂）**：
   - `./conf/`：集中存放网关与各中间件配置文件（`nginx.conf`、SSL 证书、`redis.conf` 等），改配置无需进容器。
   - `./<service>/data` 与 `./<service>/logs`：所有有状态组件的数据（如 `./minio/data`、`./es/data`、`./redis/data`、`./nacos/data`）与日志全部以相对路径挂载在当前项目目录下。**整个项目文件夹即是一个高内聚的“便携式部署包”**，整体打包即可完成备份或跨机迁移。

3. **容器内部服务名解耦与宿主机网络打通**：
   - 容器间通过 `${CONTAINER_PREFIX}-network` 通信，Nginx 反代及后端连接中间件时统一使用 **Compose 内部服务名**（如 `http://{{PROJECT_NAME}}-server:{{SERVER_CONTAINER_PORT}}`、`{{PROJECT_NAME}}-minio:9000`、`{{PROJECT_NAME}}-es:9200`），修改 `.env` 中的 `CONTAINER_PREFIX` 时无需同步改动 `nginx.conf` 或后端配置。
   - 为 Nginx 与后端容器统一配置 `extra_hosts: - "host.docker.internal:host-gateway"`，确保容器内可通过 `host.docker.internal` 稳定访问宿主机外部服务（如宿主机本地大模型、Dify、外部数据库）。

4. **实战化 Nginx 统一网关**：
   - **HTTP + 自签名 HTTPS 双端口监听**：解决内网 IP 部署时现代浏览器限制非 HTTPS 安全上下文调用剪贴板、麦克风、WebCrypto 等 API 的痛点。
   - **SPA 路由与静态缓存分级**：适配 React (Ant Design) / Vue 单页应用 `try_files $uri $uri/ /index.html;`；对静态构建产物开启 `30d` 强缓存，对入口或动态覆盖资源（`/custom-assets`）禁用缓存。
   - **AI 流式（SSE）与大文件/WebSocket 专项优化**：
     - 对 FastAPI `StreamingResponse` 或 `/llm-api/` 路由开启 `proxy_buffering off;` 及 CORS 预检，保障流式输出零延迟。
     - 对业务 API 开启 WebSocket `Upgrade` 与长超时（`86400s`）。
     - 若包含 MinIO 且需网关代理，配置 `client_max_body_size 100m;` 与 `proxy_request_buffering off;`。

5. **前后端免重编热更新与时间戳归档回滚（`deploy.sh` 闭环）**：
   - **前端（React / Vue）**：将 `./html/dist` 挂载进 Nginx 容器，配合 `html/deploy.sh` 实现**“旧版按时间戳归档至 `backup/dist_YYYY-MM-DD_HH-MM` $\rightarrow$ 解压 `dist.tgz` $\rightarrow$ 自动读取 `.env` 重启 Nginx”**。
   - **后端按语言自适应热更新**：
     - **Python (FastAPI)**：将 `requirements.txt` 构建进镜像层，同时将业务源码目录 `./server/app` 挂载进容器。日常发版只需上传 `app.tgz` 并执行 `server/deploy.sh`（自动备份旧代码 + 解压新代码 + 直接 `docker restart` 秒级生效）；仅当新增第三方依赖时执行 `server/deploy.sh --build`。
     - **Java (SpringBoot)**：将 `./jar/*.jar` 放于 `./jar/` 目录，配合 `jar/deploy.sh` 自动归档旧 Jar 包并重启/重编后端容器。

6. **中间件轻量化与安全加固基线**：
   - **Elasticsearch**：默认单节点模式（`discovery.type=single-node`），通过 `.env` 显式限制堆内存（如 `ES_MEM_LIMIT=512m` 或 `1g`）防止单机 OOM，同时挂载 `./es/data` 与 `./es/plugins`（方便放入 IK 分词器等插件）。
   - **MinIO**：分离 API 端口（`9000`）与 Web 控制台端口（`9001`），强制通过 `.env` 管理 `MINIO_ROOT_USER` 与 `MINIO_ROOT_PASSWORD`。
   - **Redis**：非标端口（如 `8761`）+ `requirepass` 密码 + `rename-command CONFIG ""` 禁用高危命令。
   - **rNacos**：需要注册/配置中心时，优先选用 Rust 轻量版 `qingpan/rnacos:stable`（内存仅数十 MB）。

---

## 二、 架构模块矩阵（根据用户需求动态组合）

在生成部署方案前，先识别或向用户确认其技术栈组合：

| 架构层次 | 可选模块 | 对应模板 / 参考片段 | 运行时挂载目录 |
| :--- | :--- | :--- | :--- |
| **前端网关层** | React (AntD) / Vue + Nginx | [nginx-dockerfile.template](./resources/dockerfiles/nginx-dockerfile.template)<br>[nginx.conf.template](./resources/conf/nginx.conf.template)<br>[html-deploy.sh.template](./resources/scripts/html-deploy.sh.template) | `./html/dist`<br>`./html/backup`<br>`./nginx/logs`<br>`./nginx/conf.d` |
| **后端服务层 (A)** | **Python (FastAPI / Uvicorn)** | [python-fastapi-dockerfile.template](./resources/dockerfiles/python-fastapi-dockerfile.template)<br>[python-deploy.sh.template](./resources/scripts/python-deploy.sh.template) | `./server/app`<br>`./server/backup`<br>`./{{PROJECT_NAME}}/logs`<br>`./{{PROJECT_NAME}}/uploadPath` |
| **后端服务层 (B)** | **Java (SpringBoot)** | [java-dockerfile.template](./resources/dockerfiles/java-dockerfile.template)<br>[java-deploy.sh.template](./resources/scripts/java-deploy.sh.template) | `./jar/backup`<br>`./{{PROJECT_NAME}}/logs`<br>`./{{PROJECT_NAME}}/uploadPath` |
| **对象存储** | **MinIO** | 见 [middleware-snippets.md](./references/middleware-snippets.md) `#MinIO` | `./minio/data`<br>`./minio/config` |
| **搜索引擎** | **Elasticsearch (7.x/8.x)** | 见 [middleware-snippets.md](./references/middleware-snippets.md) `#Elasticsearch` | `./es/data`<br>`./es/plugins`<br>`./es/logs` |
| **缓存中间件** | **Redis (安全加固版)** | [redis-dockerfile.template](./resources/dockerfiles/redis-dockerfile.template)<br>[redis.conf.template](./resources/conf/redis.conf.template)<br>见 [middleware-snippets.md](./references/middleware-snippets.md) `#Redis` | `./redis/data`<br>`./conf/redis.conf` |
| **注册/配置中心** | **rNacos (Rust 轻量版)** | 见 [middleware-snippets.md](./references/middleware-snippets.md) `#rNacos` | `./nacos/data` |
| **关系型数据库** | **PostgreSQL / MySQL** | 见 [middleware-snippets.md](./references/middleware-snippets.md) `#Database` | `./db/data`<br>`./conf/init.sql` |

---

## 三、 执行步骤（Workflow）

### 第 1 步：确定项目架构与核心参数
根据用户的描述提取架构组件与以下基础参数（未指定时采用合理默认值）：
- `{{PROJECT_NAME}}`：项目英文标识（小写，如 `myproj`、`gemr`）
- `{{ENV_NAME}}`：环境名（默认 `dev`）
- `{{NGINX_HTTP_PORT}}` / `{{NGINX_HTTPS_PORT}}`：宿主机前端 HTTP/HTTPS 端口（默认 `8084` / `8085`）
- `{{SERVER_HOST_PORT}}` / `{{SERVER_CONTAINER_PORT}}`：后端宿主机端口与容器内端口（FastAPI 默认容器端口 `8000`，SpringBoot 默认 `8186`）
- **所选中间件端口与凭据**：仅纳入用户架构中需要的中间件变量（详见 [.env.base.template](./resources/.env.base.template)）。

### 第 2 步：按需拼装 `.env` 与 `docker-compose.yml`
1. 读取 [.env.base.template](./resources/.env.base.template) 与 [docker-compose.base.yml.template](./resources/docker-compose.base.yml.template)。
2. 读取 [middleware-snippets.md](./references/middleware-snippets.md)，将用户需要的中间件（如 `MinIO`、`Elasticsearch`、`Redis`、`rNacos`、`PostgreSQL/MySQL`）对应的 `.env` 变量块、Compose `services` 块及 `depends_on` 依赖组装进主文件中；**剔除用户未使用的中间件**。

### 第 3 步：生成匹配的 Dockerfile 与热发版脚本
- **前端（必选）**：
  - 生成 `nginx-dockerfile`、`html/deploy.sh`、`html/backup.sh`、`html/restart.sh`。
- **后端（按语言二选一或多选）**：
  - **若为 Python FastAPI**：使用 [python-fastapi-dockerfile.template](./resources/dockerfiles/python-fastapi-dockerfile.template) 生成 `{{PROJECT_NAME}}-dockerfile`，并使用 [python-deploy.sh.template](./resources/scripts/python-deploy.sh.template) 生成 `server/deploy.sh`（支持 `app.tgz` 免重编热更新与 `--build` 依赖重编）。同时生成示例 `server/requirements.txt`（若尚不存在）。
  - **若为 Java SpringBoot**：使用 [java-dockerfile.template](./resources/dockerfiles/java-dockerfile.template) 生成 `{{PROJECT_NAME}}-dockerfile`，并使用 [java-deploy.sh.template](./resources/scripts/java-deploy.sh.template) 生成 `jar/deploy.sh`。

### 第 4 步：生成 `conf/nginx.conf` 与中间件配置
1. 读取 [nginx.conf.template](./resources/conf/nginx.conf.template)，将 API 路由（如 `/api/`、`/prod-api/`、`/docs`、`/openapi.json`）指向 `http://{{PROJECT_NAME}}-server:{{SERVER_CONTAINER_PORT}}/`。
2. 若包含 **MinIO**，按需在 `nginx.conf` 中保留 `/oss/`（或 `/minio/`）对象存储反代块（见 [middleware-snippets.md](./references/middleware-snippets.md)）。
3. 若包含 **Redis**，生成 `redis-dockerfile` 与 `conf/redis.conf`。
4. 若需要自签名证书，提示或使用 `openssl` 生成 `conf/self-sign.cer` 与 `conf/private.key`。

### 第 5 步：创建持久化空目录并校验
1. 创建所有被挂载的宿主机目录（如 `html/dist`、`html/backup`、`nginx/logs`、`nginx/conf.d`、`{{PROJECT_NAME}}/logs`、`{{PROJECT_NAME}}/uploadPath`，以及所选中间件的 `minio/data`、`es/data`、`es/plugins` 等）。
2. **特别提示（若包含 Elasticsearch）**：在部署说明中提醒用户，Linux 宿主机首次启动 ES 前需确保 `./es/data` 和 `./es/logs` 目录权限为 `777` 或归属 `1000:1000`（`chmod -R 777 ./es`），避免 ES 容器因权限不足启动失败。
3. 检查 `.env`、`docker-compose.yml` 与 `conf/nginx.conf` 中的服务名和端口映射完全对齐。
