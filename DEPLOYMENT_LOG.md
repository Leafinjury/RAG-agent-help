# MewHelp 本次部署记录

> 记录时间均为 `Asia/Shanghai`。敏感值（尤其是模型 Key）始终不写入本文件。
> 执行依据：`DEPLOY.md`；项目根目录：`/home/leafinjury/projects/RAG_agent_help`。

## 状态摘要

- 当前状态：**已完成**。Python 依赖、数据库/Milvus 容器、种子数据、知识库向量、MCP Server 和 FastAPI 应用均已启动并通过验收。
- 已确认 `.env` 存在，`CHAT_API_KEY`、`EMBED_API_KEY`、`RERANK_API_KEY` 均非空（值隐藏）。
- 当前服务监听：MySQL `0.0.0.0:3306`、Milvus `0.0.0.0:19530`/healthz `:9091`、物流 MCP `127.0.0.1:8101`、售后 MCP `127.0.0.1:8102`、应用 `127.0.0.1:8000`。
- 本次没有自行安装系统级 Docker/容器运行时。工作期间环境外部出现了官方 `get-docker.sh` 副本（仓库根目录、未执行），以及已运行的 Docker Engine；来源不明，未对其做修改。

## 执行明细

| # | 时间 | DEPLOY 步骤 | 命令 / 工作目录 | 下载或生成内容 | 启动位置 / 端口 | 证据、结果与备注 |
|---:|---|---|---|---|---|---|
| 0 | 2026-09-18 10:46（约） | 前置检查 | 在项目根目录执行 `uv --version`、`docker compose version`、`make --version`，并检查内存/磁盘、进程和端口 | 未下载、未生成 | 未启动 | `uv 0.12.15`、GNU Make 4.4.1 可用；`docker` 命令不存在（不是仅 daemon 未启动）；内存约 7.6 GiB、可用磁盘约 952G；项目端口空闲。 |
| 1a | 2026-09-18 10:47 | 第 1 步（首次尝试） | `uv sync`（项目根目录） | 已从 PyPI 开始下载 `cryptography`、`pyarrow`、`faiss-cpu`、`pandas`、`numpy`、`grpcio`、`zstandard` 等；临时构建目录在 uv cache 下 | 未启动服务 | 失败：现有 CPython 3.14.4 没有 `asyncmy==0.2.11` 的适配 wheel，构建时缺少 `x86_64-linux-gnu-gcc`。未改代码、未安装系统编译器。 |
| 1b | 2026-09-18 10:48–10:49 | 第 1 步（重试成功） | `uv sync --python 3.13`（项目根目录） | uv 下载并缓存 CPython 3.13.15（约 33.2 MiB）；下载依赖 wheel（含 `asyncmy` 4.9 MiB、`pyarrow` 47.8 MiB、`faiss-cpu` 17.6 MiB、`pandas` 10.4 MiB、`numpy` 15.9 MiB、`grpcio` 6.6 MiB、`cryptography` 4.5 MiB、`uvloop` 4.2 MiB、`zstandard` 5.3 MiB 等） | 创建项目根目录 `.venv/`，使用 CPython 3.13.15；无常驻服务 | 成功：解析 155 个包，安装 103 个包。已核对 `fastapi 0.139.0`、`uvicorn 0.51.0`、`asyncmy 0.2.11`、`pymilvus 3.0.0`、`langchain 1.3.13`、`langgraph 1.2.9`、`mcp 1.28.1`、`openai 2.45.0`。 |
| 2 | 2026-09-18 10:46（检查） | 第 2 步（配置） | 检查项目根目录 `.env`（未打印内容） | 未下载、未生成；`.env` 已预先存在 | 未启动服务 | 三个 `*_API_KEY` 均非空；`CHAT_BASE_URL` 与 `CHAT_MODEL` 均已设置。Key 值未读取输出。文件权限当前为 `0644`，含真实 Key，建议用户在合适时收紧为 `0600`。 |
| 3 | 2026-09-18 10:46（前置失败） | 第 3 步（容器） | 计划执行 `docker compose up -d`（项目根目录） | 尚未拉取镜像 | 未启动 `mewhelp-mysql`、`mewhelp-milvus-etcd`、`mewhelp-milvus-minio`、`mewhelp-milvus` | 未执行：`docker` 命令缺失。预期镜像（待 Docker 可用后下载）：`mysql:8`、`quay.io/coreos/etcd:v3.5.16`、`minio/minio:RELEASE.2024-05-28T17-19-04Z`、`milvusdb/milvus:v2.6.22`。预期端口：3306、19530、9091（etcd/minio 主要为容器内部）。 |
| 3b | 2026-09-18 10:51 | 第 3 步（权限复查） | `docker --version`、`docker compose version`、`docker info`、`docker compose ps`（项目根目录） | 未拉取镜像 | 系统 `dockerd` PID 162209 已运行；socket `/run/docker.sock` | Docker 29.8.1、Compose v5.5.1 可执行；但 socket 为 `root:docker`、权限 `0660`，当前 `leafinjury` 会话不在 `docker` 组，`docker info/compose ps` 均 `permission denied`。未修改 socket 权限、未停/启现有系统服务。 |
| 3c | 2026-09-18 11:01–11:06 | 第 3 步（镜像下载回退） | 先执行 `docker compose up -d`（直接源失败），再按文档从 DaoCloud mirror 拉取并 `docker tag` 回 Compose 原名（项目根目录） | 成功下载：MinIO 220MB（digest `sha256:391d…af08`）、MySQL 1.12GB（`sha256:85b9…38f2a`）、etcd 85.4MB（`sha256:d967…e159`）、Milvus 1.3GB（`sha256:a8ac…c1df`）；镜像站副本标签也保留 | 尚未启动容器 | 直接拉取 MinIO 返回 `pull access denied`；`https://docker.m.daocloud.io/v2/` 返回 401（文档定义为正常鉴权挑战）。镜像站拉取全部成功并打回原名，未改 Compose 文件。 |
| 3d | 2026-09-18 11:06:44 | 第 3 步（容器启动） | `docker compose up -d`（项目根目录） | 创建 Compose 网络 `rag_agent_help_default` 和四个 named volumes：`rag_agent_help_mewhelp-mysql-data`、`rag_agent_help_mewhelp-milvus-etcd`、`rag_agent_help_mewhelp-milvus-minio`、`rag_agent_help_mewhelp-milvus-data` | 启动 `mewhelp-mysql`（3306）、`mewhelp-milvus-etcd`（容器 2379/2380）、`mewhelp-milvus-minio`（容器 9000）、`mewhelp-milvus`（19530/9091） | 四容器健康检查均为 `healthy`。容器 ID：mysql `658c4ce11391…ac89`、etcd `32b9a8bf8a82…57cf`、MinIO `51e59362dbfc…139c`、Milvus `91759b4b34bf…350`。 |
| 4 | 2026-09-18 11:08（约） | 第 4 步（种子） | `make seed`、`make seed-conv`（项目根目录；SQL 通过 `docker exec` 注入 `mewhelp-mysql`） | FAQ/会话种子写入 MySQL | `mewhelp-mysql:3306` | `make seed`：before/after 均 6；`make seed-conv`：会话 2、消息 4。SQL 复核：`faq=6`、seed conversations=2、seed messages=4。 |
| 5a | 2026-09-18 11:09（约） | 第 5 步（切块入库） | `make kb-build`（项目根目录，`PYTHONPATH=.`） | 读取 6 份知识文档，写入 MySQL `knowledge_chunks`，共 52 个 pending 块 | `mewhelp-mysql:3306` | 输出：`建库(pending):共 52 块`。 |
| 5b | 2026-09-18 11:09（约） | 第 5 步（向量化） | `make kb-vectorize`（项目根目录，`PYTHONPATH=.`） | 调用已配置嵌入上游；在 Milvus 创建/写入 `knowledge` collection | `mewhelp-milvus:19530` | 输出：`本次向量化 52 块;Milvus 现有 52 条`，验收通过。 |
| 6 | 2026-09-18 11:13 | 第 6 步（启动） | `make dev` → `scripts/dev.sh`（项目根目录） | 复用 healthy 依赖容器；生成 `data/mcp-*.pid`、`log/mcp-*.log` 和 `log/app.log` | 物流 MCP launcher PID `183130`（实际 server PID `183140`）监听 `127.0.0.1:8101`；售后 launcher PID `183132`（实际 server PID `183139`）监听 `127.0.0.1:8102`；Uvicorn PID `183161` 监听 `127.0.0.1:8000` | `Milvus 预热完成`、MCP startup complete、Uvicorn application startup complete 均出现；首页 `curl http://localhost:8000/` 返回 HTTP 200。`make dev` 当前以前台会话保持运行。 |
| 7 | 2026-09-18 11:14（约） | 第 7 步（最终验收） | `/tmp/mewhelp-q.json` + `curl --max-time 90 http://localhost:8000/api/agent` | 生成一次验收请求/响应文件（不含 Key） | 应用 `127.0.0.1:8000` | HTTP 200，用时约 4.902 秒；`tool_calls` 包含 `query_order(order_id=1001)` 和 `query_logistics(tracking_no=SF213502378238)`；答案返回“已揽件”、当前位置“成都”，目标验收通过。 |

