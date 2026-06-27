# 后端实现与代码架构设计

## 1. 文档目的

本文档用于定义项目后端目录结构、分层职责、模块划分与实现落地方式，作为后续 FastAPI + SQLAlchemy + Alembic 开发的工程骨架说明。

本设计与 [`prd/design_v1.md`](/Users/sitaomin1994/projects/mistria_atlas/prd/design_v1.md) 和 [`prd/design_metadata.md`](/Users/sitaomin1994/projects/mistria_atlas/prd/design_metadata.md) 配套使用：

1. `design_v1.md` 负责总体 PRD 与架构草案。
2. `design_metadata.md` 负责元数据管理模块详细设计。
3. 本文档负责后端工程组织方式与代码目录架构。

---

## 2. 后端目标

后端需要满足以下目标：

1. 支撑模块化单体架构。
2. 保持目录清晰，方便逐模块扩展。
3. 支持 FastAPI REST API、SQLAlchemy ORM、Alembic 迁移。
4. 支持异步任务扩展。
5. 支持数据库型与文件型数据源 connector 扩展。
6. 为后续 Profiling、数据标准、数据质量与 AI 模块保留稳定结构。

---

## 3. 总体分层

后端建议采用“接口层 + 应用服务层 + 仓储层 + 模型层 + 基础设施层”的结构。

### 3.1 接口层

负责：

1. FastAPI 路由定义
2. 请求参数解析
3. 响应序列化
4. HTTP 错误转换

### 3.2 应用服务层

负责：

1. 业务流程编排
2. 调用 connector、repository、task
3. 状态流转控制
4. 聚合对象输出

### 3.3 仓储层

负责：

1. 数据库 CRUD
2. 条件查询与分页
3. 领域对象持久化

### 3.4 模型层

负责：

1. SQLAlchemy ORM 模型
2. 领域实体的关系定义
3. 与 Alembic 同步的数据库结构

### 3.5 基础设施层

负责：

1. 数据库连接
2. 配置加载
3. 日志
4. 任务调度
5. connector 适配
6. 通用工具

---

## 4. 目录结构

推荐目录树如下：

```text
backend/
  app/
    api/
      v1/
        endpoints/
    connectors/
    core/
    db/
    models/
    repositories/
    schemas/
    services/
    tasks/
    utils/
  migrations/
    versions/
  tests/
    integration/
    unit/
```

---

## 5. 目录职责说明

### 5.1 backend/app/api

作用：FastAPI 接口层。

建议后续放置：

1. 路由注册
2. 版本化 API
3. 各模块 endpoint 文件

例如：

1. `app/api/v1/endpoints/datasource.py`
2. `app/api/v1/endpoints/metadata.py`
3. `app/api/v1/endpoints/glossary.py`
4. `app/api/v1/endpoints/lineage.py`

### 5.2 backend/app/core

作用：全局配置与应用级基础设施。

建议后续放置：

1. `config.py`
2. `logging.py`
3. `exceptions.py`
4. `security.py`

### 5.3 backend/app/db

作用：数据库基础设施。

建议后续放置：

1. `base.py`
2. `session.py`
3. `naming.py`

### 5.4 backend/app/models

作用：SQLAlchemy ORM 模型。

建议后续放置：

1. `datasource.py`
2. `metadata.py`
3. `glossary.py`
4. `relation.py`
5. `profiling.py`
6. `quality.py`

### 5.5 backend/app/schemas

作用：Pydantic 请求与响应模型。

建议后续放置：

1. `datasource.py`
2. `dataset.py`
3. `field.py`
4. `glossary.py`
5. `metadata_search.py`

### 5.6 backend/app/repositories

作用：数据库访问层。

建议后续放置：

1. `datasource_repository.py`
2. `dataset_repository.py`
3. `field_repository.py`
4. `glossary_repository.py`
5. `scan_task_repository.py`

### 5.7 backend/app/services

作用：应用服务与业务编排。

建议后续放置：

1. `datasource_service.py`
2. `metadata_scan_service.py`
3. `dataset_service.py`
4. `field_service.py`
5. `glossary_service.py`
6. `lineage_service.py`

### 5.8 backend/app/connectors

作用：外部数据源适配层。

建议后续放置：

1. `base.py`
2. `postgres_connector.py`
3. `mysql_connector.py`
4. `sqlite_connector.py`
5. `csv_connector.py`
6. `excel_connector.py`
7. `parquet_connector.py`
8. `json_connector.py`

### 5.9 backend/app/tasks

