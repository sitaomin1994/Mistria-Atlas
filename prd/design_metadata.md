# 元数据管理模块详细设计

## 1. 文档目的

本文档用于细化轻量级数据治理平台中“元数据管理模块”的产品与技术设计，作为后续后端开发、前端实现、数据库建模与接口联调的依据。

该模块是整个平台的数据资产底座，负责将数据源中的技术元数据与人工维护的业务语义统一建模，并向 Profiling、数据标准、数据质量与 AI 检索能力提供稳定的资产对象与查询入口。

---

## 2. 模块目标

元数据管理模块的核心目标如下：

1. 管理数据源及连接信息。
2. 扫描数据库与文件型数据源，生成统一的数据资产目录。
3. 管理数据集、字段、标签、主题域、术语与关系信息。
4. 支持自动采集与人工维护并存，且互不覆盖。
5. 为后续 Profiling、标准映射、质量规则和 AI 能力提供稳定对象 ID 与上下文基础。

---

## 3. 模块边界

### 3.1 本模块负责

1. 数据源注册、编辑、启停、连通性测试。
2. Schema 扫描与文件结构解析。
3. Dataset / Field 资产目录管理。
4. 标签、主题域、术语及术语映射管理。
5. Dataset / Field 关系与基础血缘管理。
6. 元数据搜索与人工维护。

### 3.2 本模块不负责

1. Profiling 指标计算。
2. 数据质量规则执行。
3. 标准规则生成。
4. 企业级权限中心、多租户、审批流。
5. 复杂实时血缘解析。

说明：
本模块允许缓存部分摘要性信息，例如 `row_count_cached`，但不承担重计算任务。

---

## 4. 设计原则

### 4.1 轻量但可扩展

第一阶段优先实现统一建模、稳定扫描、清晰查询与人工维护能力，避免企业级重平台设计。

### 4.2 自动采集与人工维护分离

扫描结果负责更新技术元数据；人工输入负责补充治理语义。扫描时不得覆盖人工维护字段。

### 4.3 文件与数据库统一抽象

CSV、Excel、Parquet、JSON 也统一映射为 `dataset` 与 `field`，避免前后端与下游模块出现两套模型。

### 4.4 稳定主键

`dataset.id` 与 `field.id` 必须稳定，用于后续 Profiling、标准映射、质量结果、AI 文档等模块关联。

### 4.5 软删除优先

扫描发现对象消失时，只做软删除，不做物理删除，避免下游对象失联。

---

## 5. 支持范围

### 5.1 第一阶段支持的数据源类型

1. PostgreSQL
2. MySQL
3. SQLite
4. CSV
5. Excel
6. Parquet
7. JSON

### 5.2 后续扩展

1. ClickHouse
2. DuckDB
3. API
4. dbt

---

## 6. 子模块拆分

元数据管理模块建议拆分为 4 个子域。

### 6.1 数据源与连接管理

负责：

1. 数据源 CRUD
2. 连接配置校验
3. 测试连接
4. 数据源状态维护

### 6.2 元数据采集与同步

负责：

1. 触发扫描任务
2. 调用不同 connector 采集 schema / 文件结构
3. 标准化采集结果
4. 同步到资产主表
5. 记录扫描日志与统计

### 6.3 资产目录与人工维护

负责：

1. Dataset / Field 列表与详情
2. 描述、展示名、示例值、语义类型维护
3. 标签绑定
4. 主题域归属
5. 搜索与筛选

### 6.4 关系管理

负责：

1. DatasetRelation
2. FieldRelation
3. LineageEdge
4. 基础血缘展示

---

## 7. 核心领域模型

### 7.1 核心对象

1. `DataSource`
2. `Dataset`
3. `Field`
4. `Domain`
5. `Tag`
6. `GlossaryTerm`
7. `FieldGlossaryMapping`
8. `DatasetRelation`
9. `FieldRelation`
10. `LineageEdge`
11. `MetadataScanTask`
12. `MetadataScanLog`

### 7.2 模型关系

1. 一个 `DataSource` 下有多个 `Dataset`。
2. 一个 `Dataset` 下有多个 `Field`。
3. 一个 `Dataset` 可归属一个 `Domain`。
4. `Dataset` 与 `Tag` 为多对多关系。
5. `Field` 与 `Tag` 为多对多关系。
6. `Field` 与 `GlossaryTerm` 为多对多关系，通过 `FieldGlossaryMapping` 承载映射来源与置信度。
7. `DatasetRelation`、`FieldRelation`、`LineageEdge` 用于表达对象间关系。
8. 一个 `DataSource` 下有多条 `MetadataScanTask` 记录。

