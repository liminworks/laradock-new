# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 通用指引

All AI-agent and contributor guidance for this repo lives in **[AGENTS.md](./AGENTS.md)**. Read it and follow it.
（`.cursorrules`、`.github/copilot-instructions.md` 同样只是它的指针。上游刻意保持精简，勿往本文件复制其内容。）

## 本仓库特有情况（liminworks fork，上游没有的内容）

这是 laradock/laradock 的个人 fork，用作本地开发环境（api/im 服务发现试点）。历史与上游同源，可直接 merge 上游更新。

### fork 定制（merge 上游时注意保留）

- `nacos/`：自建服务（v2.4.3 standalone + derby，已注册进主 compose 的 `include:`）。认证密钥走根 `.env` 的 `NACOS_*`（随机生成，不入 git；`nacos/defaults.env` 只放空默认）；8848/9848 仅绑定 127.0.0.1；`nacos/data/` 是 derby 运行数据（gitignored），删除会丢服务发现数据
- `prometheus/prometheus.yml`：在官方极简模板上追加了抓取 job（consumer-9502:9503、grafana:3000）
- 根 `.env` 的"个人定制覆盖"段：PHP_VERSION=8.2、MYSQL_VERSION=5.7、阿里云 composer 源、REDIS_PASSWORD=secret_redis（新版 redis 强制 requirepass，应用侧需同步）等

### 验证方式

- 改过 compose/env 后跑 `docker compose config`：应零 error 且零 "variable is not set" 告警（CI 的 compose-validate 就是 `docker compose config -q`）
- 日常起停/重建走 `./laradock` CLI（`./laradock help` 查看命令；`setup` / `start` / `rebuild` / `set KEY=VALUE` 等）

### 迁移遗留（2026-10 从旧 laradock 迁入，确认稳定后可清理）

- 旧环境备份：本地分支 `backup-old-master`、旧远端仓库 github.com/liminworks/laradock、旧完整 `.env` 备份 `.env.migration-backup`
- 本机数据目录 `~/.laradock/data` 目前只有 redis

### 约定

- 提交信息用中文
- "以新版配置为主"：优先采用上游新规范写法（新服务 = `<svc>/` 目录 + include 注册）；fork 特有需求尽量收进 `.env` 覆盖或独立目录，少改上游共享文件，减少 merge 冲突
