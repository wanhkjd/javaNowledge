# PROJECT_CONTEXT

## 项目范围与根目录
- 项目根目录：`C:\Users\35998\Desktop\job\project\wanjoyMall`，由根 `pom.xml` 的 `wanjoyMall` 聚合项目确认。Git 顶层位于 `C:\Users\35998\Desktop\job\project`，不将其直接视为商城项目根目录。
- 商城包含 Java 微服务、Vue 前端及独立 Python 客服服务 `custcare-agent-py`。
- 根 Maven 配置已确认：Java 21、Spring Boot 3.5.16；模块包含公共组件、商品、购物车、用户、交易、支付、网关和 Java 客服。
- 本次阅读范围：Python 的 15 个 app 源文件及根 main.py/run.py；核对前端 API/代理、Java 网关/身份拦截器、商品/交易相关接口以及部署路由。不是整个商城逐文件审计。

## 用户需求与工作约束
- 用户为零基础，需要白话讲解 Python 客服微服务、请求调用链、贴合真实源码的面试题及自然口述答案。
- 区分“源码存在”“隔离验证”“真实集成未验证”和“改进建议”；不编造上线、性能、退款或开发经历。
- 新增项目文档统一保存在根 ai-docs，维护本文件及 EXECUTION_LOG.md。
- 本次只新增/更新阅读文档，不改业务源码、不调用付费模型、不连接真实 MySQL/Qdrant/商城服务、不执行真实订单/退款。
- 不在文档或回复中复制 .env/.env.example 密钥。现有用户修改和原有测试产物不清理。

