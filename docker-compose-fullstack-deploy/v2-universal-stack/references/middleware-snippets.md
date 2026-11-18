# 可插拔中间件与网关配置片段库 (Middleware & Gateway Snippets)

在根据用户的技术栈拼装 `.env`、`docker-compose.yml` 和 `conf/nginx.conf` 时，从本文档中选取对应的模块片段插入到基础骨架中。

---

## 1. MinIO 对象存储 (`#MinIO`)

适用于需要文件存储、图片/音视频资源管理、知识库文档存储的项目（如 Python FastAPI + React + MinIO）。

### 1.1 `.env` 变量片段
```dotenv
# MinIO 对象存储配置
MINIO_API_PORT=9000
MINIO_CONSOLE_PORT=9001
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD={{PROJECT_NAME}}#Minio2026!
```

### 1.2 `docker-compose.yml` 服务片段
```yaml
  {{PROJECT_NAME}}-minio:
    image: minio/minio:latest
    container_name: ${CONTAINER_PREFIX}-minio
    environment:
      <<: *shared-env
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    ports:
      - "${MINIO_API_PORT}:9000"
      - "${MINIO_CONSOLE_PORT}:9001"
    volumes:
      - ./minio/data:/data
      - ./minio/config:/root/.minio
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    networks:
      - {{PROJECT_NAME}}-network
    restart: unless-stopped
```

### 1.3 `conf/nginx.conf` 反代片段（可选：通过统一网关暴露桶文件或预签名上传）
```nginx
        # MinIO 对象存储 API / 静态桶资源反代
        location /oss/ {
            client_max_body_size 500m;
            proxy_buffering off;
            proxy_request_buffering off;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 300;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;
            proxy_pass http://{{PROJECT_NAME}}-minio:9000/;
        }
```

---

## 2. Elasticsearch 搜索引擎 (`#Elasticsearch`)

适用于全文检索、日志分析、RAG 向量检索/混合检索项目（如 Python FastAPI + ES）。

### 2.1 `.env` 变量片段
```dotenv
# Elasticsearch 配置
ES_HTTP_PORT=9200
ES_TCP_PORT=9300
ES_VERSION=8.11.3
ES_MEM_LIMIT=1g
ES_PASSWORD={{PROJECT_NAME}}#Es2026!
```

### 2.2 `docker-compose.yml` 服务片段
```yaml
  {{PROJECT_NAME}}-es:
    image: docker.elastic.co/elasticsearch/elasticsearch:${ES_VERSION}
    container_name: ${CONTAINER_PREFIX}-es
    environment:
      <<: *shared-env
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms${ES_MEM_LIMIT} -Xmx${ES_MEM_LIMIT}"
      # 单机内部部署默认关闭 xpack 证书校验以简化连接；如需开启密码认证可设为 true 并配置 ELASTIC_PASSWORD
      xpack.security.enabled: "false"
      ELASTIC_PASSWORD: ${ES_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "${ES_HTTP_PORT}:9200"
      - "${ES_TCP_PORT}:9300"
    volumes:
      - ./es/data:/usr/share/elasticsearch/data
      - ./es/plugins:/usr/share/elasticsearch/plugins
      - ./es/logs:/usr/share/elasticsearch/logs
    networks:
      - {{PROJECT_NAME}}-network
    restart: unless-stopped
```
> **宿主机初始化提示**：Linux 下需执行 `mkdir -p es/{data,plugins,logs} && chmod -R 777 es` 以确保容器内 `elasticsearch` 用户（UID 1000）有写入权限。

---

## 3. Redis 缓存与消息队列 (`#Redis`)

适用于会话缓存、分布式锁、Celery 队列（FastAPI 异步任务）等场景。

### 3.1 `.env` 变量片段
```dotenv
# Redis 配置（默认使用非标端口增强安全性）
REDIS_PORT=8761
REDIS_PASSWORD={{PROJECT_NAME}}#Redis2026!
```

### 3.2 `docker-compose.yml` 服务片段
```yaml
  {{PROJECT_NAME}}-redis:
    container_name: ${CONTAINER_PREFIX}-redis
    image: redis
    environment:
      <<: *shared-env
    build:
      context: .
      dockerfile: redis-dockerfile
    ports:
      - "${REDIS_PORT}:{{REDIS_PORT}}"
    volumes:
      - ./conf/redis.conf:/home/{{PROJECT_NAME}}/redis/redis.conf
      - ./redis/data:/data
    command: redis-server /home/{{PROJECT_NAME}}/redis/redis.conf
    networks:
      - {{PROJECT_NAME}}-network
    restart: unless-stopped
```

---

## 4. 轻量级 rNacos 注册与配置中心 (`#rNacos`)

适用于微服务架构或需要动态配置中心但希望节省服务器内存的场景（Rust 实现，兼容 Nacos）。

### 4.1 `.env` 变量片段
```dotenv
# rNacos 配置
NACOS_PORT_1=8087
NACOS_PORT_2=8088
NACOS_PORT_3=8099
NACOS_PASSWORD=SII#{{PROJECT_NAME}}#2026!!
```

### 4.2 `docker-compose.yml` 服务片段
```yaml
  {{PROJECT_NAME}}-nacos:
    image: qingpan/rnacos:stable
    container_name: ${CONTAINER_PREFIX}-nacos
    platform: linux/amd64
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "${NACOS_PORT_1}:8848"
      - "${NACOS_PORT_2}:9848"
      - "${NACOS_PORT_3}:10848"
    volumes:
      - ./nacos/data:/io:rw
    environment:
      <<: *shared-env
      RNACOS_INIT_ADMIN_USERNAME: admin
      RNACOS_INIT_ADMIN_PASSWORD: ${NACOS_PASSWORD}
      RNACOS_HTTP_PORT: 8848
    restart: unless-stopped
    networks:
      - {{PROJECT_NAME}}-network
```

---

## 5. 关系型数据库 PostgreSQL / MySQL (`#Database`)

当数据库也需纳入同一套便携式 Docker Compose 编排时选用（若使用宿主机已有数据库，则通过 `host.docker.internal` 访问即可）。

### 5.1 PostgreSQL 服务片段
```yaml
  {{PROJECT_NAME}}-postgres:
    image: postgres:15-alpine
    container_name: ${CONTAINER_PREFIX}-postgres
    environment:
      <<: *shared-env
      POSTGRES_DB: ${DB_NAME:-{{PROJECT_NAME}}}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - ./db/data:/var/lib/postgresql/data
      - ./conf/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - {{PROJECT_NAME}}-network
    restart: unless-stopped
```

### 5.2 MySQL 8.0 服务片段
```yaml
  {{PROJECT_NAME}}-mysql:
    image: mysql:8.0
    container_name: ${CONTAINER_PREFIX}-mysql
    environment:
      <<: *shared-env
      MYSQL_DATABASE: ${DB_NAME:-{{PROJECT_NAME}}}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT:-3306}:3306"
    volumes:
      - ./db/data:/var/lib/mysql
      - ./conf/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    networks:
      - {{PROJECT_NAME}}-network
    restart: unless-stopped
```
