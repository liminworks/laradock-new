# 笔记：本仓库的 CI/CD 与生产部署

> liminworks fork 专属笔记，整理于 2026-10-25。涵盖：仓库 CI/CD 机制、CI/CD 概念辨析、
> laradock 生产部署路径（`./laradock ship`）、云平台免费档、各可选服务的线上可用性评估。
> 时效性内容（如云平台免费额度）以标注日期为准，使用前自行复核。

---

## 1. 本仓库的 CI/CD：GitHub Actions

代码托管在 GitHub（上游 `laradock/laradock`，fork 在 `liminworks/laradock-new`），
所以 CI 用的是 **GitHub Actions**，与 GitLab 无关（仓库里没有 `.gitlab-ci.yml`）。
全部配置在 `.github/workflows/`，共 5 个：

| 文件 | 触发条件 | 干什么 |
|---|---|---|
| `main-ci.yml` | 任何分支/tag 的 push、所有 PR、每周日定时 | 核心校验 + 镜像构建（见下） |
| `hadolint.yml` | PR / push master 且改了 `**/Dockerfile` | hadolint lint 全部 Dockerfile，仅 error 级失败 |
| `shellcheck.yml` | PR / push master 且改了 `**/*.sh` | ShellCheck 检查脚本，仅 error 级失败 |
| `build-deploy-docs.yml` | push master 且改了 `DOCUMENTATION/**` | 构建文档站发布 GitHub Pages（唯一的"CD"） |
| `stale.yml` | 每天 | 上游 issue/PR 保活机器人 |

`main-ci.yml` 的 3 个 job：

1. **compose-validate**：`cp .env.example .env && docker compose config -q`——CLAUDE.md
   里"改 compose/env 后要验证"这条规则的 CI 化身。
2. **build-php**：4 个 PHP 版本 × php-fpm/php-worker/workspace = 16 个构建。把 `.env`
   全部 `=false` 改 `=true`（再关掉十几个不稳定扩展）后 `docker buildx bake
   --set *.output=type=cacheonly`，验证镜像能编过，不推送。
3. **build-other**：nginx/redis/mysql/mariadb/percona/minio/mongo 7 个构建。

### 对本 fork 的实际影响

- **会跑**：CI、hadolint、shellcheck——每次 push 都触发；其中 23 个 Docker 构建任务
  纯烧 Actions 时长（不推镜像、无产物）。docs 发布和 stale bot 有
  `if: github.repository == 'laradock/laradock'` 守卫，fork 上永不运行。
- 想省掉重型构建：提交信息带 `[skip ci]`（GitHub 原生支持），或给 build-php/build-other
  也加 repository 守卫。
- fork 的 Actions 默认可能关闭，到仓库 Actions 标签页确认实际是否在跑。

### 本地等价检查（不依赖 GitHub）

```bash
# 等价 compose-validate
cp .env.example .env && docker compose config -q

# 等价 hadolint（改了某个 Dockerfile 时）
docker run --rm -i hadolint/hadolint:v2.14.0 hadolint --failure-threshold error \
  --ignore DL3008 --ignore DL3018 --ignore DL3013 --ignore DL3016 - < php-fpm/Dockerfile

# 等价 shellcheck（改了某个 .sh 时）
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable laradock
```

---

## 2. 概念辨析

### CI 跟着代码托管平台走

| 代码托管 | CI 系统 | 需要自装 runner 吗 |
|---|---|---|
| GitHub | GitHub Actions（`.github/workflows/`） | 不用，云端自带（`runs-on: ubuntu-latest`） |
| GitLab | GitLab CI（`.gitlab-ci.yml`） | **要**，服务器只管调度，干活的是 runner |

### laradock 里的 `gitlab/`、`gitlab-runner/` 是什么

laradock 是"开发环境百货商店"（100+ 可选服务），这两个只是**给"你自己的项目"准备的
可选容器**，与本仓库自己的 CI 毫无关系：

