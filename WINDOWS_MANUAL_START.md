# Windows 手工启动 RAG Center

本文针对这台 Windows 电脑上的项目目录：

```text
C:\Users\Star\Desktop\project\rag-center
```

所有命令均在 **PowerShell** 中执行。当前电脑已经安装依赖、建好数据库，并在项目根目录和 `frontend` 目录分别配置了 `.env`。以后日常启动直接按第一节操作；只有换电脑、重建虚拟环境或更新依赖时，才需要看“首次安装或环境损坏后重建”。

## 每次启动：按顺序打开 4 个 PowerShell 窗口

### 1. 启动 Docker Desktop 和三个基础服务（窗口 A）

从 Windows 开始菜单打开 **Docker Desktop**，等待界面显示 Docker Engine 已运行。然后在 PowerShell 中执行：

```powershell
Set-Location 'C:\Users\Star\Desktop\project\rag-center'
docker info --format '{{.ServerVersion}}'
docker compose up -d postgres redis elasticsearch
docker compose ps
```

`docker compose ps` 中，`postgres`、`redis`、`elasticsearch` 应最终显示 `healthy`。Elasticsearch 通常比另外两个服务启动慢；若显示 `health: starting`，稍等后再运行一次 `docker compose ps`。

这三个容器分别提供：PostgreSQL + pgvector（5432）、Redis（6379）和 Elasticsearch（9200）。此处只启动项目实际需要的三个服务，不必启动 Compose 文件中的可选 Langfuse 服务。

### 2. 启动 FastAPI 后端（窗口 B，保持窗口打开）

新开一个 PowerShell 窗口，执行：

```powershell
Set-Location 'C:\Users\Star\Desktop\project\rag-center'
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8001
```

看到 `Application startup complete` 和 `Uvicorn running on http://127.0.0.1:8001` 即表示后端已启动。`--reload` 适合边看代码边修改；修改 `.env` 后仍建议在此窗口按 `Ctrl+C`，再重新执行启动命令。

本机使用 **8001**，因为 8000 端口曾被其他程序占用。不要只把后端改回 8000 而忘记同步修改前端代理地址。

### 3. 启动 Celery 文档索引 Worker（窗口 C，保持窗口打开）

再开一个 PowerShell 窗口，执行：

```powershell
Set-Location 'C:\Users\Star\Desktop\project\rag-center'
.\.venv\Scripts\python.exe -m celery -A app.celery_app worker --loglevel=info --pool=solo
```

看到 `Connected to redis://localhost:6379/0` 和 `ready` 即表示 Worker 已启动。Windows 上要保留 `--pool=solo`。上传文档后的分块、Embedding 和索引工作由这个进程异步完成；只启动 API 而不启动 Worker，文档可能一直停留在 `PROCESSING`。修改 Worker 相关代码或 `.env` 后，按 `Ctrl+C` 并重新启动 Worker。

### 4. 启动前端（窗口 D，保持窗口打开）

再开一个 PowerShell 窗口，执行：

```powershell
Set-Location 'C:\Users\Star\Desktop\project\rag-center\frontend'
npm run dev -- --host 127.0.0.1
```

看到 `Local: http://127.0.0.1:5173/` 后，在浏览器中打开：

- 应用界面：<http://127.0.0.1:5173/>
- 后端 Swagger API 文档：<http://127.0.0.1:8001/docs>

保持 B、C、D 三个 PowerShell 窗口打开；关闭窗口或按 `Ctrl+C` 会停止对应进程。Docker Desktop 也需要继续运行。

## 检查是否真的启动成功

在任意 PowerShell 窗口中执行：

```powershell
Set-Location 'C:\Users\Star\Desktop\project\rag-center'
docker compose ps
Invoke-RestMethod 'http://127.0.0.1:8001/api/v1/knowledge-bases/tree'
Invoke-RestMethod 'http://127.0.0.1:5173/api/v1/knowledge-bases/tree'
```

前两条 API 请求应返回 `code: 0`、`msg: success`。如果 `knowledge_bases` 是空数组，表示服务正常，但还没有创建知识库。第三条请求经过 Vite 前端代理，能检查前端是否连到了 8001 后端。`docker compose ps` 中三个容器应为 `healthy`。

