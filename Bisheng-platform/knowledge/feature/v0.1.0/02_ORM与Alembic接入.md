# v0.1.0 ORM 与 Alembic 接入

## 原有问题与本阶段范围

重构目录已有数据库连接和 Session 管理，但尚无领域 ORM、统一 metadata 或迁移入口。本阶段只建立新服务的库表结构和 schema 迁移能力，不搬迁、不读取、不转换原 `knowledge-backend` 的业务数据。

## 实际修改与实现结果

- 按领域新增 7 张 ORM 表：模型配置领域的 `model_provider`、`model`；目录领域的 `knowledge_library`、`knowledge_folder`、`knowledge_document`、`document_version`；入库领域的 `knowledge_file`。
- `knowledge_document.primary_version_id` 是主版本唯一指针。`document_version` 不再定义 `is_primary`，并通过唯一约束保证同一文档的 `version_no` 不重复。
- 模型提供方、模型、库、文件夹、文档、版本与文件之间建立本服务内的外键；模型绑定的类型和租户一致性仍由应用服务校验。未创建租户、用户、权限、检索切片或 Celery 任务 ORM。
- 新迁移位于 `migrations-refactor/`，配置入口为 `alembic-refactor.ini`，revision 为 `v010001`。新迁移流使用独立的 `alembic_version_refactor`，默认 `alembic.ini` 与原 `migrations/` 保持原样。
- Alembic 环境加载新重构目录的 SQLModel metadata。SQLite 使用同步驱动执行迁移；MySQL 使用异步 `aiomysql` 驱动，并兼容将 `mysql`/`mysql+pymysql` URL 转换为 `mysql+aiomysql`。
- 应用启动不执行迁移，也不调用 `create_all`。schema 由显式 Alembic revision 管理；revision 不包含旧数据迁移。
- `pyproject.toml` 将重构目录纳入 wheel 包路径。

## 方法与设计取舍

模型按模型配置、目录和入库领域分别归属，统一通过 `domain/models.py` 导入，确保 Alembic autogenerate 和差异检查看到完整 metadata。使用 SQLModel 延续当前重构基础设施的 ORM 技术选择，并在同一个领域模型定义中显式声明数据库约束。

保留原默认 Alembic 配置与迁移目录，给重构服务单独配置文件、迁移目录和版本表，避免两个 migration stream 互相读取 revision。迁移始终显式运行，不在应用 lifespan 中创建或修改业务表。

`knowledge_document` 到 `document_version` 存在循环关系：版本行引用逻辑文档，逻辑文档的主版本指针引用版本行。迁移先建两张表，再补主版本外键；SQLite 使用 Alembic batch 操作以兼容其不支持直接 `ALTER TABLE ADD CONSTRAINT` 的限制。

## 验证情况

- 在工作区 `.data/` 中的独立 SQLite schema 数据库上执行 `upgrade head`、`alembic check`、`downgrade base`、再次 `upgrade head`；均成功，升级后版本为 `v010001`，metadata 差异检查无待生成操作。
- 迁移隔离测试预置无关表和记录，验证升级保留其数据、降级移除本次 7 张领域表并保留既有表和记录。`document_version` 不含 `is_primary`，`knowledge_document.primary_version_id` 外键存在。
- 使用 `mysql+pymysql` URL 执行 `upgrade head --sql`，成功生成 MySQL DDL；未连接真实 MySQL，因此未验证服务器版本差异或实际执行行为。
- 默认旧 Alembic 入口仍报告原 head `0014_oxr_settings`。
- 新 migration 测试通过；本阶段涉及的 Alembic 与领域模型代码通过 Ruff 检查和格式检查。已有 `tests/refactor/test_bootstrap.py` 曾在本轮运行中卡在 MinIO 异步 mock 测试，未能完成全套结果；此前该测试文件记录为 15 项通过。

## 影响与后续事项

新服务需要针对目标数据库执行 `alembic -c alembic-refactor.ini upgrade head`。运行前必须通过 `KNOWLEDGE_DATABASE_URL` 指向新服务目标库；该命令只创建本阶段定义的结构，不承接旧服务数据。旧服务仍使用默认 Alembic 配置。

当前未连接真实 MySQL。下一步可继续对照架构设计和原 Bisheng 模型细化字段语义、状态枚举及模型配置写入规则；任何 schema 变更继续新增 Alembic revision，不回写已应用迁移。