- `gitlab/` = 自托管 GitLab 服务器（Web 8989 / SSH 2289，复用主环境 redis + postgres）
- `gitlab-runner/` = GitLab CI 的执行器，启动时用根 `.env` 的
  `GITLAB_CI_SERVER_URL` + `GITLAB_RUNNER_REGISTRATION_TOKEN` 自动注册

GitLab CI 架构：`GitLab 服务器（大脑，解析 .gitlab-ci.yml 并调度）→ 派发 → GitLab Runner（工人）`。
自建 GitLab 必须配 runner 才能跑 CI。不用它们就忽略，按名启动才会计入 compose。

### GitLab Runner vs Jenkins：不在一个层面

```
Jenkins（一体机）：  [ 大脑：流水线调度/插件 ] + [ 工人：内置 executor/agent ]  ← 完整独立系统
GitLab CI（拆开）：  [ 大脑：GitLab 服务器 ] ──派发 job──> [ 工人：GitLab Runner ]
```

| | GitLab CI | Jenkins |
|---|---|---|
| 前提 | 代码托管在 GitLab | 无前提，哪个平台都行 |
| 配置 | `.gitlab-ci.yml`（YAML，声明式） | `Jenkinsfile`（Groovy，自由但复杂） |
| 维护 | Runner 几乎免维护 | 要自己养：升级/安全/插件地狱 |
| 现状 | 自建 GitLab 的事实标准 | 主要是存量系统，新项目少有从零选它 |

选择口诀：代码在 GitLab → GitLab CI + Runner；代码在 GitHub → GitHub Actions；
Jenkins 适合流程极复杂或已有历史包袱的场景。

---

## 3. 生产部署：`./laradock ship` 路径

### 原理

laradock 能上线的根本原因：它只是"Dockerfile + compose 配方"，而 **Docker 镜像本身就是
可移植的部署单元**。`production/README.md` 一句话："镜像就是通用部署适配器"。

```
本地 Mac                          镜像仓库                    线上服务器
┌──────────────┐    --push    ┌───────────┐    pull    ┌─────────────────────┐
│ ./laradock   │ ──────────> │ ACR/Docker │ ────────> │ 一个 web 容器         │──> 托管数据库/Redis
│ ship（打包）  │              │ Hub/GHCR  │            │ (nginx+php-fpm+代码) │    (RDS/ElastiCache，
└──────────────┘              └───────────┘            └─────────────────────┘     绝不用容器跑)
```

dev 与 prod 的关键差异：

| | 开发模式 | 生产模式 |
|---|---|---|
| 代码 | 目录 mount 进容器 | 烧进镜像（`composer --no-dev`） |
| 调试 | Xdebug | 关 Xdebug，opcache `validate_timestamps=0` |
| 数据库 | 容器里的 mysql | 云托管服务，绝不跑容器 |
| 拓扑 | 各服务独立容器 | nginx + php-fpm 合一个自包含镜像，监听 `$PORT` |

### 操作步骤

```bash
# 1. 构建（本质：docker build -f production/Dockerfile <应用目录>，见 laradock CLI cmd_ship）
./laradock ship 仓库地址/app:1.0
docker run -p 8080:8080 仓库地址/app:1.0     # 本地验证 → http://localhost:8080

# 2. 推送
docker login <registry> && docker push 仓库地址/app:1.0

# 3. 线上跑（四选一，同一镜像）
#   A 单服务器（最常见）：
cp production/.env.example production/.env    # 填真实值（不入 git、不进镜像）
docker compose -f production/compose.yml up -d
#   B 云 PaaS：production/providers/ 有 9 家现成参考配置（Cloud Run/ECS/App Runner/Fly/Render…）
#   C Kubernetes：kubectl apply -f production/kubernetes.yaml
#   D 多台自有服务器：Kamal（providers/kamal-deploy.yml）
```

镜像内容（`production/Dockerfile`）：nginx + php-fpm 合一、supervisor 托管、生产 php.ini
（关错误显示）、opcache 全开、健康检查、entrypoint 支持命令覆盖（worker/migrate 复用同一镜像）。

### 应用层配套（按需）