首次使用时，可在前端创建知识库并上传一个小文档：前端发起上传请求 → API 创建任务 → Redis 将任务交给 Celery → Worker 调用 Embedding API 并写入 PostgreSQL / Elasticsearch。上传后留意窗口 C 的处理日志。

本机已经创建了 `tenant_demo` 租户，并将它设为项目内部的 `pro` 档位，方便继续创建自己的知识库和练习混合检索。如果在新数据库上重做练习，需先按“首次安装或环境损坏后重建”中的第 6 步创建它；`AUTH_ENABLED=false` 只选择演示租户 ID，不会自动在数据库插入租户记录。

## 当前配置在哪里

后端读取项目根目录的 `.env`；前端读取 `frontend/.env`。这两个文件已经配置好，并已被 Git 忽略。**不要在日常启动时执行 `Copy-Item .env.example .env`，否则会覆盖现有 API Key 和地址。**

当前关键配置为：

| 位置 | 配置 | 当前用途 |
|---|---|---|
| 根目录 `.env` | `MODEL_BASE_URL`、`MODEL_API_KEY`、`EMBEDDING_MODEL=qwen3.7-text-embedding`、`EMBEDDING_DIMENSIONS=1536` | 千问平台 Embedding |
| 根目录 `.env` | `LLM_BASE_URL`、`LLM_API_KEY`、`LLM_MODEL=deepseek-v4.1-flash` | 通过千问平台调用 DeepSeek Flash |
| 根目录 `.env` | `DATABASE_URL`、`CELERY_BROKER_URL`、`ELASTICSEARCH_URL` | 本机 Docker 服务 |
| 根目录 `.env` | `AUTH_ENABLED=false` | 本地调试使用演示租户 |
| `frontend/.env` | `RAG_API_PROXY_TARGET=http://127.0.0.1:8001` | 把前端 `/api` 请求转给后端 |

需要查看或修改配置时，可以运行 `notepad .env` 或 `notepad frontend\.env`。**不要把实际 API Key 写入 README、代码、截图或公开的 Git 提交。**当前 `QUERY_REWRITE_ENABLED=false`、`EVIDENCE_ENABLED=false`，所以普通检索默认不会调用 LLM；Embedding 在文档索引和向量检索时使用。修改模型或向量维度后，已有文档需要重新索引，不能直接混用旧向量。

## 首次安装或环境损坏后重建

这台电脑现在不需要重复安装。以下步骤供将来重建环境时使用。

1. 安装并确认 Python 3.11+、Node.js 18+、Docker Desktop：

   ```powershell
   py -3.11 --version
   node --version
   npm --version
   docker --version
   ```

2. 在项目根目录创建 Windows Python 虚拟环境并安装依赖。如果 `.venv\Scripts\python.exe` 已存在，跳过创建虚拟环境；不要把来自 macOS/Linux 的 `.venv` 当作 Windows 环境使用。

   ```powershell
   Set-Location 'C:\Users\Star\Desktop\project\rag-center'
   py -3.11 -m venv .venv
   .\.venv\Scripts\python.exe -m pip install --upgrade pip
   .\.venv\Scripts\python.exe -m pip install -e '.[parsers]'
   ```

   直接调用 `.venv\Scripts\python.exe`，无需激活虚拟环境，也无需修改 PowerShell 执行策略。

3. 安装前端依赖：

   ```powershell
   Set-Location 'C:\Users\Star\Desktop\project\rag-center\frontend'
   npm ci
   ```

4. 只有当配置文件确实不存在时，才从示例复制并填写 API Key、模型、数据库等配置：

   ```powershell
   Set-Location 'C:\Users\Star\Desktop\project\rag-center'
   if (-not (Test-Path .env)) { Copy-Item .env.example .env }
   if (-not (Test-Path frontend\.env)) { Copy-Item frontend\.env.example frontend\.env }
   ```

   本机前端代理目标需设为 `RAG_API_PROXY_TARGET=http://127.0.0.1:8001`。根目录 `.env` 中的 Embedding 模型维度必须与接口实际返回的维度一致。

