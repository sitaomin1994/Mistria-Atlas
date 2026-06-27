下面是一份适合直接喂给 **vibe coding / AI 编码工具** 的项目方案说明。你可以把它当成项目 PRD + 技术架构草案 + 开发指令。

---

# 项目名称

**轻量级数据治理平台（元数据管理 + 数据标准管理 + 数据质量管理 + AI 检索增强）**

---

# 一、项目目标

我要开发一个适合个人项目和演示场景的数据治理工具，核心目标不是做企业级重平台，而是做一个 **轻量、可扩展、可接入 AI 的治理工作台**。

系统第一阶段聚焦三个核心能力：

1. **元数据管理**

   * 管理数据源、表、字段、术语、标签、血缘、关系
   * 支持自动扫描 schema 和人工补充说明

2. **数据标准管理**

   * 管理字段标准、格式标准、码表标准、业务标准
   * 支持字段和标准的映射

3. **数据质量管理**

   * 支持 profiling
   * 支持基于标准生成质量规则
   * 支持规则执行、结果存储、问题展示

后续预留 AI 能力：

* 自然语言搜索元数据
* 自动解释数据资产
* 自动推荐字段标准
* 自动概览某个主题域或数据资产

---

# 二、项目定位

这是一个 **面向个人开发者 / 演示 / PoC / 教学** 的轻量治理平台，不做复杂企业功能。

当前不做：

* 多租户
* 复杂审批流
* 企业级权限中心
* 实时流式血缘
* 海量连接器市场

优先做：

* 模块化
* 清晰的数据模型
* 简洁的前后端结构
* 后续容易接 AI

---

# 三、总体技术架构

采用 **模块化单体 + 可扩展 AI 服务层** 的架构。

## 1. 技术栈建议

### 后端

* Python
* FastAPI
* SQLAlchemy
* Alembic
* Pydantic

### 主数据库

* PostgreSQL

### 文件分析 / 本地数据 profiling

* DuckDB

### 辅助数据处理

* pandas（仅辅助，不作为主计算引擎）

### 异步任务

* Celery + Redis
  或先简化为 FastAPI BackgroundTasks，后期再升级

### 前端

* Next.js / React
* TypeScript
* Ant Design 或 shadcn/ui

### AI 预留

* PostgreSQL + pgvector
* 独立 AI service（后续）

---

# 四、核心架构原则

## 1. 主库用关系型数据库

系统核心数据全部存 PostgreSQL：

* 元数据
* 标准
* 规则
* profiling 结果
* 质量结果
* 映射关系
* 日志

原因：

* 结构清晰
* 好做管理系统
* 容易扩展
* 后续可加 pgvector

## 2. Profiling 计算尽量下推

对数据库型数据源：

* 优先使用 SQL 在源数据库内计算基础 profiling 指标
* 不要把大量数据整表拉到 Python 里再算

对文件型数据源：

* 优先使用 DuckDB 计算
* pandas 只做补充分析

## 3. Python 做编排，不做主引擎

Python 负责：

* 任务调度
* 规则编排
* SQL 生成
* 结果汇总
* 标准推荐
* AI 预留接口

## 4. AI 不直接嵌在主系统逻辑里

预留独立 AI 层：

* 元数据主库
* 向量检索层
* AI 服务层

---

# 五、系统模块设计

---

## 模块 1：元数据管理模块

### 功能

1. 数据源管理
2. Schema 扫描
3. 数据资产目录
4. 字段字典
5. 标签分类
6. 业务术语
7. 术语映射
8. 血缘管理
9. 元数据搜索
10. 元数据人工维护

### 支持的数据源

第一阶段：

* PostgreSQL
* MySQL
* SQLite
* CSV
* Excel
* Parquet
* JSON

后续：

* API
* ClickHouse
* DuckDB
* dbt

### 元数据对象

* DataSource
* Dataset
* Field
* Tag
* Domain
* GlossaryTerm
* LineageEdge
* FieldRelation
* DatasetRelation

---

## 模块 2：数据标准管理模块

### 功能

1. 标准分类管理
2. 标准定义管理
3. 标准版本管理
4. 标准码表管理
5. 标准适用范围管理
6. 字段与标准映射
7. 标准推荐映射

### 标准类型

1. 基础格式标准

   * 手机号
   * 邮箱
   * 日期
   * 金额
   * 邮编

2. 基础约束标准

   * 非空
   * 唯一
   * 长度限制
   * 数值范围

3. 码表标准

   * 性别
   * 国家
   * 地区
   * 币种
   * 状态码

4. 业务字段标准

   * 客户编号
   * 订单号
   * 商品编码
   * 患者编号
   * 诊断编码

5. 业务口径标准

   * 后续再做

