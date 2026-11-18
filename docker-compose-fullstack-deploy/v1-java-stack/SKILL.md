---
name: docker-compose-fullstack-deploy
description: >-
  生成标准化的前后端分离（Nginx + Java/SpringBoot + Redis + 轻量级 rNacos + AI 流式网关）Docker Compose
  单机/多环境便携式部署方案。当用户需要为项目搭建或生成 Docker Compose 部署目录、配置文件、前后端热更新与备份脚本时使用此技能。
---

# 全栈便携式 Docker Compose 部署方案生成指南 (v1 - Java/SpringBoot + Vue + Redis + rNacos 专版)

本技能用于为前后端分离项目（含微服务注册中心与 AI 流式网关场景）快速生成一套**高内聚、多环境隔离、配置与数据全外挂、支持前后端秒级热更新与回滚**的 Docker Compose 部署体系。

## 核心架构规范
1. **`.env` 驱动的多环境隔离**：使用 `CONTAINER_PREFIX` 统一控制所有容器名称及网络名称，端口与密码集中收拢在 `.env` 文件中。
2. **便携式高内聚目录结构**：配置文件集中于 `./conf/`，持久化数据（`data`）、日志（`logs`）、上传文件（`uploadPath`）均以相对路径挂载在项目根目录下。
3. **容器与宿主机网络打通**：统一配置 `extra_hosts: - "host.docker.internal:host-gateway"`。
4. **轻量组件与安全基线**：选用 `qingpan/rnacos:stable` 替代传统 Java Nacos；Redis 使用非标端口并禁用高危 `CONFIG` 命令。
5. **实战化 Nginx 网关**：HTTP + 自签名 HTTPS 双监听，`/llm-api/` 开启 `proxy_buffering off;` 支持 SSE 流式输出，`/prod-api/` 支持 WebSocket。
6. **热发版与时间戳备份回滚**：`html/deploy.sh` 自动将旧版 `dist` 按时间戳移入 `backup/` 并解压新 `dist.tgz` 后重启 Nginx。