5. Docker 基础服务启动后，执行数据库迁移。首次建库或拉取了新的迁移文件时运行；正常日常启动无需重复：

   ```powershell
   Set-Location 'C:\Users\Star\Desktop\project\rag-center'
   docker compose up -d postgres redis elasticsearch
   .\.venv\Scripts\python.exe -m alembic upgrade head
   .\.venv\Scripts\python.exe -m alembic current
   ```

6. 新数据库还需创建本地演示租户。以下命令仅在租户不存在时创建，已有租户不会重复创建：

   ```powershell
   $tenantExists = docker exec rag-center-postgres psql -U postgres -d rag_center -Atc "SELECT 1 FROM tenants WHERE id = 'tenant_demo';"
   if ($tenantExists -ne '1') {
       .\.venv\Scripts\python.exe scripts\create_tenant.py --id tenant_demo --name '本地学习演示'
   }
   ```

   `AUTH_ENABLED=false` 时，请求会归到 `tenant_demo`，但它仍须在数据库中真实存在。新建租户默认是 `free` 套餐；直接调用检索 API 时要指定 `"profile":"speed"`，因为免费档只允许此档位。

   若要与当前电脑一样，允许更多知识库并练习默认的 `balanced` 混合检索，可在租户创建后执行：

   ```powershell
   .\.venv\Scripts\python.exe scripts\update_tenant_plan.py --tenant-id tenant_demo --plan pro
   ```

   这里的 `pro` 仅是项目数据库中的功能档位，不会改变 Embedding / LLM 平台的计费套餐。

## 常见问题

### Docker 命令提示无法连接 Engine

先在开始菜单打开 Docker Desktop，等它启动完成，再运行 `docker info`。只打开 PowerShell 并不会自动启动 Docker Engine。若 Docker Desktop 自己报错，先查看其错误详情；不要直接选择“Reset to factory defaults”，这会影响本地容器数据。

### 8001 或 5173 端口已占用

先判断是否是之前启动的本项目进程：

```powershell
Get-NetTCPConnection -State Listen -LocalPort 8001,5173 | Select-Object LocalPort,OwningProcess
$targetProcessId = 12345  # 换成上一条命令显示的 OwningProcess 数字
Get-CimInstance Win32_Process -Filter "ProcessId=$targetProcessId" | Select-Object ProcessId,CommandLine
```

如果已经能打开前端和 API 文档，说明项目可能仍在运行，无需再启动一份。若确定是旧的本项目进程，优先回到它所在的终端按 `Ctrl+C`。不要用 `Stop-Process -Name python` 或 `taskkill /IM python.exe`，那会停止其他 Python 程序。

### 前端能打开，但请求失败

确认后端窗口 B 仍在运行，并检查 `frontend/.env` 中的 `RAG_API_PROXY_TARGET` 是否为 `http://127.0.0.1:8001`。修改前端 `.env` 后，在窗口 D 按 `Ctrl+C` 并重新执行 `npm run dev -- --host 127.0.0.1`。

### 上传后一直显示 PROCESSING

检查窗口 C 的 Celery Worker 是否显示 `ready`；再执行 `docker compose ps`，确认 Redis 和数据库健康。Worker 没有运行时，API 可以接收上传请求，但不会处理异步索引任务。

### Embedding 或 LLM 返回 401 / 余额错误

在对应平台核对 API Key、余额、Base URL、模型名。Embedding 使用根目录 `.env` 的 `MODEL_*`，LLM 使用 `LLM_*`；它们可以指向不同平台。修改后重启后端和 Worker。不要把完整 Key 粘贴到错误报告或终端共享截图中。

## 手工停止

1. 分别在前端、Worker、API 的 PowerShell 窗口按 `Ctrl+C`。
2. 如需停止 Docker 基础服务，在项目根目录运行：

   ```powershell
   docker compose stop postgres redis elasticsearch
   ```

`docker compose stop` 不会删除数据库和 Elasticsearch 的持久化卷。下次按本文第一节启动即可。不要使用 `docker compose down -v`，该命令会删除本地数据卷。