### 核心能力

* 标准可以转化为可执行规则
* 字段可以绑定一个或多个标准
* 支持人工映射和推荐映射

---

## 模块 3：数据 Profiling 模块

### 功能

对每张表、每个字段生成基础 profiling 结果。

### 基础 profiling 指标

1. 表级

* 行数
* 列数

2. 字段级

* 字段类型
* 是否可空
* 空值数
* 空值率
* distinct_count
* unique_ratio
* min
* max
* avg
* length_min
* length_max
* topN
* 示例值
* 基础分布摘要

### 计算策略

#### 对数据库

优先用 SQL 直接计算：

* count
* null_count
* distinct_count
* min/max
* avg
* topN
* length stats

#### 对文件

使用 DuckDB

#### 对特殊逻辑

使用 Python 补充

---

## 模块 4：数据质量管理模块

### 功能

1. 规则模板管理
2. 规则实例管理
3. 规则执行任务
4. 质量结果管理
5. 质量问题列表
6. 简单质量评分

### 规则来源

1. 手工定义
2. 从标准自动生成
3. 系统内置规则模板

### 初期支持的规则类型

1. 非空校验
2. 唯一校验
3. 格式校验
4. 枚举/码表校验
5. 数值范围校验
6. 长度校验
7. 简单跨字段逻辑校验

### 后期再做

* 跨表一致性
* 业务准确性
* 高级异常检测

---

## 模块 5：AI 预留模块

第一阶段只预留，不要求全部实现。

### 未来能力

1. 自然语言搜索元数据
2. “这张表是干嘛的”自动解释
3. 字段标准推荐
4. 主题域概览总结
5. 质量问题总结
6. 规则解释

### 预留设计

* 每个 dataset / field / standard / glossary term 生成文本摘要
* 后续写入 pgvector
* 后续增加 AI service 调用 LLM

---

# 六、核心数据流设计

---

## 流程 1：元数据采集流程

1. 用户注册数据源
2. 系统连接数据源
3. 自动扫描 schema
4. 生成 dataset / field 元数据
5. 用户补充描述、标签、术语
6. 存入 PostgreSQL

---

## 流程 2：Profiling 流程

1. 用户选择数据源 / 表
2. 系统生成 profiling 任务
3. 如果是数据库表：

   * 动态生成 SQL
   * 在源数据库执行统计
4. 如果是文件：

   * 使用 DuckDB 计算
5. 将 profiling 结果写入 PostgreSQL
6. 前端展示 profiling 结果

---

## 流程 3：标准映射流程

1. 用户创建标准
2. 系统读取字段信息
3. 基于字段名、类型、样本值推荐标准
4. 用户确认映射
5. 存储 field_standard_mapping

---

## 流程 4：质量检测流程

1. 用户选择表 / 字段
2. 系统读取字段绑定的标准
3. 标准转成规则实例
4. 生成质量检测任务
5. 执行规则
6. 保存检测结果
7. 展示问题列表和统计结果

---

## 流程 5：AI 检索预留流程

1. 用户输入自然语言问题
2. 系统在 metadata summary / standard summary 中检索
3. 取回相关对象上下文
4. 交给模型总结
5. 返回带依据的回答

---

# 七、数据库设计建议

以下是建议的核心表。

---

## 1. 元数据相关表

### datasource

* id
* name
* type
* connection_config
* description
* status
* created_at
* updated_at
* last_scan_at

### dataset

* id
* datasource_id
* name
* display_name
* schema_name
* type
* description
* domain_id
* owner
* row_count_cached
* created_at
* updated_at

### field

* id
* dataset_id
* name
* display_name
* data_type
* nullable
* is_primary_key
* is_foreign_key
* description
* example_value
* created_at
* updated_at

### tag

* id
* name
* type
* description

### dataset_tag

* id
* dataset_id
* tag_id

### field_tag

* id
* field_id
* tag_id

### glossary_term

* id
* name
* english_name
* definition
* category
* synonyms
* created_at
* updated_at

### field_glossary_mapping

* id
* field_id
* glossary_term_id
* confidence
* mapping_type

### lineage_edge

* id
* source_dataset_id
* target_dataset_id
* relation_type
* created_at

---

## 2. 标准相关表

### standard_category

* id
* name
* parent_id
* description

### standard_definition

* id
* name
* code
* category_id
* description
* standard_type
* version
* status
* created_at
* updated_at

### standard_rule_template

* id
* standard_definition_id
* rule_type
* rule_config_json
* description

### standard_code_set

* id
* standard_definition_id
* code
* label
* sort_order
* is_active

### field_standard_mapping

* id
* field_id
* standard_definition_id
* mapping_source
* confidence
* status
* created_at
* updated_at