## 额外只读验证

- `bash -n .env` 通过；没有输出或记录任何 Key。
- 在 `.venv`（CPython 3.13.15）中成功导入 `fastapi`、`uvicorn`、`asyncmy`、`pymilvus`、`langchain`、`langgraph`、`mcp`、`openai`、`app.config`。
- `docker compose config --quiet` 通过（仅解析本地 Compose 文件，不访问 daemon）；确认服务为 `mysql`、`etcd`、`minio`、`milvus-standalone`。
- 当前仓库新增的 `get-docker.sh` 是 2026-09-18 10:49 生成的官方 Docker 安装脚本副本；本次未执行、未修改、未删除。
- 最终 `docker compose ps` 显示四个容器均 `Up (healthy)`；`curl http://localhost:9091/healthz` 返回 `OK`。

## 当前服务清单与停止命令

- Docker：四个 Compose 容器均 healthy；宿主端口为 3306、19530、9091。
- MCP：物流 `127.0.0.1:8101`、售后 `127.0.0.1:8102`；PID 文件在 `data/mcp-logistics.pid`、`data/mcp-aftersales.pid`，日志在 `log/mcp-logistics.log`、`log/mcp-aftersales.log`。
- 应用：`127.0.0.1:8000`，当前 `make dev` 前台会话中的 Uvicorn PID 为 `183161`；应用日志在 `log/app.log`。
- 停止应用和 MCP：`make dev-down`（或前台会话按 `Ctrl+C`）；按需停止容器：`docker compose down`。不要在需要保留数据时加 `-v`。
