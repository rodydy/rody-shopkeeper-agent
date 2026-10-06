# AGENTS.md

本文件面向 AI 编码智能体，介绍本仓库的结构、技术栈与开发约定。项目文档与代码注释以中文为主，本文件同样使用中文。

## 项目概述

本项目是「电商问数」智能数据分析 Agent（`shopkeeper-agent`，`src/pyproject.toml` 中的项目名），一个用于系统学习 LangGraph 的实战教学项目。它围绕电商数仓问数场景实现完整链路：

1. **元数据知识库构建**：从教学数仓抽取表、字段、指标和字段取值，分别写入 MySQL（结构化元数据）、Qdrant（字段/指标向量）和 Elasticsearch（字段取值全文索引）。
2. **自然语言问数**：用户提问后，LangGraph 工作流完成关键词抽取、多路召回、上下文过滤、SQL 生成/校验/修正/执行，并通过 FastAPI 的 SSE 接口把过程流式返回给 React 前端。

注意：项目刻意**不覆盖**生产级治理能力（登录鉴权、多租户、SQL 安全审计、限流、评测集、监控告警等），属于学习定位，不要擅自引入这类重型框架。

## 仓库布局

仓库根目录下的 `src/` 才是实际项目目录，**所有后端命令都应在 `src/` 目录下执行**：

```text
src/
├── app/                    # 后端 Python 包
│   ├── agent/              # LangGraph 图编排：graph.py、state.py、context.py、llm.py
│   │   └── nodes/          # 12 个问数节点（关键词抽取、三路召回、过滤、SQL 生成/校验/修正/执行）
│   ├── api/                # FastAPI 层：routers/、schemas/、dependencies.py（依赖组装）、lifespan.py（生命周期）
│   ├── clients/            # 外部服务客户端管理器（mysql / qdrant / es / embedding，应用级单例，init/close）
│   ├── conf/               # 配置 dataclass 与 OmegaConf 加载逻辑
│   ├── core/               # loguru 日志封装与 request_id ContextVar
│   ├── entities/           # 业务语义数据对象（dataclass）
│   ├── models/             # SQLAlchemy ORM 模型
│   ├── prompt/             # Prompt 模板加载工具
│   ├── repositories/       # 数据访问层：mysql/{meta,dw}、qdrant/、es/
│   ├── scripts/            # build_meta_knowledge.py 元数据知识库构建入口
│   └── services/           # MetaKnowledgeService（构建）、QueryService（问数编排）
├── conf/                   # app_config.yaml（应用配置）、meta_config.yaml（待同步的表/字段/指标定义）
├── docker/                 # docker-compose.yaml、MySQL 初始化 SQL（meta.sql、dw.sql）、ES 插件、embedding 模型挂载目录
├── docs/                   # README 引用的图片
├── examples/               # qdrant_quickstart_demo.py 独立演示
├── frontend/               # React + Vite + Tailwind 前端
├── prompts/                # SQL 生成、修正、过滤等 .prompt 模板
├── test/                   # 空目录，项目暂无自动化测试
├── main.py                 # FastAPI 应用入口（注册 lifespan、路由、request_id 中间件）
└── pyproject.toml          # Python 依赖与 Ruff 配置（uv.lock 锁定版本）
```

## 技术栈

- **后端**：Python >= 3.14、FastAPI、LangGraph、LangChain（`langchain-huggingface`、`langchain-deepseek`）、SQLAlchemy 2.x 异步（asyncmy 驱动）、Qdrant Client、elasticsearch[async] 8.x、jieba（关键词抽取）、OmegaConf + PyYAML（配置）、loguru（日志）、python-dotenv。
- **基础设施（Docker Compose）**：MySQL 8.0（元数据库 `meta` + 教学数仓 `dw`）、Elasticsearch + Kibana、Qdrant v1.16、TEI（text-embeddings-inference）加载 `BAAI/bge-large-zh-v1.5`（1024 维向量）。
- **前端**：React 19、Vite 6、TypeScript、Tailwind CSS 3，包管理器 pnpm 10。
- **依赖管理**：后端 `uv`，前端 `pnpm`。

## 构建与运行命令

以下命令均在 `src/` 目录下执行（前端命令在 `src/frontend/` 下执行）：

```bash
# 1. 安装后端依赖
uv sync

# 2. 配置大模型 API Key（.env 已被 .gitignore 忽略）
cp .env.example .env        # 然后把 LLM_API_KEY 替换为真实密钥

# 3. 下载 Embedding 模型到 Docker 挂载目录（体积较大，不进仓库）
uv run hf download BAAI/bge-large-zh-v1.5 --local-dir docker/embedding/bge-large-zh-v1.5

# 4. 启动基础服务（MySQL 3306 / ES 9200 / Kibana 5601 / Qdrant 6333 / Embedding 8081）
docker compose -f docker/docker-compose.yaml up -d
# MySQL 首次启动自动执行 docker/mysql/meta.sql 和 dw.sql 完成建库建表

# 5. 构建元数据知识库（写入 MySQL、Qdrant、ES）
uv run python -m app.scripts.build_meta_knowledge -c conf/meta_config.yaml

# 6. 启动后端（默认 http://127.0.0.1:8000）
uv run fastapi dev main.py

# 7. 启动前端（默认 5173，/api 由 Vite 代理到 VITE_DEV_PROXY_TARGET，默认 127.0.0.1:8000）
cd frontend && pnpm install && pnpm dev
```

前端其他命令：`pnpm build`（tsc 类型检查 + 构建）、`pnpm lint`（仅 `tsc --noEmit` 类型检查，无 ESLint）。

## 接口约定