---

## 8. PostgreSQL 详细表设计

以下为首期建议的核心表设计。主键建议统一使用 UUID，时间字段统一使用 `timestamptz`。

### 8.1 datasource

用途：数据源注册、连接维护、状态管理。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| name | varchar(100) | 数据源名称，唯一 |
| type | varchar(32) | 数据源类型 |
| status | varchar(32) | `draft/active/scan_running/scan_failed/offline` |
| connection_config | jsonb | 连接配置 |
| description | text | 描述 |
| owner | varchar(100) | 负责人 |
| last_scan_at | timestamptz | 最近扫描时间 |
| last_scan_status | varchar(32) | 最近扫描状态 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

索引建议：

1. `uk_datasource_name(name)`
2. `idx_datasource_type(type)`
3. `idx_datasource_status(status)`

### 8.2 domain

用途：主题域组织。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| name | varchar(100) | 名称，唯一 |
| code | varchar(64) | 编码，唯一 |
| parent_id | uuid FK | 父主题域 |
| description | text | 描述 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

### 8.3 dataset

用途：统一承载数据库表、视图、文件对象。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| datasource_id | uuid FK | 所属数据源 |
| source_system_id | varchar(255) | 源端稳定标识 |
| name | varchar(255) | 技术名称 |
| display_name | varchar(255) | 展示名称 |
| schema_name | varchar(255) | schema 名称，文件型可空 |
| dataset_type | varchar(32) | `table/view/file` |
| storage_type | varchar(32) | `database/file` |
| description | text | 人工维护描述 |
| source_comment | text | 源端注释 |
| domain_id | uuid FK | 所属主题域 |
| owner | varchar(100) | 负责人 |
| lifecycle_status | varchar(32) | `active/deprecated` |
| is_deleted | boolean | 软删除标记 |
| row_count_cached | bigint | 缓存行数 |
| field_count_cached | integer | 缓存字段数 |
| last_scanned_at | timestamptz | 最近扫描时间 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

约束与索引建议：

1. 唯一键：`(datasource_id, source_system_id)`
2. 索引：`datasource_id`、`domain_id`、`schema_name`、`name`、`is_deleted`

### 8.4 field

用途：字段级技术属性与治理属性管理。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| dataset_id | uuid FK | 所属数据集 |
| source_column_id | varchar(255) | 源端列标识 |
| name | varchar(255) | 技术名称 |
| display_name | varchar(255) | 展示名称 |
| ordinal_position | integer | 字段顺序 |
| data_type | varchar(128) | 标准化类型 |
| source_data_type | varchar(128) | 源端原始类型 |
| nullable | boolean | 是否可空 |
| is_primary_key | boolean | 是否主键 |
| is_foreign_key | boolean | 是否外键 |
| default_value | text | 默认值 |
| description | text | 人工维护描述 |
| source_comment | text | 源端注释 |
| example_value | text | 示例值 |
| semantic_type | varchar(64) | 语义类型预留 |
| is_deleted | boolean | 软删除标记 |
| last_scanned_at | timestamptz | 最近扫描时间 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

约束与索引建议：

1. 唯一键：`(dataset_id, name)`
2. 索引：`dataset_id`、`name`、`data_type`、`semantic_type`

### 8.5 tag

用途：标签管理。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| name | varchar(100) | 标签名称，唯一 |
| tag_type | varchar(32) | `business/sensitivity/topic` 等 |
| color | varchar(32) | 展示颜色 |
| description | text | 描述 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

### 8.6 dataset_tag

用途：数据集标签映射。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| dataset_id | uuid FK | 数据集 |
| tag_id | uuid FK | 标签 |
| created_at | timestamptz | 创建时间 |

约束建议：

1. 唯一键：`(dataset_id, tag_id)`

### 8.7 field_tag

用途：字段标签映射。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| field_id | uuid FK | 字段 |
| tag_id | uuid FK | 标签 |
| created_at | timestamptz | 创建时间 |

约束建议：

1. 唯一键：`(field_id, tag_id)`

