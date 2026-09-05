# 校园智享生活平台

Campus AI Life Platform 是一个面向高校学生的“校园餐饮 + AI 助手”后端项目。它以 Sky Take Out 的完整餐饮链路为业务底座，融合 Spring AI 的多轮对话、RAG、Tool Calling、ReAct Agent 与 SSE；AI 通过受控 Service 工具读取真实菜品和本人订单，而不是成为与业务割裂的聊天演示。

## 核心功能

- 学生端：登录、菜品/套餐浏览、购物车、地址、下单、支付流程、订单查询/催单/取消。
- 管理端：员工、分类、菜品、套餐、订单、营业状态、数据统计。
- AI 助手：普通/Agent 对话、短期记忆、MySQL 历史、SSE 流式事件。
- RAG：校园 FAQ Markdown 加载、切分、Embedding、PGVector 入库与相似度检索。
- 工具：餐食搜索、菜品推荐、本人订单查询、网页搜索/抓取、资源下载、PDF 计划生成。
- 安全：JWT 数据隔离、工具白名单和参数边界、SSRF/目录穿越防护、Agent 步数上限、密钥环境变量化。

## 技术栈

Java 21、Spring Boot 3.4.4、Spring MVC、MyBatis、MySQL、Redis、JWT、Spring AI 1.0、PostgreSQL/PGVector、Maven、SSE、PDFBox、Jsoup、Springdoc OpenAPI。

## 架构与模块

```text
HTTP -> JWT Interceptor -> Controller -> Service -> Mapper -> MySQL
                              |
                              +-> CampusAiChatService -> ChatModel
                                      |-> ChatMemory + MySQL history
                                      |-> RAG Advisor -> PGVector
                                      +-> Tool registry -> Dish/Order Service
```

- `sky-common`：统一返回、异常、JWT、上下文与通用工具。
- `sky-pojo`：业务/AI 的 DTO、Entity、VO。
- `sky-server`：启动模块、MVC、业务 Service/Mapper，以及 `com.sky.ai` AI 子域。
- `sql/campus_ai.sql`：AI 会话表增量脚本。
- `docs`：架构、数据库与面试说明。

选择在 `sky-server` 内新增 AI 子域，是为了直接复用受事务和权限约束的业务 Service，避免 AI 模块反向依赖启动模块或直接访问数据库。详细决策见 `docs/architecture-analysis.md`。

## 关键流程

外卖下单：JWT 用户 → 校验地址和购物车 → 从数据库重新读取在售菜品/套餐价格 → 后端汇总金额 → 同一事务写订单及明细 → 清空购物车。客户端提交的金额不作为结算依据。

AI Chat：JWT 用户 → 校验会话归属 → 从 MySQL 恢复历史到窗口 ChatMemory → 组合身份提示词/RAG/工具 → 调用模型 → 保存用户和助手消息 → 返回 JSON 或 SSE 事件。

RAG：classpath Markdown → 段落切分 → Embedding → PGVector；查询时执行相似度检索，将少量相关片段作为上下文交给模型。向量索引可由原始文档重建。

Tool Calling：模型只能从固定工具白名单选择。`FoodSearchTool` 调用 `DishService` 的白名单检索；`OrderQueryTool` 在服务端绑定已认证用户 ID，模型没有传入其他用户 ID 的入口。

ReAct Agent：模型输出 JSON 的 Reason/Action → 校验工具名 → 执行并记录 Observation → 继续推理或输出 Final Answer。最大步数和单工具重试次数均受配置限制。

## 数据库

- MySQL：用户、菜品、购物车、订单、`ai_conversation`、`ai_message`。
- Redis：营业状态及可扩展缓存；不作为业务事实源。
- PostgreSQL + PGVector：文档向量索引，仅在启用 RAG 时连接。

先导入 `demo/mysql.sql`，再导入 `sql/campus_ai.sql`。PGVector 初始化方式见 `docs/database.md`。

## 本地启动

前置：JDK 21、MySQL 8、Redis 6+；仅启用 RAG 时需要 PostgreSQL 15+ 和 vector 扩展。

```powershell
Copy-Item sky-server/src/main/resources/application-local.example.yml `
  sky-server/src/main/resources/application-local.yml
./mvnw clean test
./mvnw -pl sky-server -am spring-boot:run -Dspring-boot.run.profiles=local
```

不配置模型、搜索或 PGVector 时，餐饮主系统仍可启动；AI Chat 返回友好的未配置提示。生产环境必须替换示例 JWT secret，且不要提交 `application-local.yml`。

常用环境变量：

| 变量 | 用途 | 默认行为 |
|---|---|---|
| `MYSQL_URL/USERNAME/PASSWORD` | MySQL | 本机 `sky_take_out` |
| `REDIS_HOST/PORT/PASSWORD` | Redis | `127.0.0.1:6379` |
| `JWT_ADMIN_SECRET/JWT_USER_SECRET` | JWT 签名 | 仅开发占位值 |
| `AI_CHAT_MODEL` | `openai` 启用兼容模型 | `none` |
| `AI_EMBEDDING_MODEL` | `openai` 启用 Embedding | `none` |
| `AI_API_KEY/AI_BASE_URL/AI_MODEL` | 模型服务 | 无 Key 时关闭 |
| `AI_RAG_ENABLED` | 启用 PGVector RAG | `false` |
| `PGVECTOR_URL/USERNAME/PASSWORD/DIMENSIONS` | 向量库 | 仅 RAG 启用时使用 |
| `SEARCH_API_KEY` | 网页搜索 | 缺失时工具友好降级 |
| `AI_AGENT_MAX_STEPS/AI_TOOL_RETRIES` | Agent 边界 | `8/1` |
| `AI_OUTPUT_DIR` | PDF/下载输出根目录 | `./data/ai-output` |

完整样例在 `application-local.example.yml`。

## API 示例

所有 `/user/ai/**` 请求都必须带用户 JWT 请求头（默认 `authentication`）。

```http
POST /user/ai/chat
Content-Type: application/json
authentication: <JWT>

{"conversationId":null,"message":"我只有 20 元，推荐一点辣的","agent":false}
```

```http
GET /user/ai/chat/stream?message=帮我安排周六约会，预算200元&agent=true
Accept: text/event-stream
authentication: <JWT>
```

SSE 会发送 `conversation`、`message`、`done` 或 `error` 事件。其他入口：`GET /user/ai/conversations`、`GET /user/ai/conversations/{id}/messages`、`POST /admin/ai/knowledge/init`。Swagger UI：`/swagger-ui.html`。

## 面试亮点

1. 用业务 Service 包装模型工具：AI 能用真实数据，但没有任意 SQL 和越权查询能力。
2. 双层记忆：ChatMemory 控制 Prompt 窗口，MySQL 提供长期历史、归属校验和审计。
3. 业务一致性与 AI 安全并重：订单后端重算且事务落库；Agent 同时具有白名单、重试、步数、SSRF 和文件边界。

更具体的代码级回答见 `docs/interview-guide.md`。

## 参考与许可

业务结构参考 `Sonder-MX/sky-take-out`，AI 设计参考 `liyupi/yu-ai-agent`。二者的迁移取舍记录在架构分析中；使用和分发前请同时核对上游仓库的许可证与素材授权。
