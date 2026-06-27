

## 项目通用规则

- 修改代码前先阅读相关文件和现有模式。
- 不要改动与任务无关的文件。
- 修改后尽量运行相关测试或说明无法运行的原因。
- 文档、PRD、设计资料主要放在 `prd/`。
- `DQM/` 是之前开发的Data Quality Engine代码，不要修改它，可以借鉴其中的实现。
- `course/` 是之前一门课程中的 NL2SQL Agent 项目代码，可以借鉴其中的实现、风格和架构，但不要修改它。
- `data/` 是存放数据的文件夹。

## 代码主风格参考

- 每个函数都需要类型注释，参数和返回值都要标注类型。
- 每个函数和模块需要详细的doctring注释，说明函数的功能，参数，返回值，示例，用英文描述。
- 代码风格遵循PEP8，使用黑色（black）格式化代码，使用isort管理import顺序。
- 变量命名要有意义，使用snake_case风格，类名使用PascalCase风格。
- loguru进行日志记录
- 避免长代码文件，文件代码过长之后，进行模块化和拆分，方便后续维护
- 避免过度设计，不要引入过于复杂和抽象的设计模式，易于理解和维护
- 采用企业级现代python OOP开发风格和最佳实践，保持代码清晰、简洁、易读。

## `course` 项目代码风格参考

- 后续开发优先参考 `course/data-agent/data-agent` 的后端风格：Python 3.12、FastAPI、async/await、类型标注、轻量分层。
- 后端目录在`backend/`，前端目录在`frontend/`，保持前后端代码完全分离。
- 后端主要代码在`app/`, 分层保持清晰：
    `api/` 放路由、schema 和依赖注入；
    `services/` 负责编排业务流程；
    `agent/` 放 LangGraph 状态、上下文、图和节点；
    `repositories/` 封装 MySQL、Qdrant、Elasticsearch 等存储访问；
    `clients/` 管理外部客户端生命周期；
    `entities/` 使用 dataclass 表达领域对象；
    `models/` 使用 SQLAlchemy ORM 表达数据库表。
- 配置使用 dataclass + OmegaConf 从 `conf/*.yaml` 加载，避免在业务代码里硬编码连接信息、模型参数和日志参数。
- 外部资源通过 manager 初始化和关闭，在 FastAPI lifespan 中统一管理；业务层通过依赖注入或 context 获取 client/repository，不要在函数内部临时创建数据库、向量库、ES、LLM 客户端。
- Repository 保持薄封装，只处理单一存储后端的查询、写入、索引创建和实体转换；跨仓储编排放到 Service 或 Agent 节点中。
- Service 类通过构造函数注入依赖，并将依赖保存为实例属性；对外方法优先使用异步接口。
- Agent 节点保持 `async def node(state, runtime)` 形式，从 `state` 读取输入，从 `runtime.context` 读取依赖，用 `runtime.stream_writer` 输出 `progress`/`result` 事件。
- Agent 节点开始、成功、失败时输出结构化进度事件，例如 `{"type": "progress", "step": "生成SQL", "status": "running"}`；失败时记录中文日志后按场景决定抛出异常或返回错误状态。
- 日志使用 `app.core.log.logger`，日志内容以中文描述业务动作和关键变量；请求链路通过 `request_id_ctx_var` 注入日志上下文。
- Prompt 放在 `prompts/*.prompt`，通过 `load_prompt(name)` 读取；LLM 调用优先使用 LangChain 的 `PromptTemplate | llm | OutputParser` 组合，复杂上下文用 `yaml.dump(..., allow_unicode=True, sort_keys=False)` 序列化。
- 命名保持现有习惯：文件、函数、变量使用 snake_case；类使用 PascalCase；Repository 类按后端命名如 `MetaMySQLRepository`、`ColumnQdrantRepository`；数据对象使用 `*Info`、`*Config`、`*State` 后缀。
- 数据结构优先显式建模：领域数据用 dataclass，API 入参用 Pydantic `BaseModel`，LangGraph state/context 用 `TypedDict`，ORM model 使用 SQLAlchemy `Mapped`/`mapped_column`。
- 批量写入或 embedding 调用使用小批次循环处理，默认 batch size 可参考 10-20，避免一次性提交过大请求。
- 保持代码直接、少抽象：除非能减少实际重复或贴合既有分层，不新增复杂基类、框架封装或通用工具层。