### 8.8 glossary_term

用途：业务术语管理。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| name | varchar(100) | 中文名称，唯一 |
| english_name | varchar(100) | 英文名称 |
| category | varchar(64) | 分类 |
| definition | text | 定义 |
| synonyms | jsonb | 同义词列表 |
| status | varchar(32) | `draft/active` |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

### 8.9 field_glossary_mapping

用途：字段与术语映射。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| field_id | uuid FK | 字段 |
| glossary_term_id | uuid FK | 术语 |
| mapping_type | varchar(32) | `manual/rule/ai` |
| confidence | numeric(5,4) | 置信度 |
| status | varchar(32) | `suggested/confirmed/rejected` |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

约束建议：

1. 唯一键：`(field_id, glossary_term_id)`

### 8.10 dataset_relation

用途：描述数据集间的逻辑关系。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| source_dataset_id | uuid FK | 源数据集 |
| target_dataset_id | uuid FK | 目标数据集 |
| relation_type | varchar(32) | `depends_on/references/same_subject` |
| description | text | 描述 |
| created_at | timestamptz | 创建时间 |

### 8.11 field_relation

用途：描述字段间逻辑关系。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| source_field_id | uuid FK | 源字段 |
| target_field_id | uuid FK | 目标字段 |
| relation_type | varchar(32) | `fk_candidate/same_meaning/derived_from` |
| confidence | numeric(5,4) | 置信度 |
| created_at | timestamptz | 创建时间 |

### 8.12 lineage_edge

用途：数据血缘边。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| source_dataset_id | uuid FK | 上游数据集 |
| target_dataset_id | uuid FK | 下游数据集 |
| relation_type | varchar(32) | 首期建议固定 `lineage` |
| lineage_method | varchar(32) | `manual/sql_parse/imported` |
| confidence | numeric(5,4) | 置信度 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

### 8.13 metadata_scan_task

用途：记录每一次元数据扫描任务。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| datasource_id | uuid FK | 所属数据源 |
| trigger_type | varchar(32) | `manual/scheduled` |
| scan_scope | jsonb | 扫描范围 |
| status | varchar(32) | `pending/running/success/partial_success/failed` |
| total_datasets | integer | 目标对象数 |
| scanned_datasets | integer | 已扫描对象数 |
| new_datasets | integer | 新增对象数 |
| updated_datasets | integer | 更新对象数 |
| deleted_datasets | integer | 删除对象数 |
| error_message | text | 错误摘要 |
| started_at | timestamptz | 开始时间 |
| finished_at | timestamptz | 结束时间 |
| created_at | timestamptz | 创建时间 |

### 8.14 metadata_scan_log

用途：扫描任务执行日志。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid PK | 主键 |
| scan_task_id | uuid FK | 任务 ID |
| log_level | varchar(16) | `info/warn/error` |
| stage | varchar(32) | `connect/discover/sync` 等 |
| message | text | 日志内容 |
| detail_json | jsonb | 扩展信息 |
| created_at | timestamptz | 创建时间 |

---

## 9. 扫描与同步设计

### 9.1 总体流程

1. 用户创建或选择数据源。
2. 用户执行测试连接。
3. 用户触发扫描。
4. 系统创建 `metadata_scan_task`，状态置为 `pending`。
5. 后台任务启动，状态切为 `running`。
6. Connector 读取 schema、表、字段、注释、主键信息或文件结构。
7. 扫描器将结果标准化为统一 payload。
8. 同步服务执行 dataset / field upsert。
9. 同步完成后回写任务统计与日志。
10. 更新数据源最近扫描状态与时间。

### 9.2 同步策略

采用“当前资产表直接 upsert，人工维护字段不覆盖”的方式。

#### dataset 匹配规则

使用 `datasource_id + source_system_id` 匹配，不使用单纯名称匹配。

数据库型数据源建议：

1. `source_system_id = catalog.schema.table`
2. 若无 catalog，则至少为 `schema.table`

文件型数据源建议：

1. `source_system_id = 规范化文件路径`

#### field 匹配规则

使用 `dataset_id + name` 匹配。

#### 自动更新字段

扫描时允许更新：

1. `name`
2. `schema_name`
3. `dataset_type`
4. `storage_type`
5. `source_comment`
6. `last_scanned_at`
7. `source_data_type`
8. `data_type`
9. `nullable`
10. `ordinal_position`
11. `is_primary_key`
12. `is_foreign_key`
13. `default_value`
14. `field_count_cached`