- **队列/定时**：`docker compose -f production/compose.yml --profile worker --profile scheduler up -d`
- **迁移**：每次发布跑一次 `docker run --rm 镜像 php artisan migrate --force`
- **持久文件**（storage/、上传）：必须上对象存储（OSS/S3）或挂卷，不能留在镜像里（发版即换镜像）
- **HTTPS**：容器只暴露 HTTP，TLS 交给前面的负载均衡/CDN
- **密钥**：走运行时 env 或密钥管理，绝不烧进镜像（`production/dockerignore.sample` 防泄漏）

### 本 fork 的四个坑（重要）

1. **多项目布局 vs ship 的单应用假设**：`.env` 的 `APP_CODE_PATH_HOST=../` 指向
   `PhpstormProjects/` 整个目录（dev 挂载多项目没问题），但 ship 会把它整个 COPY 进镜像。
   多项目场景绕开 CLI，直接手动指定项目目录：
   ```bash
   docker build --platform linux/amd64 -f production/Dockerfile \
     --build-arg PHP_VERSION=8.2 \
     -t 仓库地址/app:1.0 /path/to/具体项目
   ```
   `.dockerignore` 要放在该项目目录根下。
2. **PHP 版本不一致**：本地 `.env` 是 `PHP_VERSION=8.2`，但 `production/Dockerfile` 的
   `ARG PHP_VERSION=8.3` 是默认值，CLI 不读你的 `.env` → 必须显式 `--build-arg PHP_VERSION=8.2`。
3. **扩展差异**：生产底座是 Docker Hub 公共镜像 `laradock/php-fpm:latest-<版本>`（上游默认
   扩展集），本地 `.env` 里开的扩展它不一定有，缺的在 Dockerfile 里补
   `RUN docker-php-ext-install ... && pecl install ...`。
4. **Python 服务不适用**：该镜像是 PHP-only（php-fpm 底座）。FastAPI 等服务需自己另写
   Dockerfile，不走 ship。

### 同区域原则（sld_internal 的教训）

`production/README.md` 第一原则"数据库用托管服务"之外还有一条隐含规则：**应用与
DB/Redis 必须同区域/同 VPC**。sld_internal 本地实测过跨境 RTT 每请求 12~15 次、直接拖垮
性能，线上跨区部署会放大得更厉害。

---

## 4. 云平台免费档（2026-10 时点）

"免费档"= 云平台白送的部署额度，用来在花钱前把"推镜像 → 部署 → 访问"流程走通。

| 平台 | 免费额度 | 限制/坑 |
|---|---|---|
| Google Cloud Run | 每月 200 万请求长期免费 + CPU/内存时长；新用户送 $300/90 天 | 默认缩容到 0，有冷启动 |
| Render | 免费跑 1 个 web 服务 | 15 分钟无流量休眠（冷启动约 1 分钟）、512MB、带宽 5GB、免费库 30 天过期 |
| Railway | 一次性 $5 试用金 | 用完即止 |
| Heroku | 无免费档（最低约 $5/月） | — |
| Oracle Cloud | 永久免费小服务器（ARM 4核/24G 档有名） | 开户审核严，要绑卡 |

- 衔接方式：选 PaaS 用 `production/providers/` 对应文件；选免费 VM 则方式 A 照用。
- **练手建议**：Cloud Run 额度最实在；Render 适合演示不适合验证。
- **国内注意**：海外免费档从国内访问慢且不稳，练手可以，正式环境选国内云
  （阿里云/腾讯云新用户试用 ECS、轻量服务器），那台机器上走方式 A。
