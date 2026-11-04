# CVAT 二次开发运行指南（Windows 实测环境版）

> 配套文档：《CVAT代码架构总结.md》
> 适用环境：Windows 10/11 本地机，仓库位于 `D:\files\github\cvat`（develop 分支）
> 检测到的本机现状：Python 3.13.16、Node v22.23.2、npm 10.9.8、corepack 0.34.6、**Docker 未装**、**WSL 未启用**、**git 不在 PATH**

---

## 0. 三条路线怎么选

| 路线 | 做什么 | 适合 | 改代码体验 |
|---|---|---|---|
| **A. 全 Docker 快速跑通** | 一条命令起全部容器 | 第一次看效果、验收、跑通环境 | ❌ 改代码要重建镜像 |
| **B. 前端热更 + Docker 后端（推荐起步）** | 后端在容器、前端在 Windows 本地热更 | 主要改前端/画布/标注交互 | ✅ 前端即时生效；后端改完 rebuild |
| **C. 官方完整开发模式（WSL2）** | Docker 依赖 + WSL2 内本地 Django + 本地前端 | 深改后端、断点调试、写测试 | ✅✅ 前后端都能断点热更 |

> 为什么没有"纯 Windows 本地后端"方案：CVAT 后端依赖 Postgres、Redis、Kvrocks、OPA、ClickHouse 以及 xmlsec/libldap/PyAV 等 C 扩展，官方明确不支持 Windows 原生跑后端，开发必须在 Docker（或 WSL2）内。

---

## 1. 前置安装（三条路线都要）

### 1.1 Docker Desktop（必装）

1. 下载安装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)
2. 安装时勾选 **Use WSL 2 instead of Hyper-V**（安装向导默认勾选）
3. 确认 Windows 虚拟化已开启（任务管理器 → 性能 → CPU → 虚拟化：已启用；未开启需进 BIOS 打开）
4. 装完重启，打开 Docker Desktop 确认 `docker --version` 和 `docker compose version` 可用

### 1.2 git（建议装）

