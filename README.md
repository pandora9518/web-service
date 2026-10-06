# duyi-service

基于 **FastAPI + SQLAlchemy 2.0 (async) + PostgreSQL** 的电商后端服务（商品 / 分类 / SKU），采用 uv workspace 管理的多包工程。

> 学习项目：Python 框架课程实战

## 技术栈

| 类别     | 选型                                             |
| -------- | ------------------------------------------------ |
| Web 框架 | FastAPI（ASGI，Uvicorn 驱动）                    |
| 数据库   | PostgreSQL 16 + asyncpg（异步驱动）              |
| ORM      | SQLAlchemy 2.0（DeclarativeBase + AsyncSession） |
| 数据迁移 | Alembic（autogenerate）                          |
| 配置管理 | pydantic-settings（.env + env_prefix 分组）      |
| 包管理   | uv（workspace monorepo）                         |
| 调试     | debugpy（attach 模式）                           |

## 工程结构

```
web-service/
├── apps/web-service/
│   ├── alembic.ini              # 迁移配置
│   ├── migrations/              # 迁移脚本（Alembic）
│   └── app/
│       ├── main.py              # 入口：装配 app、路由、中间件、异常处理
│       ├── api/                 # 路由层（感知 HTTP）
│       ├── service/             # 业务层（DTO 进出，不感知 HTTP）
│       ├── model/               # 数据层（ORM 模型、关联表）
│       ├── schema/              # Pydantic DTO（请求/响应契约）
│       ├── exception/           # 业务异常体系 + 全局异常处理器
│       └── core/                # 基础设施：配置、数据库、中间件、openapi
├── Makefile                     # 常用命令封装
├── pyproject.toml               # uv workspace 根
└── .env.example                 # 环境变量模板
```

## 快速开始

### 1. 准备

- Python ≥ 3.14、[uv](https://docs.astral.sh/uv/)
- PostgreSQL 16（可用 Docker 一行启动）：

```bash
docker run -d --name pg_db \
  -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=123123 \
  -p 5432:5432 -v pgdata:/var/lib/postgresql/data postgres:16

docker exec -it pg_db psql -U admin -c "CREATE DATABASE duyi_db;"
```

### 2. 安装依赖

```bash
uv sync --all-packages
```

### 3. 配置环境变量

```bash
cp .env.example .env   # 按需修改数据库连接等
```

### 4. 初始化表结构

```bash
make db-upgrade        # 执行 Alembic 迁移
```

### 5. 启动

```bash
make dev               # http://localhost:8080
```

接口文档：`/docs`（Swagger UI，交互调试）、`/redoc`（只读文档）。生产环境自动关闭。

## 常用命令

| 命令                            | 说明                                    |
| ------------------------------- | --------------------------------------- |
| `make dev`                      | 启动开发服务器（:8080）                 |
| `make debug`                    | 以 debugpy 启动（:5678，VSCode attach） |
| `make db-migrate message="xxx"` | 改模型后生成迁移脚本（需人工审查）      |
| `make db-upgrade`               | 升级到最新表结构                        |
| `make db-downgrade version=-1`  | 回退一步迁移                            |

## 架构说明

```
请求 → 中间件洋葱 → 路由(api) → 依赖注入(get_db) → 业务层(service) → 数据层(model)
                ↑统一响应壳        ↑Pydantic校验      ↑请求级事务      ↑flush不commit
```

- **三层职责**：api 只管 HTTP 语义；service 承载业务规则（DTO 进出）；model 只管存取
- **请求级事务**：`Depends(get_db)` 每请求一个 session（`begin()` 自动 commit/rollback），service 内仅 `flush`
- **统一异常**：`BusinessException` 体系 + 全局 handler 按 MRO 查 `ERROR_MAP`，响应统一为 `{code, message}`
- **统一响应**：中间件将 `/api/` 响应包装为 `{code, data, message}`，openapi 同步生成 `ApiResponse_Xxx` schema

## 环境变量

见 [.env.example](.env.example)，主要包括：`WEB_APP_NAME`、`WEB_CORS_ORIGINS`、`DB_HOST/PORT/NAME/USER/PASSWORD`。