- 免费档别放真数据：内存小、会休眠、免费数据库会过期。
- 参考：[Cloud Run pricing](https://cloud.google.com/run/pricing) ·
  [Render 免费档文档](https://render.com/docs/free)

---

## 5. 可选服务的线上可用性评估

### gitlab / gitlab-runner：本地玩具档，别直接上线

| 硬伤 | 事实 | 后果 |
|---|---|---|
| 默认 root 密码在 git 里 | `gitlab/defaults.env`：`GITLAB_ROOT_PASSWORD=laradock` | 公开可查 |
| 端口裸绑公网 + 明文 HTTP | `ports: "8989:80"` 绑 0.0.0.0，`external_url` 为 http | 谁都能访问 |
| runner = 宿主机 root | `gitlab-runner/compose.yml` 挂 `/var/run/docker.sock`，shell executor | runner 被攻破 ≈ 丢整台服务器 |
| CI 与业务混部署 | 同一台机器 | CI 持有全部代码密钥，高价值攻击目标，必须隔离 |
| 本身很重 | 官方建议 4G+ 内存 + 持续运维 | 塞业务服务器既慢又添乱 |

线上正道：gitlab.com SaaS，或**单独一台服务器**按官方文档部署，与业务物理隔离。

### prometheus / grafana：默认裸奔，线上前必须加固

事实：

- **Grafana**：compose 未设 `GF_SECURITY_ADMIN_PASSWORD` → 走默认 **admin/admin**；
  `ports: "${GRAFANA_PORT}:3000"` 绑 0.0.0.0，公网可达。
- **Prometheus**：**设计上就没有认证功能**（端口通 = 指标和查询 API 全开），且挂在
  frontend 网络。不是配置遗漏，是它的哲学：默认放内网、靠网络层保护。

加固三板斧（fork 已对 nacos 做过同样的事——随机密钥参数化 + 仅绑 127.0.0.1，照搬即可）：

1. 端口只绑本机：`"127.0.0.1:3000:3000"`、`"127.0.0.1:9091:9090"`；远程看走 SSH 隧道
   `ssh -L 3000:127.0.0.1:3000 服务器`。
2. Grafana 补 `GF_SECURITY_ADMIN_PASSWORD`，值放根 `.env`（随机生成、不入 git），照
   `NACOS_*` 模式。
3. Prometheus 无密码可设 → 靠网络隔离；必须远程时套反向代理加 basic auth。

本地 Mac 也建议改：连公共 Wi-Fi 时 0.0.0.0 绑定 = 局域网内 admin/admin 人人可登。
生产监控应单独规划（云托管可观测套件或独立监控机），不与业务混部署。

### certbot：线上工具，开发环境用不上

它向 Let's Encrypt 申请**公共受信任**证书，前提是公网注册域名 + 公网 80 端口可达
（HTTP-01 校验）——本地 `localhost`/内网根本不满足，所以与开发环境无关。

开发环境确实需要 https 的少数场景（OAuth 回调测试、Service Worker/PWA、Secure cookie、
混合内容调试）正解是**自签证书**，推荐 mkcert：

```bash
brew install mkcert
mkcert -install          # 本地 CA 装进系统钥匙串
mkcert localhost myapp.test
```

线上分两种形态：

| 形态 | TLS 在哪 | certbot |
|---|---|---|
| 裸服务器，nginx 直接对公网 | nginx 容器里 | ✅ 有用（记得配自动续期） |
| production/ 镜像 + 云 LB/CDN | LB/CDN 层（阿里云 SLB/CDN 可挂免费证书） | ❌ 不需要 |

certbot 在本仓库的适用面很窄：只有"单台裸服务器 + laradock nginx 直接对外"这一种
老式部署才需要。

---

## 6. 相关文件索引

| 内容 | 位置 |
|---|---|
| CI workflow 定义 | `.github/workflows/`（5 个文件） |
| 生产镜像 Dockerfile | `production/Dockerfile` |
| 单服务器生产 compose | `production/compose.yml`（+ `production/.env.example`） |
| 各云平台部署参考 | `production/providers/`（9 家） |
| K8s / Kamal | `production/kubernetes.yaml`、`production/providers/kamal-deploy.yml` |
| 冒烟测试 | `production/smoke-test.sh`、`production/smoke-test-k8s.sh` |
| ship 命令实现 | `laradock` CLI 的 `cmd_ship`（约 1213 行起） |
| 生产路径总说明 | `production/README.md` |
| 加固先例（nacos） | git 历史 `e2aeb809`、`nacos/` 目录 |