---

## 3. Profiling 相关表

### profile_task

* id
* datasource_id
* dataset_id
* task_status
* trigger_type
* started_at
* finished_at

### dataset_profile_result

* id
* profile_task_id
* dataset_id
* row_count
* column_count
* profiled_at

### field_profile_result

* id
* profile_task_id
* field_id
* inferred_type
* null_count
* null_ratio
* distinct_count
* unique_ratio
* min_value
* max_value
* avg_value
* length_min
* length_max
* top_values_json
* sample_values_json
* profiled_at

---

## 4. 数据质量相关表

### quality_rule_template

* id
* name
* rule_type
* description
* default_config_json

### quality_rule_instance

* id
* field_id
* standard_definition_id
* rule_template_id
* config_json
* enabled
* created_at

### quality_check_task

* id
* datasource_id
* dataset_id
* task_status
* started_at
* finished_at

### quality_check_result

* id
* quality_check_task_id
* rule_instance_id
* dataset_id
* field_id
* check_status
* total_count
* failed_count
* pass_ratio
* result_json
* checked_at

### quality_issue

* id
* quality_check_result_id
* dataset_id
* field_id
* issue_type
* issue_level
* issue_summary
* issue_detail_json
* created_at

---

## 5. AI 预留表

### ai_document

* id
* object_type
* object_id
* title
* content
* content_summary
* updated_at

后续可增加 embedding 列或单独 pgvector 表。

---

# 八、后端分层设计

建议采用如下目录结构：

```text
backend/
  app/
    api/
      v1/
        datasource.py
        metadata.py
        glossary.py
        standards.py
        profiling.py
        quality.py
        ai.py
    core/
      config.py
      database.py
      security.py
    models/
      datasource.py
      dataset.py
      field.py
      glossary.py
      standard.py
      profiling.py
      quality.py
    schemas/
      datasource.py
      dataset.py
      field.py
      glossary.py
      standard.py
      profiling.py
      quality.py
    services/
      datasource_service.py
      metadata_scan_service.py
      profiling_service.py
      standard_mapping_service.py
      quality_rule_service.py
      quality_check_service.py
      ai_context_service.py
    repositories/
      datasource_repo.py
      dataset_repo.py
      field_repo.py
      standard_repo.py
      profiling_repo.py
      quality_repo.py
    tasks/
      profiling_tasks.py
      quality_tasks.py
    utils/
      sql_builder.py
      type_inference.py
      sampling.py
      logger.py
    main.py
```

---

# 九、前端页面设计

建议前端菜单如下：

1. 数据源管理
2. 数据资产目录
3. 术语管理
4. 数据标准
5. 标准映射
6. Profiling 分析
7. 数据质量检测
8. 血缘关系
9. 搜索中心
10. 系统设置

---

## 关键页面

### 1. 数据源列表页

* 数据源名称
* 类型
* 状态
* 最近扫描时间

### 2. 数据集详情页

* 基本信息
* 字段列表
* 标签
* 术语映射
* 血缘关系
* Profiling 摘要
* 质量检测摘要

### 3. 字段详情页

* 数据类型
* 描述
* 示例值
* profiling 指标
* 绑定标准
* 规则结果

### 4. 标准列表页

* 标准名称
* 类型
* 版本
* 适用范围

### 5. 标准映射页

* 字段
* 推荐标准
* 已绑定标准
* 置信度
* 确认状态

### 6. Profiling 结果页

* 表级统计
* 字段级统计
* topN
* 空值率
* 分布摘要

### 7. 数据质量页

* 规则列表
* 检测结果
* 失败数
* 问题详情

---

# 十、Profiling 实现策略

采用 **分层执行策略**。

## 1. 数据库型数据源

优先 SQL 下推。

### 典型 SQL 统计

* `count(*)`
* `count(col)`
* `count(distinct col)`
* `min(col)`
* `max(col)`
* `avg(col)`
* `length(col)`
* `group by col order by count(*) desc limit N`

## 2. 文件型数据源

使用 DuckDB：

* 读取 CSV / Parquet / JSON
* 做 SQL 风格分析

## 3. Python 负责

* 任务编排
* SQL 生成
* 特殊逻辑
* 汇总入库

---

# 十一、标准与质量的关系设计

系统要明确三层关系：

## 1. 标准定义层

定义“应该是什么样”
例如：

* 手机号必须 11 位
* 性别必须属于标准码表
* 客户编号必须唯一

## 2. 字段映射层

定义“哪个字段用哪个标准”
例如：

* `customer.phone` -> 手机号标准
* `patient.gender_code` -> 性别码标准

## 3. 规则执行层

定义“怎么检查”
例如：

