# 知识图谱检索栈（Neo4j + LightRAG）

为多路召回新增「知识图谱」通道所需的基础设施。本栈只新增两个进程：

- **Neo4j 5.26 社区版**：图存储，附带 `apoc` + `graph-data-science`（GDS 社区版含 Leiden/Louvain 社区发现，单机 4 核封顶）
- **LightRAG**（:9621）：图谱构建 + 检索微服务，提供 REST API 与 WebUI

KV / 向量 / 文档状态**复用宿主机已有的 PostgreSQL**（后端同一实例，需已装 pgvector），不再单独起库。LightRAG 自建 `LIGHTRAG_*` 前缀表，与业务 `t_*` 表互不干扰。

> 图谱通道是**增强而非必须**：后端默认 `rag.graph.type=none`，不启用本栈时行为零变化。

## 前置条件

- 宿主机 PostgreSQL 可访问，且已安装 `pgvector`（与后端共用，库名默认 `ragent`）
- 已准备百炼 API Key（LLM 与 Embedding 共用）
- 已安装 Docker / Docker Compose

Compose 从同目录 `.env` 读取本地参数；仓库只提交 `.env.example`，实际 `.env` 已加入 `.gitignore`。默认 PostgreSQL 配置与项目 `application.yaml` 一致：`postgres/postgres`、数据库 `ragent`。

`LLM_BINDING` 与 `EMBEDDING_BINDING` 仍配置为 `openai`，这是 LightRAG 对 OpenAI 兼容接口的适配器名称；实际供应商由 `*_BINDING_HOST`、模型名和 API Key 决定，当前均指向阿里云百炼。

## 启动

```bash
cd resources/docker/graphrag
cp .env.example .env
# 编辑 .env，填写 BAILIAN_API_KEY
docker compose -f lightrag-neo4j-stack.compose.yaml up -d
```

### 在 IDEA 中启动

1. 在 `resources/docker/graphrag` 目录将 `.env.example` 复制为 `.env`，填写 `BAILIAN_API_KEY`
2. 打开 `lightrag-neo4j-stack.compose.yaml`
3. 点击编辑器左侧 Compose 的启动按钮，直接启动整个栈

Docker Compose 默认把首个 Compose 文件所在目录作为项目目录，因此会自动加载旁边的 `.env`。如果使用的是手工创建且修改过项目目录的 IDEA Run Configuration，可在 `Run | Edit Configurations` 中为该 Docker Compose 配置选择此 `.env` 文件。

> `.env` 的自动加载只适用于这个 Docker Compose 启动。IDEA 中单独启动 Spring Boot 时，如需同一组 API Key，还要在 Spring Boot Run Configuration 的 Environment variables 中选择该 `.env`。

首次拉起会构建本地 `ragent/neo4j-gds:5.26-2.13.11` 镜像，并下载一次约 62 MB 的 GDS JAR；该步骤可能需要一到几分钟。GDS 随后固化在镜像层中，普通启动与重启不再访问插件下载站。`docker compose ps` 两个服务均 `healthy` 后即就绪。

如果 PostgreSQL 也运行在本机 Docker，且 LightRAG 经 `host.docker.internal:5432` 连接时出现 `unexpected connection_lost`，可让两个容器直接通信。首次启动编排创建 `lightrag-net` 后，执行 `docker network connect lightrag-net postgres`（`postgres` 为已有数据库容器名），把本目录 `.env` 的 `POSTGRES_HOST` 设为 `postgres`，再执行 `docker compose -f lightrag-neo4j-stack.compose.yaml up -d --no-deps lightrag`。现有 PostgreSQL 会继续运行；如果以后重建其容器，需要重新接入这个网络。

### Windows 本机运行 LightRAG

若 Docker 内访问百炼出现 TLS 握手失败，但 Windows 能正常直连，可把 LightRAG 运行在 Windows 上。Neo4j、PostgreSQL 继续在 Docker 中运行。Python 服务读取本目录的 `.env`，沿用相同的数据库和图谱，监听 `127.0.0.1:9621`。以下命令在仓库根目录运行。