当前 PATH 里没有 git（clone 已由你完成，不影响运行），但后续 `git pull` 更新 develop 需要。装 [Git for Windows](https://git-scm.com/download/win) 即可。

### 1.3 前端工具链（已有，核对版本）

- Node **v22** 可用（官方文档写 Node 20，22 为 LTS，一般兼容；若 webpack 构建报 Node 版本相关错误，用 nvm-windows 切到 20）
- yarn 未装，用 corepack 启用（仓库根 `package.json` 指定 `packageManager: yarn@4.9.2`）：

```powershell
corepack enable yarn
yarn --version   # 应显示 4.x
```

### 1.4 WSL2 + Ubuntu（仅路线 C 需要，B 可跳过）

```powershell
wsl --install -d Ubuntu-24.04   # 需重启一次
wsl --set-default-version 2
wsl -d Ubuntu-24.04            # 进入 Ubuntu
```

---

## 2. 路线 A：全 Docker 快速跑通（20 分钟）

```powershell
cd D:\files\github\cvat
docker compose up -d --build
```

首次构建约 10–20 分钟（拉基础镜像 + 编译 Python 依赖）。完成后：

```powershell
# 创建管理员账号（交互式输入用户名/邮箱/密码）
docker compose exec cvat_server python manage.py createsuperuser
```

浏览器打开 **http://localhost:8080** 登录。

停止/清理：

```powershell
docker compose down          # 停止
docker compose down -v       # 停止并删除数据卷（任务/标注全删，慎用）
```

> 此模式下改代码流程：`docker compose build cvat_server`（或 `cvat_ui`）→ `docker compose up -d`。迭代慢，不推荐日常开发用。

---

## 3. 路线 B：前端热更 + Docker 后端（推荐起步）

后端全部容器化，前端在 **Windows 原生** 跑 webpack dev server（3000 端口，热更新）。

### 3.1 起后端（全部服务）

```powershell
cd D:\files\github\cvat
docker compose up -d --build
```

### 3.2 初始化（首次）

```powershell
docker compose exec cvat_server python manage.py createsuperuser
```

（容器 `backend_entrypoint.sh` 的 `init` 命令已自动执行 migrate / migrateredis / syncperiodicjobs，无需手动）

### 3.3 前端依赖 + 启动 dev server

```powershell
# 仓库根目录
corepack enable yarn
yarn --immutable              # 安装全部 workspace 依赖（cvat-ui/cvat-core/cvat-canvas/...）

# 启动前端 dev server，代理到 Docker 后端 8080
cd cvat-ui
yarn webpack serve --env API_URL=http://localhost:8080 --mode=development
```

> 关键点：`cvat-ui/package.json` 的 `start` 脚本默认代理到 `http://localhost:7000`（本地 Django 的端口，路线 C 才用）。**路线 B 必须手动传 `--env API_URL=http://localhost:8080`**，让 dev server 把 `/api/`、`/static/` 等请求转发给容器里的后端。

打开 **http://localhost:3000**。

- 改 `cvat-ui/src`、`cvat-core/src`、`cvat-canvas/src` 里的代码 → 浏览器即时刷新
- 改后端代码 → 需要 `docker compose build cvat_server && docker compose up -d cvat_server`（可只重建 server 容器）
- 开发代理规则（webpack.config.js）：`/api/`、`/static/`、`/admin`、`/django-rq`、`/documentation` 等走代理，其余前端本地服务

### 3.4 验证

1. 登录后创建 Project → Task → 上传图片/视频
2. 观察 `cvat_worker_chunks` 容器日志确认 chunk 生成：
   ```powershell
   docker compose logs -f cvat_worker_chunks
   ```
3. 打开任务进入标注页，画一个框，确认能保存

---

## 4. 路线 C：官方完整开发模式（WSL2，深改后端用）

目标形态：

```
Windows 原生:  yarn webpack dev server :3000 ──代理──▶ 本地 Django :7000
WSL2 Ubuntu:   python manage.py runserver 7000 + RQ worker + scheduler（VS Code 可断点）
Docker:        Postgres / Redis / Kvrocks / OPA /（提供 OPA rules 的 cvat_server 副本）
```

### 4.1 准备

```powershell
# 1) WSL2 + Ubuntu（见 1.4）
# 2) 仓库放进 WSL（二选一）
#    推荐：在 WSL 里重新 clone（/mnt/d 跨文件系统 IO 慢，编译依赖时明显）
wsl -d Ubuntu-24.04
cd ~ && git clone https://github.com/cvat-ai/cvat.git
#    或直接访问现有仓库（IO 慢，但省空间）
cd /mnt/d/files/github/cvat
```

### 4.2 WSL 内安装系统依赖

```bash
sudo apt-get update && sudo apt-get --no-install-recommends install -y \
  build-essential curl git python3-dev python3-pip python3-venv python3-tk \
  libldap2-dev libsasl2-dev libgeos-dev cargo redis-server
```

Python 必须是 **3.12**（CVAT 官方只支持 3.12，Ubuntu 24.04 自带 3.12）：

```bash
python3 --version   # 应为 3.12.x
```

### 4.3 Docker 起依赖服务（在 WSL 内或 Windows 终端均可，Docker Desktop 已共享）

```bash
cd <仓库路径>
mkdir -p logs keys
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build \
  cvat_opa cvat_db cvat_redis_inmem cvat_redis_ondisk cvat_server
```

> 这个 `cvat_server` 容器是"给 OPA 提供规则"的副本（跑在 8080）；**本地 Django（7000）才是你的主服务**。改动了 `iam/rules/*.rego` 后需重跑这条命令让规则生效。

### 4.4 WSL 内装 Python 依赖并初始化

```bash
python3 -m venv .env
source .env/bin/activate
pip install -U pip wheel setuptools
pip install --no-binary lxml,xmlsec -r cvat/requirements/development.txt -r dev/requirements.txt

python manage.py migrate
python manage.py migrateredis
python manage.py collectstatic
python manage.py syncperiodicjobs
python manage.py createsuperuser
```

### 4.5 启动后端（VS Code 断点调试）

```bash
source .env/bin/activate && code .   # VS Code Remote-WSL 打开仓库根目录
```

在 VS Code 中：
- 打开 `.vscode/launch.json`（仓库自带完整调试配置）
- 选择 **`server: debug`**（compound，同时启动 Django 7000 + RQ worker + RQ scheduler），F5
- 或分别运行 `server: django` / `server: RQ - worker` / `server: RQ - scheduler`

### 4.6 启动前端

```powershell
# Windows 原生终端（仓库根目录）
corepack enable yarn
yarn --immutable
yarn run start:cvat-ui      # 代理到 http://localhost:7000（默认 API_URL），打开 http://localhost:3000
```

前后端都起来后：浏览器访问 **http://localhost:3000**，可在 VS Code 里给 Django/Python 代码打断点，Chrome 里给 TS/TSX 打断点（`ui.js: debug` 配置）。

---

## 5. 常见问题排查

| 症状 | 原因/处理 |
|---|---|
| `docker compose up` 卡在 build 很久 | 首次编译 Python 依赖（datumaro 源码安装等），正常；换国内镜像源可加速拉取 |
| 前端 3000 打开但接口 502/404 | 检查 API_URL 代理目标：路线 B 应为 8080，路线 C 应为 7000 |
| 创建任务后一直"初始化中" | 看 `docker compose logs -f cvat_worker_chunks`；chunk 队列没起来或媒体文件读取失败 |
| 视频任务画面黑/不出帧 | 浏览器需支持 H.264 解码（Chrome/Edge OK）；检查 dev server 的 COOP/COEP header（webpack.config.js 已配） |
| `yarn --immutable` 报错 | 先 `corepack enable yarn` 并确认版本 4.x；Node 版本异常时切 20 |
| Python 3.13 装 CVAT 依赖报错 | CVAT 只支持 3.12，Windows 原生后端不折腾，直接走路线 A/B 或 C |
| 页面提示规则/权限异常 | `iam/rules/` 的 Rego 规则由 OPA 执行；规则改动后重建 cvat_server 容器 |
| 内存不足（前端构建 OOM） | 构建前设 `$env:NODE_OPTIONS="--max-old-space-size=8192"` |
| 想清空重来 | `docker compose down -v`（会删数据库与媒体数据） |

## 6. 与二次开发对应的"我改什么、重启什么"

| 改动 | 生效方式 |
|---|---|
| `cvat-ui/src/**`（React 组件/容器） | 路线 B/C 自动热更新 |
| `cvat-core/src/**`、`cvat-canvas/src/**` | 路线 B/C 自动热更新（webpack 重新编译） |
| 后端 `engine/views.py`、`models.py`、`task.py` 等 | 路线 C：Django dev server 自动重载（`--noreload` 除外）；路线 A/B：重建 cvat_server 容器 |
| 后端模型变更（models.py + 迁移） | `python manage.py makemigrations` + `migrate`（路线 C）；容器内同理 |
| `iam/rules/*.rego` 权限规则 | 重建 cvat_server 容器（OPA 从 /api/auth/rules 重新拉 bundle，轮询 5–15s） |
| 新增 RQ 队列/worker | settings.RQ_QUEUES + docker-compose 加 worker 服务 + 重建 |
| 新增数据集格式（dataset_manager/formats） | 重启后端进程；前端格式列表接口自动更新 |