- 唯一业务接口：`POST /api/query`，请求体 `{"query": "统计华北地区的销售总额"}`。
- 响应为 SSE（`text/event-stream`），每条消息格式为 `data: {json}\n\n`，消息类型有三种：
  - `progress`：节点进度（`step`、`status: running|success|error`），由各节点通过 `runtime.stream_writer` 写出；
  - `result`：最终查询结果；
  - `error`：全局异常（流式响应开始后无法改状态码，异常也被包装成 SSE 消息）。
- 前端在 `src/frontend/src/lib/agentApi.ts` 中手动解析 SSE 流，修改消息格式时需前后端同步改。

## Agent 工作流（src/app/agent/graph.py）

节点执行顺序：`extract_keywords` → 并行 `recall_column` / `recall_value` / `recall_metric` → `merge_retrieved_info` → 并行 `filter_table` / `filter_metric` → `add_extra_context` → `generate_sql` → `validate_sql` → 条件分支（无错误走 `run_sql`，有错误走 `correct_sql` → `run_sql`）→ `END`。

关键约定：

- **State 与 Context 分离**：`DataAgentState`（state.py）只放节点间读写合并的业务数据；`DataAgentContext`（context.py）放外部依赖（各 Repository、Embedding 客户端），节点内通过 `runtime.context` 读取，不要把工具对象塞进 State。
- 每个节点用 `writer({"type": "progress", ...})` 上报进度，用 `from app.core.log import logger` 记日志，出错时先上报 `error` 状态再 `raise`。
- 本地调试整条链路可直接运行 `python -m app.agent.graph`（文件内有 `__main__` 调试块，需先启动 Docker 服务）。
- Prompt 模板放在 `src/prompts/*.prompt`，代码通过 `app/prompt/prompt_loader.py` 加载。

## 配置管理

- `src/conf/app_config.yaml` 是应用主配置（日志、两套 MySQL、Qdrant、Embedding、ES、LLM）。`src/app/conf/app_config.py` 用 dataclass 定义结构、OmegaConf 做 schema 合并，产出全局 `app_config` 对象；新增配置项必须同时在 dataclass 和 YAML 中添加。
- 敏感值通过 `${oc.env:...}` 从 `src/.env` 注入（目前只有 `LLM_API_KEY`）。默认 LLM 是硅基流动的 OpenAI 兼容接口（`base_url: https://api.siliconflow.cn/v1`）。
- `src/conf/meta_config.yaml` 定义教学数仓的表、字段（含角色、别名、`sync` 开关）和指标，构建脚本据此同步元数据；其中 `sync: true` 的字段才会把真实取值写入 ES。
- 应用级客户端（MySQL engine、Qdrant、ES、Embedding）由 `app/clients/*_client_manager.py` 统一管理，在 FastAPI lifespan 中 `init()` 一次、关闭时 `close()`；请求级资源（Session、Repository、Service）通过 `app/api/dependencies.py` 的 `Depends` 链组装，路由函数不直接创建基础设施对象。

## 代码风格

- 代码注释、docstring、日志文案一律使用**中文**，与现有代码保持一致。本项目是教学项目，每个模块有中文模块级 docstring 说明职责，关键逻辑配有解释性中文注释，新增代码应延续这一风格。
- Ruff 负责 lint 与格式化：`line-length = 88`，`target-version = py314`，lint 规则启用 `E, F, I`、忽略 `E501`。配置 pre-commit 后提交时会自动跑 `ruff check --fix` 和 `ruff format`（见 `src/.pre-commit-config.yaml`）；手动检查可用 `uv run ruff check . && uv run ruff format --check .`。
- `.editorconfig`：UTF-8、LF 换行、Python/SQL 4 空格缩进、YAML/TOML 2 空格缩进。
- 分层约定：Router → Service → Repository → Client Manager；MySQL 元数据访问在 Repository 内再经 `mappers/` 做 ORM 与 entity 转换。保持依赖单向，不要在 Repository 里写业务编排。
- 日志统一用 `app.core.log.logger`（loguru），自动注入 `request_id`（由 `main.py` 中间件写入 ContextVar），不要自己 `print` 或新建 logger。

## 测试说明

项目**没有自动化测试套件**：`src/test/` 为空目录，无 pytest 配置，CI 也未配置。验证变更的方式：

1. `uv run ruff check .` 保证静态检查通过；
2. 本地跑通真实链路：启动 Docker 服务后执行 `uv run python -m app.agent.graph`（内置调试入口），或启动后端后用 `curl -N -X POST http://127.0.0.1:8000/api/query -H "Content-Type: application/json" -d '{"query":"统计华北地区的销售总额"}'` 观察 SSE 输出；
3. 前端改动跑 `pnpm lint`（tsc 类型检查）和 `pnpm build`。

## 安全注意事项

- `LLM_API_KEY` 等密钥只放 `src/.env`，该文件已被 `.gitignore` 忽略，**禁止提交**；`.env.example` 只保留占位符。
- MySQL 默认账号密码（`didilili/dili123`、root `dili123`）是 Docker 教学环境默认值，仅用于本地开发，不要当成真实凭据处理，也不要将生产凭据写入 `conf/*.yaml`。
- 项目生成的 SQL 会真实在数仓执行，目前仅有 EXPLAIN 校验和错误修正，没有 SQL 安全审计与白名单——修改 SQL 生成/执行链路时注意这一边界，避免扩大执行范围（如引入写操作）。
- 前端无鉴权，`/api/query` 对内网完全开放，部署时需注意暴露面。

## 其他说明

- Git 分支与配套教程章节一一对应（`04-structure-config` … `17-api-integration-logging`），`main` 是完整闭环版本；改动应基于 `main`。
- 上游教程仓库：[didilili/shopkeeper-agent](https://github.com/didilili/shopkeeper-agent)，完整图文讲义见 README。