若数据库使用不同的宿主机端口，在本目录的 `.env` 设置 `POSTGRES_HOST_PORT` 为实际发布端口，例如 `15432`。`POSTGRES_PORT` 继续用于容器间连接。

```powershell
docker compose -f resources/docker/graphrag/lightrag-neo4j-stack.compose.yaml stop lightrag
docker compose -f resources/docker/graphrag/lightrag-neo4j-stack.compose.yaml up -d neo4j
# 首次创建环境时使用 Python 3.12
py -3.12 -m venv .output/lightrag-venv
.output/lightrag-venv/Scripts/python.exe -m pip install 'lightrag-hku[api]==1.5.7' asyncpg pgvector neo4j truststore
# 之后启动只需执行以下两行
$env:PYTHONUTF8 = '1'
.output/lightrag-venv/Scripts/python.exe resources/docker/graphrag/start-lightrag-local.py
```

## 验证

| 检查项 | 方式 |
|--------|------|
| LightRAG 存活 | `curl http://localhost:9621/health` 返回 `{"status":"healthy",...}` |
| LightRAG WebUI | 浏览器打开 http://localhost:9621/webui |
| Neo4j Browser | 浏览器打开 http://localhost:7474 （账号密码见 `.env` 的 `NEO4J_USERNAME` / `NEO4J_PASSWORD`） |
| 写入连通 | `curl -X POST http://localhost:9621/documents/text -H 'Content-Type: application/json' -d '{"text":"张三是某公司CEO，公司总部在杭州。","file_source":"smoke_test"}'`，稍候在 Neo4j Browser 执行 `MATCH (n) RETURN n LIMIT 25` 可见抽取出的实体与关系 |

> 清空图数据：`curl -X DELETE http://localhost:9621/documents`

## 对接后端

栈就绪后，在后端 `application.yaml` 打开图谱通道并重启：

```yaml
rag:
  graph:
    type: lightrag                        # 由 none 改为 lightrag，注册 LightRagClient + GraphSearchChannel
    lightrag:
      base-url: http://127.0.0.1:9621     # 本机默认；后端与本栈不同机时改为对应地址
      query-mode: hybrid                  # 只回图结构证据，与向量 / 关键词通道正交（可选 naive / local / global / hybrid / mix）
    embedding-model: text-embedding-v4    # 管理端展示的模型名，与 Compose 保持一致
  search:
    channels:
      graph:
        enabled: true                     # 检索时并入图谱通道
```

默认只在本机访问 LightRAG，后端可留空 `rag.graph.lightrag.api-key`。启用了 LightRAG 鉴权时填写该项，客户端随请求发送 `X-API-Key`。

开启后，新增或按整篇重建的文档会由 `GraphSyncingVectorStoreService` 提交给 LightRAG。已有文档需要单独提交或重建才能在图谱中看到。整篇删除会同步清理图谱，单独编辑分块后需要重建整篇才能更新图谱。图谱同步失败会记入日志，原向量写入仍然有效。

## 注意事项

- 后端 `rag.default.dimension` 与 Compose 的 `EMBEDDING_DIM` 当前均为 `1536`；`EMBEDDING_SEND_DIM=true` 会要求百炼返回相同维度
- **Embedding 模型或维度首次索引后不可更换**。修改前需清空 LightRAG 存储（PG 中 `LIGHTRAG_*` 表 + Neo4j 数据）再重新索引
- Compose 中的账号密码只面向本机开发，请勿把当前端口和固定密码直接用于生产或暴露到公网
- **镜像 tag 建议 pin 到已验证的具体版本**，勿长期用 `latest`（社区仓库历史上出现过超前日期版本 / 虚构模型名）
- LightRAG 与后端**共用宿主机 PostgreSQL**：卸载本栈只需 `docker compose down`，如需彻底清库再 `docker compose down -v`（删 Neo4j 数据卷）并手动清理 PG 中 `LIGHTRAG_*` 表
- 同一 LightRAG 实例中的知识库共享图，检索通道按返回的 `file_path` 识别证据所属文档和知识库。
- 内存偏紧的机器可下调 compose 中 Neo4j 的 `server_memory_heap_max__size` / `pagecache_size`