#### 不覆盖的人工维护字段

扫描时不得覆盖：

1. `display_name`
2. `description`
3. `owner`
4. `domain_id`
5. `example_value`
6. `semantic_type`
7. 标签绑定
8. 术语映射
9. 关系配置

### 9.3 删除与恢复策略

1. 扫描中未发现的 dataset / field，不物理删除，置 `is_deleted = true`。
2. 后续扫描再次发现时，恢复为 `is_deleted = false`。
3. 已软删除对象仍允许被历史 Profiling、标准映射、质量结果引用。

### 9.4 改名策略

第一版不强做 rename merge。

规则：

1. 若源端对象名变化导致 `source_system_id` 改变，则按“旧对象软删除 + 新对象创建”处理。
2. 后续若需要自动识别 rename，可基于字段相似度扩展，但不属于 MVP。

---

## 10. 状态机设计

### 10.1 datasource.status

1. `draft`：已创建但尚未验证连接
2. `active`：可正常使用
3. `scan_running`：扫描中
4. `scan_failed`：最近一次扫描失败
5. `offline`：连接不可达

### 10.2 metadata_scan_task.status

1. `pending`
2. `running`
3. `success`
4. `partial_success`
5. `failed`

### 10.3 状态流转规则

1. 任务创建后：`pending`
2. 后台执行开始：`running`
3. 全部成功：`success`
4. 部分对象失败但整体完成：`partial_success`
5. 连接失败或整体异常：`failed`

数据源状态联动规则：

1. 扫描开始时，将 `datasource.status` 更新为 `scan_running`
2. 扫描成功后更新为 `active`
3. 扫描失败后更新为 `scan_failed`
4. 若测试连接失败，则更新为 `offline`

---

## 11. 并发与异常处理

### 11.1 并发控制

1. 同一数据源同一时刻只允许一个运行中的扫描任务。
2. 若已有运行中任务，再次触发扫描时直接返回已有任务信息。
3. 扫描范围通过 `scan_scope` 支持全量与局部重扫。

### 11.2 错误处理

1. 连接失败：任务直接 `failed`，记录在 `connect` 阶段日志。
2. 单个 dataset 读取失败：继续扫描其他对象，任务置 `partial_success`。
3. 单个字段解析失败：跳过异常字段并记录日志。
4. 同步失败：记录 `sync` 阶段错误，并在任务详情页展示摘要与日志。

---

## 12. 后端服务拆分

建议后端分为以下服务层。

### 12.1 datasource_service

负责：

1. 数据源 CRUD
2. 连接配置规范化
3. 连接测试
4. 数据源状态维护

### 12.2 metadata_scan_service

负责：

1. 扫描任务创建
2. 任务状态更新
3. 调用 connector 采集
4. payload 标准化
5. 同步资产主表
6. 扫描统计与日志记录

### 12.3 dataset_service

负责：

1. 数据集列表与详情
2. 搜索与筛选
3. 人工维护字段更新
4. 聚合详情返回

### 12.4 field_service

负责：

1. 字段详情与编辑
2. 字段标签管理
3. 字段术语映射

### 12.5 glossary_service

负责：

1. 术语 CRUD
2. 术语搜索
3. 字段术语映射确认

### 12.6 lineage_service

负责：

1. DatasetRelation 管理
2. FieldRelation 管理
3. LineageEdge 查询与维护

---

## 13. Connector 抽象设计

建议定义统一抽象接口。

### 13.1 BaseConnector

统一方法：

1. `test_connection()`
2. `discover_datasets()`
3. `discover_fields(dataset_ref)`
4. `fetch_comments_and_keys()`

### 13.2 首期 Connector

1. `PostgresConnector`
2. `MySQLConnector`
3. `SQLiteConnector`
4. `CSVConnector`
5. `ExcelConnector`
6. `ParquetConnector`
7. `JSONConnector`

说明：
文件型 connector 也要返回统一的 dataset / field payload，不能额外设计独立模型。

---

## 14. REST API 设计

### 14.1 数据源接口