## Python 服务目录与技术栈
- 服务目录：`C:\Users\35998\Desktop\job\project\wanjoyMall\custcare-agent-py`。
- Python >=3.11；本次隔离验证使用已有 .venv，解释器版本 3.11.14。
- FastAPI/Uvicorn：HTTP 接口及 ASGI 运行；LangGraph：状态与固定条件分流；LangChain 的 OpenAI 兼容客户端：模型和 Embedding。
- SQLAlchemy/PyMySQL：同步 MySQL ORM；Qdrant：命名 dense/sparse 向量检索；httpx：异步调用 Java HTTP 接口。
- 开发入口：run.py 加载 app.main:app，默认监听 0.0.0.0:8088，reload=True。根 main.py 只是 Hello 模板，不启动客服。
- app/main.py：应用、跨域及路由；app/api/v1/chat.py：回答/SSE/评价；app/api/v1/kb.py：知识查询/同步。
- app/core/config.py、database.py：配置及数据库连接；app/models/orm.py：5 张表映射。
- app/graph/state.py、builder.py、nodes/*：状态、建图、路由/规则/订单/导购节点。
- app/rag/embeddings.py、hybrid_retriever.py：稠密/稀疏表示及融合检索；app/tools/mall_tools.py：商品/订单查询。
- pyproject.toml/uv.lock 声明与锁定依赖；.python-version 指向3.11。Python README 当前为空，data 未发现文件。

## 配置与运行前提（真实运行未验证）
- 配置优先级：环境变量 > 当前工作目录 .env > 字段默认值。
- 默认聊天模型 qwen-plus、Embedding text-embedding-v3/1024维；模型客户端名称为 ChatOpenAI/OpenAIEmbeddings，但配置 base URL 指向阿里百炼兼容服务，不等于使用 OpenAI 官方模型。
- 默认 MySQL 库 wm_agent_care、Qdrant 集合 mewhelp_kb、TOP_K=3、CONFIDENCE_THRESHOLD=0.65。
- 必须先核对依赖、模型凭据、MySQL库表、Qdrant已存在集合及 dense/sparse/维度、Java服务可达与身份协议。源码未见 create_all、迁移或 create_collection；未发现这套 Python 库表/集合的专用初始化定义。
- 在前提完成后，可从服务目录使用已有 .venv Python 运行 run.py；uv sync/uv run 是另一个依赖及运行方案。本次未安装依赖或启动服务。
- GET / 只返回 healthy 和配置摘要，没有探测模型/数据库/向量库/商城依赖，不代表业务就绪。

## 接口与架构（源码已确认）
- GET /custcare/answer：message 查询参数，返回完整 JSON 答案；只保存助手消息及耗时，不返回 message_id。
- POST /custcare/chat/stream：message、可选 session_id/selected_order_id；SSE 先发 session，等待 ainvoke 完成，再发 intent/action/每4字符 message/end。不是模型边生成边输出；该接口不保存 ChatMessage。
- POST /custcare/feedback：按 message_id 更新反馈，点踩写待审核池；未校验消息归属，feedback 未限定±1。返回字典 code=404 不等于 HTTP 404。
- GET /custcare/kb/chunks：全部知识（含停用），无分页/本地管理员鉴权。
- POST /custcare/kb/sync-all：启用切片逐条向量化/upsert，再提交 MySQL 同步标记；单条失败继续，未清理停用或删除知识的旧向量，无本地管理员鉴权。
- 图结构：START -> router -> 一个业务分支 -> END。faq -> rag；order_query/refund_workflow -> order；product_search/chitchat -> agent。
- 模型主要做问题改写、意图识别、表达生成；业务函数由代码固定调用。未见 bind_tools/ToolNode/ReAct工具循环，不将其描述为多自主Agent系统。
- router 要求JSON，异常转FAQ，但未做意图枚举/schema验证；客户端构造位于try外。它可能以模型order_id=null覆盖前端显式selected_order_id。

## 业务能力与持久化边界
- RAG：MySQL已存在知识切片 -> dense/sparse及正文来源payload写Qdrant；问答时两路各取TOP_K*2（默认6），串行调用query_points，手工RRF取3条。
- 稀疏表示是中文单字/英文数字片段的归一化词频，没有IDF/BM25。索引abs(hash(word))%1000000存在默认跨进程随机化与碰撞风险；更换算法需要重建索引。
- RRF在本项目写死k=60，融合分用于排名；score保留稠密原分，仅稀疏命中固定0.5。confidence_score取融合第一条score，非最大稠密分、非正确概率。未发现证明0.65最优的标注评测。
- 低分/无召回返回固定兜底并尝试记录FlywheelPool；高分拼接资料生成并附来源。检索异常未统一兜底，提示词不能保证零幻觉。
- 商品：调用 /search/list，过滤status=1，金额分转元。工具支持品牌/价格参数，当前节点仅传整句关键词及limit，没有结构化预算/品牌提取。
- 订单：user_id>0后调用 /orders/mine，3秒httpx超时，列最近3单；售后选单取5单。金额分转元、订单号字符串化、状态码转中文。未实现物流轨迹或指定订单详情。
- 售后无订单号：输出select_order action后END，期待客户端再次请求，不是interrupt/resume；现有前端未接入此action。
- 售后有订单号：只写本地REFUND/PROCESSING工单，编号TK+8位UUID前缀；未真实退款、未改订单/余额、未校验订单归属/可退状态、没有业务幂等。32bit随机编号不保证无碰撞。固定退款时间/进度话术超出实现。
- 5个ORM：KbChunk、ChatSession、ChatMessage、FlywheelPool、AftersalesTicket；ChatSession定义未使用，历史消息未从数据库加载。
- messages为普通List，没有add_messages reducer；每轮只传HumanMessage并覆盖旧历史，节点不追加AIMessage。MemorySaver仅进程内，重启丢、多worker不共享，客户端session_id无归属校验。
- 数据飞轮只自动收集低分/点踩待审核样本，没有运营审核、修正发布、评测回归或模型训练闭环。点踩不保存完整原始问答对，现有回答接口未返回message_id。
- MySQL与Qdrant不在同一事务，存在同步失败/旧向量残留。连接池10+20是每进程每引擎设置，不是数据库全局最多30连接。
- 同步SQLAlchemy/Embedding/Qdrant被直接调用于async节点，会阻塞事件循环；async不自动把普通函数放线程池。部分分支把依赖错误混成无商品/无订单。

## 与现有 Java/前端接入差异（重要）
- Java custcare-service默认8087、Python默认8088，是两套不同实现。Java既有FAQ/敏感词/工具调用/会话记忆不能当成Python能力。
- 仓库前端仍GET /custcare/answer，/api普通代理默认指向网关8080；网关客服路由URI为lb://custcare-service，当前只匹配/custcare/answer。Python未见Nacos注册或默认网关接入。
- 以上仅为仓库配置；远程Nacos和真实进程未检查。不能说现有前端已经在用Python，也不能断定远程配置与模板一致。
- Python读取/发送X-User-Id；Java网关JWT校验后注入user-info，公共拦截器从user-info写UserContext。只发X-User-Id不能按现有Java代码建立本人订单上下文。
- Python自身不校验JWT，user_id>0不等于可信身份；对齐头名之外还需可信网关/服务间认证及禁止外部绕过。CORS不是鉴权。

## 已确认问题与建议（未实施）
1. 高优先级：身份/管理员/会话/反馈/订单归属鉴权、内部端口保护；样例含非空形似key，真实性未验证，若真实应由用户轮换和检查历史。
2. 修复选单覆盖、消息追加与持久化、多实例会话隔离；收敛售后话术，补业务校验和真实流程后再承诺退款。
3. 稀疏索引稳定化、知识停用删除/同步状态/重试/版本；MySQL-Qdrant一致性。
4. 原生异步或受控线程池、HTTP连接复用、明确超时/异常、真实模型流式、断连处理和消息落库。
5. 建表/集合初始化、部署说明、Python自动化测试、标注评测、监控与安全测试。

## 本次验证与资料
- 任务CUSTCARE-READ-20261102-001完成17源码AST/路由/状态合并/选单/ID/词频/跨seed/RRF/SSE/源码哈希10项隔离检查，均得到预期结果。其中多项确认了上述缺陷，不代表业务功能验收通过。
- 本次未执行真实模型/MySQL/Qdrant/商城集成、负载、安全及部署测试；既有Java文档历史测试未重跑。
- 完整教学与30道口述题：`C:\Users\35998\Desktop\job\project\wanjoyMall\ai-docs\CUSTCARE_AGENT_BEGINNER_GUIDE.md`。
- 具体验证证据：`C:\Users\35998\Desktop\job\project\wanjoyMall\ai-docs\CUSTCARE_READ_VALIDATION.md`；执行及清理详情见同目录EXECUTION_LOG.md。
- 2026-11-02首次建立项目上下文与执行记录；此前根及Python子目录未发现上述维护文档，无可核实的历史执行记录。当前仓库已有大量用户修改/未跟踪文件，本次保留原状。