作用：后台任务入口。

建议后续放置：

1. `metadata_tasks.py`
2. `profiling_tasks.py`
3. `quality_tasks.py`

说明：
首期可先用 FastAPI BackgroundTasks，后续平滑升级到 Celery。

### 5.10 backend/app/utils

作用：通用工具函数。

建议后续放置：

1. `time.py`
2. `pagination.py`
3. `sql_builder.py`
4. `type_inference.py`

### 5.11 backend/tests

作用：测试目录。

建议分层：

1. `tests/unit` 放单元测试
2. `tests/integration` 放集成测试

---

## 6. 模块到目录的映射

### 6.1 元数据管理模块

建议主要落在以下目录：

1. `app/api/v1/endpoints`
2. `app/models`
3. `app/schemas`
4. `app/repositories`
5. `app/services`
6. `app/connectors`

### 6.2 数据标准模块

建议后续新增：

1. `app/models/standard*.py`
2. `app/schemas/standard*.py`
3. `app/repositories/standard*_repository.py`
4. `app/services/standard*_service.py`

### 6.3 Profiling 模块

建议后续重点使用：

1. `app/connectors`
2. `app/services`
3. `app/tasks`
4. `app/models/profiling.py`

### 6.4 数据质量模块

建议后续重点使用：

1. `app/services`
2. `app/tasks`
3. `app/models/quality.py`

### 6.5 AI 模块

建议后续以独立 service 为主，但当前先保留：

1. `app/services/ai_*`
2. `app/models/ai_*`

---

## 7. 推荐启动结构

后续建议把当前顶层的 `backend/main.py` 逐步转为兼容入口，真正应用入口迁移到：

1. `backend/app/main.py`

并由该入口完成：

1. 加载配置
2. 初始化 FastAPI
3. 注册路由
4. 挂载生命周期事件
5. 初始化异常处理与日志

---

## 8. 元数据模块首期文件建议

基于当前已完成的元数据详细设计，建议第一批实际文件如下：

### 8.1 配置与基础设施

1. `backend/app/core/config.py`
2. `backend/app/db/base.py`
3. `backend/app/db/session.py`

### 8.2 ORM 模型

1. `backend/app/models/datasource.py`
2. `backend/app/models/metadata.py`
3. `backend/app/models/glossary.py`
4. `backend/app/models/relation.py`

### 8.3 Schema

1. `backend/app/schemas/datasource.py`
2. `backend/app/schemas/dataset.py`
3. `backend/app/schemas/field.py`
4. `backend/app/schemas/glossary.py`

### 8.4 Repository

1. `backend/app/repositories/datasource_repository.py`
2. `backend/app/repositories/dataset_repository.py`
3. `backend/app/repositories/field_repository.py`
4. `backend/app/repositories/glossary_repository.py`
5. `backend/app/repositories/scan_task_repository.py`

### 8.5 Service

1. `backend/app/services/datasource_service.py`
2. `backend/app/services/metadata_scan_service.py`
3. `backend/app/services/dataset_service.py`
4. `backend/app/services/field_service.py`
5. `backend/app/services/glossary_service.py`
6. `backend/app/services/lineage_service.py`

### 8.6 API

1. `backend/app/api/v1/endpoints/datasource.py`
2. `backend/app/api/v1/endpoints/metadata.py`
3. `backend/app/api/v1/endpoints/glossary.py`
4. `backend/app/api/v1/endpoints/lineage.py`

---

## 9. 开发顺序建议

### 9.1 第一步

先完成基础设施：

1. 配置
2. 数据库 Session
3. Base ORM
4. Alembic 对接

### 9.2 第二步

完成元数据主表模型：

1. datasource
2. dataset
3. field
4. glossary_term
5. lineage_edge
6. metadata_scan_task

### 9.3 第三步

完成数据源与元数据查询接口：

1. 数据源 CRUD
2. 测试连接
3. 数据集列表
4. 数据集详情
5. 字段详情

### 9.4 第四步

完成扫描任务与 connector。

### 9.5 第五步

补齐标签、术语、关系与血缘。

---

## 10. 当前已落地目录

当前仓库已生成以下后端目录骨架：

```text
backend/
  app/
    api/
      v1/
        endpoints/
    connectors/
    core/
    db/
    models/
    repositories/
    schemas/
    services/
    tasks/
    utils/
  tests/
    integration/
    unit/
```

这套骨架用于承接后续元数据模块与其他模块实现，避免后面边开发边重构目录。