1. `POST /api/v1/datasources`
2. `GET /api/v1/datasources`
3. `GET /api/v1/datasources/{id}`
4. `PUT /api/v1/datasources/{id}`
5. `DELETE /api/v1/datasources/{id}`
6. `POST /api/v1/datasources/{id}/test-connection`
7. `POST /api/v1/datasources/{id}/scan`
8. `GET /api/v1/datasources/{id}/scan-tasks`

### 14.2 扫描任务接口

1. `GET /api/v1/metadata/scan-tasks/{task_id}`
2. `GET /api/v1/metadata/scan-tasks/{task_id}/logs`

### 14.3 数据集接口

1. `GET /api/v1/metadata/datasets`
2. `GET /api/v1/metadata/datasets/{id}`
3. `PUT /api/v1/metadata/datasets/{id}`
4. `GET /api/v1/metadata/datasets/{id}/fields`
5. `POST /api/v1/metadata/datasets/{id}/tags`
6. `DELETE /api/v1/metadata/datasets/{id}/tags/{tag_id}`
7. `GET /api/v1/metadata/datasets/{id}/lineage`
8. `GET /api/v1/metadata/datasets/{id}/relations`

### 14.4 字段接口

1. `GET /api/v1/metadata/fields/{id}`
2. `PUT /api/v1/metadata/fields/{id}`
3. `POST /api/v1/metadata/fields/{id}/tags`
4. `DELETE /api/v1/metadata/fields/{id}/tags/{tag_id}`
5. `POST /api/v1/metadata/fields/{id}/glossary-mappings`
6. `PUT /api/v1/metadata/field-glossary-mappings/{id}`

### 14.5 术语与标签接口

1. `GET /api/v1/metadata/glossary-terms`
2. `POST /api/v1/metadata/glossary-terms`
3. `PUT /api/v1/metadata/glossary-terms/{id}`
4. `GET /api/v1/metadata/tags`
5. `POST /api/v1/metadata/tags`
6. `PUT /api/v1/metadata/tags/{id}`

### 14.6 搜索接口

1. `GET /api/v1/metadata/search?q=...`

第一版搜索建议采用 PostgreSQL 的关键字搜索与过滤组合，不强依赖 AI 或向量检索。

---

## 15. API 返回视图建议

### 15.1 Dataset 列表页返回

建议包含：

1. 数据集基础信息
2. 数据源名称与类型
3. 字段数
4. 标签摘要
5. 主题域
6. 最近扫描时间
7. 删除标记

### 15.2 Dataset 详情页返回

建议由聚合接口直接返回：

1. 基本信息
2. 来源信息
3. 字段摘要
4. 标签
5. 术语映射摘要
6. 血缘摘要
7. Profiling 摘要占位
8. 质量摘要占位

这样可以减少前端首屏请求瀑布。

---

## 16. 前端页面与信息架构

### 16.1 页面清单

元数据模块建议优先实现以下页面：

1. 数据源列表页
2. 数据源详情页
3. 数据资产目录页
4. 数据集详情页
5. 字段详情抽屉或详情页
6. 术语管理页
7. 血缘关系页（可选增强）

### 16.2 数据源列表页

展示字段建议：

1. 名称
2. 类型
3. 状态
4. 最近扫描时间
5. 最近扫描结果
6. 操作按钮

操作建议：

1. 编辑
2. 测试连接
3. 立即扫描
4. 查看详情

### 16.3 数据源详情页

建议包含：

1. 基本信息
2. 连接配置摘要
3. 最近扫描任务列表
4. 扫描统计卡片
5. 任务日志入口

### 16.4 数据资产目录页

建议布局：

1. 左侧筛选区
2. 右侧列表区

筛选项建议：

1. 数据源
2. schema
3. 主题域
4. 标签
5. 删除状态

主表字段建议：

1. 数据集名称
2. schema
3. 类型
4. 字段数
5. owner
6. 标签
7. 最近扫描时间

### 16.5 数据集详情页

建议采用 Tab 结构：

1. 概览
2. 字段
3. 术语映射
4. 血缘关系
5. 运行摘要

各 Tab 内容建议：

#### 概览

1. 基本属性
2. 描述
3. 标签
4. 主题域
5. 来源信息

#### 字段

1. 字段表格
2. 支持搜索与排序
3. 点击字段弹出详情抽屉

#### 术语映射

1. 显示字段与术语绑定关系
2. 支持确认、拒绝、手工补绑

#### 血缘关系

1. 展示上游与下游节点
2. 首期只读