* 正则校验
* 枚举值校验
* 唯一性校验

---

# 十二、字段与标准匹配策略

采用三种模式：

## 1. 人工映射

第一版必须支持。

## 2. 规则推荐

根据以下信息推荐：

* 字段名
* 字段描述
* 数据类型
* 样本值
* distinct_count
* top_values

## 3. AI 辅助推荐

后续增强，不作为第一阶段依赖。

---

# 十三、开发阶段建议

---

## Phase 1：元数据 MVP

目标：先让系统能看和管元数据

实现：

1. 数据源管理
2. Schema 扫描
3. 数据资产目录
4. 字段字典
5. 标签分类
6. 术语管理
7. 血缘关系基础展示

---

## Phase 2：Profiling

目标：让系统能自动分析数据

实现：

1. Profiling 任务
2. 表级和字段级 profiling
3. Profiling 结果展示
4. DuckDB 文件支持
5. SQL 下推数据库分析

---

## Phase 3：数据标准管理

目标：定义规范

实现：

1. 标准分类
2. 标准定义
3. 标准版本
4. 码表管理
5. 字段-标准映射

---

## Phase 4：数据质量管理

目标：把标准转成检查能力

实现：

1. 规则模板
2. 规则实例
3. 质量检测任务
4. 结果展示
5. 问题列表

---

## Phase 5：AI 增强

目标：让系统更智能

实现：

1. ai_document 生成
2. pgvector 接入
3. 自然语言搜索
4. 表/字段自动解释
5. 标准推荐说明

---

# 十四、给 vibe coding 的开发指令模板

你可以把下面这段直接喂给 AI 编码工具。

---

## 开发任务描述

请帮我实现一个轻量级数据治理平台，后端使用 Python + FastAPI + SQLAlchemy + PostgreSQL，前端使用 React / Next.js。系统采用模块化单体架构，包含以下模块：

1. 元数据管理
2. 数据标准管理
3. Profiling
4. 数据质量管理
5. AI 预留接口

### 后端要求

* 使用 FastAPI 提供 REST API
* 使用 SQLAlchemy 建模
* 使用 Alembic 管理数据库迁移
* 使用 PostgreSQL 作为主数据库
* 支持 CSV / Excel / Parquet / PostgreSQL / MySQL / SQLite 数据源
* Profiling 对数据库型数据源优先通过 SQL 下推计算
* Profiling 对文件型数据源优先通过 DuckDB 计算
* pandas 仅作为辅助处理工具
* 所有 profiling 和质量结果统一写回 PostgreSQL

### 核心功能

#### 元数据管理

* 数据源注册、编辑、删除
* schema 扫描
* dataset 和 field 管理
* 标签管理
* 术语管理
* 字段和术语映射
* dataset 血缘关系管理

#### 数据标准管理

* 标准分类管理
* 标准定义管理
* 标准版本管理
* 码表管理
* 字段与标准映射

#### Profiling

* 创建 profiling 任务
* 生成表级和字段级统计
* 支持 null_count, null_ratio, distinct_count, unique_ratio, min, max, avg, length_min, length_max, top_values

#### 数据质量

* 规则模板管理
* 规则实例管理
* 质量检测任务
* 支持非空、唯一、格式、长度、枚举、范围校验
* 结果和问题列表展示

#### AI 预留

* 为 dataset / field / standard / glossary term 生成 ai_document
* 预留后续接 pgvector 和大模型的接口

### 数据库表

请优先实现以下数据表：

* datasource
* dataset
* field
* tag
* dataset_tag
* field_tag
* glossary_term
* field_glossary_mapping
* lineage_edge
* standard_category
* standard_definition
* standard_rule_template
* standard_code_set
* field_standard_mapping
* profile_task
* dataset_profile_result
* field_profile_result
* quality_rule_template
* quality_rule_instance
* quality_check_task
* quality_check_result
* quality_issue
* ai_document

### 工程结构

请按以下目录结构组织后端代码：

* api
* models
* schemas
* services
* repositories
* tasks
* utils

### 目标

先完成可运行的 MVP，优先保证：

1. 元数据可采集
2. Profiling 可运行
3. 标准可配置
4. 质量规则可执行
5. 架构便于后续接 AI

---

# 十五、最终一句话方案

这个项目最适合的整体方案是：

**用 PostgreSQL 做统一治理主库，用 SQL 下推 + DuckDB 做 profiling 计算，用 Python/FastAPI 做任务编排和规则执行，用模块化单体承载元数据、标准和质量三大模块，并预留 pgvector + AI service 作为后续智能化扩展。**

如果你要，我下一步可以直接继续给你一版 **“后端数据库建表 SQL + FastAPI 接口清单”**。