#### 运行摘要

1. Profiling 摘要卡片
2. 质量结果摘要卡片
3. 当前仅预留展示位

### 16.6 字段详情页 / 抽屉

建议分两块：

1. 技术属性
2. 治理属性

技术属性包括：

1. 类型
2. 可空
3. 主键
4. 默认值
5. 源注释

治理属性包括：

1. 展示名
2. 描述
3. 示例值
4. 语义类型
5. 标签
6. 术语映射

建议首期优先采用抽屉交互，减少页面跳转。

---

## 17. 前后端职责边界

1. 搜索、筛选、分页统一由后端处理。
2. 数据集详情页首屏通过聚合接口返回，降低请求数量。
3. 血缘首期只做只读查询，不做前端复杂编辑器。
4. 标签绑定与术语确认采用轻量弹窗或抽屉，不引入复杂审批流。

---

## 18. 非功能设计

### 18.1 性能要求

1. 数据源列表、数据集列表、字段列表都必须支持分页。
2. 搜索接口默认限制返回数量。
3. 扫描任务异步执行，避免阻塞请求线程。
4. 列表页不得依赖全量加载再前端筛选。

### 18.2 可观测性

1. 每次扫描必须有任务记录。
2. 扫描过程必须记录阶段日志。
3. 前端必须能看到最近任务状态与失败原因。

### 18.3 安全要求

1. `connection_config` 中的敏感信息后续建议加密存储。
2. 接口日志不得打印明文密码。
3. 删除数据源前必须校验关联影响。

### 18.4 可扩展性

1. 预留 `semantic_type`、`source_comment`、`description`、`example_value` 等字段用于标准推荐与 AI 检索。
2. Connector 层必须可插拔，便于后续新增数据源类型。

---

## 19. 与后续模块的衔接

### 19.1 与 Profiling 模块

1. Profiling 任务引用 `dataset.id` 与 `field.id`
2. Profiling 结果可回填 `row_count_cached` 或摘要视图

### 19.2 与数据标准模块

1. 标准推荐依赖字段名、描述、类型、示例值、语义类型
2. 标准映射以 `field.id` 为锚点

### 19.3 与数据质量模块

1. 质量规则绑定到 `dataset.id` / `field.id`
2. 软删除对象仍应保留历史结果关联

### 19.4 与 AI 模块

1. 后续可基于 dataset / field / glossary_term 生成摘要文档
2. 摘要文本可写入 AI 检索表或向量表

---

## 20. MVP 范围

元数据模块第一阶段建议只做以下最小闭环：

1. 数据源管理
2. 测试连接
3. Schema / 文件结构扫描
4. Dataset / Field 目录查询
5. Dataset / Field 人工维护
6. 标签管理
7. 术语管理与字段术语映射
8. Dataset 级基础血缘展示

第一阶段暂不做：

1. 自动 rename merge
2. 高级字段关系发现
3. 实时血缘解析
4. AI 自动解释
5. 复杂权限与审批

---

## 21. 推荐开发顺序

### Phase 1：基础对象与数据源管理

1. 建立数据库表
2. 完成数据源 CRUD
3. 完成测试连接接口

### Phase 2：扫描任务与资产同步

1. 建立扫描任务与日志机制
2. 完成数据库型 connector
3. 完成文件型 connector
4. 完成 dataset / field upsert

### Phase 3：资产目录与详情页

1. 完成数据集列表接口
2. 完成字段列表与详情接口
3. 完成前端目录页与详情页

### Phase 4：标签、主题域、术语

1. 完成标签与主题域管理
2. 完成术语 CRUD
3. 完成字段术语映射

### Phase 5：关系与血缘

1. 完成 dataset relation 与 lineage edge
2. 完成血缘展示页面

---

## 22. 结论

元数据管理模块应作为整个平台的资产底座，采用“连接层 + 资产层 + 语义增强层 + 关系层 + 扫描运行层”的结构，核心设计重点不是单纯采集 schema，而是保证：

1. 自动采集与人工治理并存。
2. 资产对象主键稳定。
3. 扫描同步可追踪、可恢复、可扩展。
4. 下游 Profiling、标准、质量与 AI 能稳定复用该模块输出。

该设计符合当前项目“轻量、可演示、可扩展、后续易接 AI”的目标，也适合作为 MVP 的第一阶段实施基础。
