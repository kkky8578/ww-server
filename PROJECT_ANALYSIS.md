# ww-server 项目分析报告

生成日期：2026-06-24  
分析范围：`src` 主 NestJS 服务、`mcp` 子项目、运行配置、监控配置与测试文件。

## 1. 项目定位

`ww-server` 是一个面向求职/面试场景的后端服务，核心能力集中在“用户账户 + AI 简历押题 + AI 模拟面试 + 支付充值 + 微信扫码登录 + Prometheus 监控”。代码中多处使用 `wwzhidao`、`面试汪` 等命名，整体可以理解为一个 AI 面试 SaaS 后端。

项目由两部分组成：

| 部分 | 位置 | 作用 |
| --- | --- | --- |
| 主服务 | `src/` | NestJS HTTP API，提供用户、微信、支付、AI 面试、监控等能力 |
| MCP 子项目 | `mcp/` | 基于 `@modelcontextprotocol/sdk` 的 stdio MCP Server，把部分业务接口包装为 AI 可调用工具 |

技术栈以 TypeScript 为主：

- 后端框架：NestJS 11
- 数据库：MongoDB + Mongoose
- AI 编排：LangChain + DeepSeek
- 鉴权：JWT + Passport
- 支付：支付宝预下单 + 微信支付 Native 下单
- 文档解析：`pdf-parse`、`mammoth`
- 监控：`prom-client`、Prometheus、Grafana
- 日志：Winston + DailyRotateFile
- 测试：Jest、Supertest

## 2. 顶层结构

```text
.
├── src/                         # NestJS 主服务
│   ├── ai/                      # AI 模型工厂、会话管理
│   ├── auth/                    # JWT 策略、守卫、Public 装饰器
│   ├── common/                  # 响应封装、异常过滤、日志、指标、trace-id
│   ├── interview/               # AI 简历押题、模拟面试、分析报告
│   ├── payment/                 # 支付订单、支付宝/微信支付 Provider
│   ├── sts/                     # 预留模块，目前没有实际接口
│   ├── user/                    # 用户注册登录、账户权益、消费记录
│   ├── wechat/                  # 微信公众号扫码登录、菜单、消息回调
│   ├── app.module.ts            # 根模块
│   └── main.ts                  # 应用启动入口
├── mcp/                         # 独立 MCP Server 子项目
├── test/                        # e2e 测试
├── README.md                    # 项目介绍文档
├── docker-compose.monitoring.yml
├── prometheus.yml
├── package.json
└── tsconfig.json
```

## 3. 运行入口与全局机制

### 3.1 启动流程

入口文件是 `src/main.ts`：

1. 读取 `NODE_ENV`，默认 `development`。
2. 通过 `createWinstonLogger()` 创建 Winston logger。
3. `NestFactory.create(AppModule, { logger })` 创建应用。
4. 注册全局 `ValidationPipe`：
   - `whitelist: true`：剔除 DTO 中未声明字段。
   - `transform: true`：自动做类型转换。
5. 开启 CORS。
6. 监听 `PORT`，默认 `3000`。

### 3.2 根模块

`src/app.module.ts` 是主装配点，导入：

- `ConfigModule.forRoot({ envFilePath: '.env.development', isGlobal: true })`
- `MongooseModule.forRoot(process.env.MONGODB_URI || 'mongodb://localhost:27017/wwzhidao')`
- `WinstonModule`
- `PassportModule`
- 全局 `JwtModule`
- `UserModule`
- `WechatModule`
- `PaymentModule`
- `StsModule`
- `InterviewModule`
- `MetricsModule`

全局增强能力：

| 能力 | 实现 | 状态 |
| --- | --- | --- |
| 响应统一封装 | `ResponseInterceptor` 作为 `APP_INTERCEPTOR` | 已全局注册 |
| 异常统一处理 | `AllExceptionsFilter` 作为 `APP_FILTER` | 已全局注册 |
| Trace ID | `TraceIdMiddleware` 作用于 `*` | 已启用 |
| JWT 鉴权 | `JwtStrategy` + `JwtAuthGuard` | 已实现，按接口使用 |
| 指标统计 | `MetricsInterceptor` | 只作为 provider 注册，未作为全局拦截器启用 |

## 4. 模块分析

### 4.1 UserModule

位置：`src/user/`

核心职责：

- 用户注册：`POST /user/register`
- 用户登录：`POST /user/login`
- 当前用户信息：`GET /user/info`
- 更新资料：`PUT /user/profile`
- 用户消费记录：`GET /user/consumption-records`

主要实现：

- `UserService.register()` 检查用户名/邮箱唯一性，创建用户。
- `UserSchema.pre('save')` 使用 `bcryptjs` 加密密码。
- `UserService.login()` 校验邮箱和密码，签发 JWT。
- `UserService.getUserConsumptionRecords()` 从 `interview/schemas/consumption-record.schema.ts` 聚合用户消费记录统计。

核心数据模型 `User`：

- 身份字段：`username`、`email`、`phone`、`password`
- 微信字段：`openid`、`unionid`、`wechatNickname`、`wechatAvatar`、`isWechatBound`
- 权益字段：
  - `wwCoinBalance`
  - `resumeRemainingCount`
  - `specialRemainingCount`
  - `behaviorRemainingCount`
  - `aiInterviewRemainingCount`
  - `aiInterviewRemainingMinutes`
- 支付幂等字段：`processedOrders`

注意：项目里存在两个消费记录模型：

- `src/interview/schemas/consumption-record.schema.ts`：主业务消费记录，简历押题/面试消耗使用它。
- `src/user/schemas/consumption-record.schema.ts`：更早期或轻量的用户消费记录模型，`UserService.createConsumptionRecord()` 使用它，但当前主查询走的是 interview 模型。

### 4.2 Auth

位置：`src/auth/`

核心文件：

- `jwt.strategy.ts`
- `jwt-auth.guard.ts`
- `public.decorator.ts`

鉴权模式：

- 通过 `Authorization: Bearer <token>` 提取 JWT。
- `@Public()` 标记公开接口。
- `JwtAuthGuard` 读取 `IS_PUBLIC_KEY`，公开接口直接放行，否则执行 Passport JWT 验证。

JWT 默认密钥为 `wwzhidao-secret`，生产环境必须通过 `JWT_SECRET` 覆盖。

### 4.3 WechatModule

位置：`src/wechat/`

主要接口：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/wechat/qrcode` | 生成微信公众号临时二维码 |
| `POST` | `/wechat/validateToken` | 接收微信消息/扫码事件 XML |
| `GET` | `/wechat/check-qr-status?id=...` | 前端轮询二维码扫描状态 |
| `POST` | `/wechat/create-menu` | 创建公众号菜单 |

扫码登录流程：

1. 前端调用 `/wechat/qrcode`。
2. 服务端调用微信接口生成临时二维码，并把状态存入内存 `Map<string, QrCodeState>`。
3. 用户扫码后，微信推送 `subscribe` 或 `SCAN` 事件到 `/wechat/validateToken`。
4. 服务端根据 `EventKey` 找到二维码状态并标记 `CONFIRMED`。
5. 前端轮询 `/wechat/check-qr-status`。
6. 服务端通过 `openid` 查找或创建用户，生成 JWT 返回。

工程特点：

- 微信 `access_token` 有内存缓存。
- 支持公众号自定义菜单。
- 可以上传图片素材并以图文方式回复菜单点击事件。

### 4.4 PaymentModule

位置：`src/payment/`

主要接口：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/payment/order` | 创建支付订单 |
| `POST` | `/payment/order/status` | 主动查询支付状态 |

支付通道：

- `alipay`
- `wechat`

套餐校验在 `PaymentService.planAmountMap` 中完成：

| 套餐 | 金额规则 |
| --- | --- |
| `custom` | 整数，1 到 10000 |
| `single` | 18.8 |
| `pro` | 28.8 |
| `max` | 68.8 |
| `ultra` | 128.8 |

创建订单流程：

1. 校验套餐 ID 和金额。
2. 生成订单号。
3. 构建支付 metadata。
4. 写入 `PaymentRecord`，状态为 `pending`。
5. 根据支付通道调用：
   - `AlipayPaymentService.initiatePayment()`
   - `WechatPaymentService.initiatePayment()`
6. 返回二维码支付地址 `codeUrl`。

支付成功落账流程：

1. 主动查询支付宝/微信支付状态。
2. 成功时调用 `finalizePaymentSuccess()`。
3. 用 `findOneAndUpdate` 将订单从 `pending` 原子更新为 `processing`，防止重复发货。
4. 校验实付金额。
5. 根据套餐发放权益：
   - `custom`：增加 `wwCoinBalance`
   - `single`：增加 `specialRemainingCount`
   - `pro`：增加简历押题、专项面试、综合面试各 1 次
   - `max`：三类各 3 次
   - `ultra`：简历 6 次、专项 16 次、综合 8 次
6. 通过 `processedOrders` 防止用户权益重复发放。
7. 写入 `UserTransaction` 充值流水。
8. 将 `PaymentRecord` 标记为 `success`。

### 4.5 InterviewModule

位置：`src/interview/`

这是项目业务复杂度最高的模块，包含：

- 简历分析
- 多轮对话
- 简历押题 SSE
- 模拟面试 SSE
- 面试暂停/恢复/结束
- 分析报告生成
- 旺旺币兑换套餐

主要接口：

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| `POST` | `/interview/analyze-resume` | JWT | 简历分析，创建会话 |
| `POST` | `/interview/continue-conversation` | 无 | 基于 sessionId 继续对话 |
| `POST` | `/interview/resume/quiz/stream` | JWT | 简历押题 SSE 流式生成 |
| `POST` | `/interview/mock/start` | JWT | 开始模拟面试 SSE |
| `POST` | `/interview/mock/answer` | JWT | 回答问题后生成下一题 SSE |
| `POST` | `/interview/mock/end/:resultId` | JWT | 主动结束面试 |
| `POST` | `/interview/mock/pause/:resultId` | JWT | 暂停面试 |
| `POST` | `/interview/mock/resume/:resultId` | JWT | 恢复面试 |
| `GET` | `/interview/analysis/report/:resultId` | JWT | 查询押题或模拟面试分析报告 |
| `POST` | `/interview/exchange-package` | JWT | 用旺旺币兑换次数 |

#### 简历押题流程

接口：`POST /interview/resume/quiz/stream`

核心方法：`InterviewService.generateResumeQuizWithProgress()`

流程：

1. 如果请求携带 `requestId`，先查 `ConsumptionRecord.metadata.requestId`，防止重复提交。
2. 原子扣减 `User.resumeRemainingCount`。
3. 创建 `ConsumptionRecord`，状态为 `pending`。
4. 提取简历内容：
   - 优先使用 `resumeContent`
   - 其次使用 `resumeURL` 下载并解析 PDF/DOCX
   - `resumeId` 目前只记录在输入数据中，没有看到实际查询简历库的实现
5. 调用 `InterviewAIService.generateResumeQuizQuestionsOnly()` 生成问题。
6. 调用 `InterviewAIService.generateResumeQuizAnalysisOnly()` 生成匹配度分析。
7. 保存 `ResumeQuizResult`。
8. 更新 `ConsumptionRecord` 为 `success`。
9. 通过 SSE 推送进度和完成数据。
10. 如果流程失败，调用 `refundCount()` 归还次数并记录失败状态。

关键数据：

- 消费类型：`resume_quiz`
- 结果表：`ResumeQuizResult`
- 幂等键：`metadata.requestId`
- 权益字段：`resumeRemainingCount`

#### 模拟面试流程

接口：

- `POST /interview/mock/start`
- `POST /interview/mock/answer`

支持两类面试：

- `special`：专项面试
- `behavior`：综合/行为/HR 面试

开始流程：

1. 根据面试类型选择扣费字段：
   - `specialRemainingCount`
   - `behaviorRemainingCount`
2. 原子扣减用户次数。
3. 提取简历内容。
4. 创建内存会话 `InterviewSession`。
5. 创建 `AIInterviewResult`，状态为 `in_progress`，并保存完整 `sessionState`。
6. 创建 `ConsumptionRecord`，状态为 `success`。
7. 调用 AI 流式生成开场白。
8. SSE 推送 `start` 和 `waiting` 事件。

回答流程：

1. 根据 `sessionId` 找到内存会话；如果内存不存在，会尝试从数据库的 `sessionState` 恢复。
2. 记录候选人回答。
3. 根据面试历史、简历、JD、已用时间调用 AI 生成下一题。
4. AI 输出中使用 `[STANDARD_ANSWER]` 分割问题和参考答案。
5. SSE 先推送问题，再推送参考答案。
6. 同步更新 `AIInterviewResult.qaList` 和 `sessionState`。
7. 达到结束条件后保存最终结果并触发报告生成。

暂停/恢复：

- `pauseMockInterview()` 将状态更新为 `paused`，保存 `pausedAt` 和 `sessionState`。
- `resumeMockInterview()` 读取 `sessionState`，状态改回 `in_progress`，并放回内存会话池。

报告查询：

- `getAnalysisReport()` 会先查 `ResumeQuizResult`，再查 `AIInterviewResult`。
- 如果模拟面试报告状态是 `pending` 或 `failed`，会触发 `generateAssessmentReportAsync()`。
- 如果状态是 `generating`，接口返回“报告生成中”的业务异常。

#### 旺旺币兑换

接口：`POST /interview/exchange-package`

规则：

- 每次兑换消耗 `20` 旺旺币。
- 每次兑换增加 `1` 次对应权益。
- 支持类型：
  - `resume` -> `resumeRemainingCount`
  - `special` -> `specialRemainingCount`
  - `behavior` -> `behaviorRemainingCount`
- 兑换后写入 `UserTransaction`，类型为 `expense`，货币为 `WWB`。

### 4.6 AIModule

位置：`src/ai/`

职责：

- `AIModelFactory`：创建 DeepSeek 模型实例。
- `SessionManager`：维护传统简历分析对话的内存会话。

模型工厂提供三种创建方式：

| 方法 | 温度 | 场景 |
| --- | --- | --- |
| `createDefaultModel()` | 环境变量或 0.7 | 默认生成 |
| `createStableModel()` | 0.3 | 稳定评估 |
| `createCreativeModel()` | 0.8 | 创意生成 |

环境变量：

- `DEEPSEEK_API_KEY`
- `DEEPSEEK_MODEL`
- `DEEPSEEK_TEMPERATURE`
- `DEEPSEEK_MAX_TOKENS`

### 4.7 MetricsModule

位置：`src/common/metrics/`

接口：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/metrics` | 输出 Prometheus 文本格式指标 |

指标类型：

- HTTP 请求总数：`http_requests_total`
- HTTP 响应耗时：`http_request_duration_ms`
- DB 查询耗时：`db_query_duration_ms`
- DB 活跃连接：`db_active_connections`
- AI 调用次数：`ai_calls_total`
- AI 调用耗时：`ai_call_duration_ms`
- AI token 消耗：`ai_tokens_used_total`
- AI 成本：`ai_cost_total`
- 虚拟币消费：`virtual_coin_spent_total`
- 完成面试数：`interviews_completed_total`
- 在线用户数：`online_users`
- 错误数：`errors_total`

配套部署：

- `docker-compose.monitoring.yml` 启动 Prometheus、Grafana、node-exporter。
- `prometheus.yml` 默认抓取：
  - `localhost:3000/metrics`
  - `localhost:9100`

### 4.8 MCP 子项目

位置：`mcp/`

这是一个独立 Node/TypeScript 项目，使用 stdio 启动 MCP Server。

配置：

- `WWZHIDAO_API_BASE_URL`：后端地址，默认 `http://127.0.0.1:8888`
- `WWZHIDAO_JWT_TOKEN`：访问后端接口所需 JWT

注册的 MCP 工具：

| 工具名 | 底层接口 |
| --- | --- |
| `get_current_user_info` | `GET /user/info` |
| `get_user_consumption_records` | `GET /user/consumption-records` |
| `get_resume_quiz_history` | `GET /interview/resume/quiz/history?page=&limit=` |
| `get_resume_quiz_result_detail` | `GET /interview/resume/quiz/result/:resultId` |
| `get_analysis_report` | `GET /interview/analysis/report/:resultId` |

需要注意：主服务当前没有实现 `/interview/resume/quiz/history` 和 `/interview/resume/quiz/result/:resultId` 这两个路由，MCP 对应工具会调用失败。

## 5. 数据模型汇总

| 模型 | 位置 | 作用 |
| --- | --- | --- |
| `User` | `src/user/schemas/user.schema.ts` | 用户身份、微信绑定、权益余额、订单去重 |
| `ConsumptionRecord` | `src/interview/schemas/consumption-record.schema.ts` | 简历押题、专项面试、综合面试等功能消耗记录 |
| `ResumeQuizResult` | `src/interview/schemas/interview-quiz-result.schema.ts` | 简历押题问题、匹配度、雷达图、学习建议 |
| `AIInterviewResult` | `src/interview/schemas/ai-interview-result.schema.ts` | 模拟面试问答、会话状态、评分报告 |
| `PaymentRecord` | `src/payment/payment-record.schema.ts` | 支付订单、支付状态、通知载荷 |
| `UserTransaction` | `src/user/schemas/user-transaction.schema.ts` | 充值/消费流水 |
| `UserConsumption` | `src/user/schemas/consumption-record.schema.ts` | 旧版或轻量消费记录，目前不是主消费查询模型 |

核心索引：

- `ConsumptionRecord`：
  - `{ userId: 1, type: 1, createdAt: -1 }`
  - `{ userId: 1, status: 1 }`
  - `{ requestId: 1 }`
- `ResumeQuizResult`：
  - `{ userId: 1, createdAt: -1 }`
  - `{ userId: 1, company: 1 }`
  - `{ userId: 1, isArchived: 1 }`
- `AIInterviewResult`：
  - `{ userId: 1, interviewType: 1, createdAt: -1 }`
  - `{ userId: 1, overallScore: -1 }`
  - `{ userId: 1, status: 1, updatedAt: -1 }`

## 6. 核心业务关系

```mermaid
flowchart LR
  Client[前端/H5] --> API[NestJS API]
  API --> Auth[JWT 鉴权]
  API --> User[用户与权益]
  API --> Wechat[微信扫码登录]
  API --> Payment[支付订单]
  API --> Interview[AI 面试业务]
  Interview --> DeepSeek[DeepSeek / LangChain]
  Interview --> Mongo[(MongoDB)]
  Payment --> Alipay[支付宝]
  Payment --> WechatPay[微信支付]
  User --> Mongo
  Wechat --> Mongo
  Metrics[/metrics] --> Prometheus[Prometheus/Grafana]
  MCP[MCP Server] --> API
```

## 7. 环境变量

仓库中存在 `.env.production`，可见键名包括：

- `DB_TYPE`
- `JWT_SECRET`
- `JWT_EXPIRES_IN`
- `NODE_ENV`
- `DEEPSEEK_API_KEY`
- `DEEPSEEK_MODEL`
- `MAX_TOKENS`

代码还引用了以下环境变量，实际部署时需要补齐：

- 基础：
  - `PORT`
  - `MONGODB_URI`
  - `JWT_SECRET`
  - `JWT_EXPIRES_IN`
- DeepSeek：
  - `DEEPSEEK_API_KEY`
  - `DEEPSEEK_MODEL`
  - `DEEPSEEK_TEMPERATURE`
  - `DEEPSEEK_MAX_TOKENS`
- 微信公众号：
  - `WECHAT_APP_ID`
  - `WECHAT_APP_SECRET`
  - `WECHAT_REDIRECT_URI`
  - `WECHAT_TOKEN`
  - `WECHAT_ENCODING_AES_KEY`
- 微信支付：
  - `WECHAT_PAY_APP_ID`
  - `WECHAT_PAY_MCH_ID`
  - `WECHAT_PAY_NOTIFY_URL`
  - `WECHAT_PAY_MCH_SERIAL`
  - `WECHAT_PAY_PRIVATE_KEY`
  - `WECHAT_PAY_API_V3_KEY`
  - `WECHAT_PAY_API_BASE`
- 支付宝：
  - `ALIPAY_GATEWAY`
  - `ALIPAY_APP_ID`
  - `ALIPAY_PRIVATE_KEY`
  - `ALIPAY_PUBLIC_KEY`
  - `ALIPAY_NOTIFY_URL`
- MCP：
  - `WWZHIDAO_API_BASE_URL`
  - `WWZHIDAO_JWT_TOKEN`

## 8. 测试与工程状态

当前测试文件：

- `src/app.controller.spec.ts`
- `src/user/user.controller.spec.ts`
- `src/user/user.service.spec.ts`
- `test/app.e2e-spec.ts`

覆盖现状：

- `AppController` 只验证根路由返回 `Hello World!`。
- `UserController` 和 `UserService` 测试只检查定义存在，而且测试模块没有 mock `UserService`、Mongoose Model、`JwtService` 等依赖，实际运行时大概率会因为依赖未解析而失败。
- 复杂业务没有测试覆盖：支付幂等、权益发放、简历押题失败回滚、SSE、模拟面试恢复、报告生成等都未覆盖。

仓库状态：

- 主项目没有 `node_modules`。
- 主项目没有 `package-lock.json` 或 `pnpm-lock.yaml`。
- `mcp/package.json` 声明 `packageManager: pnpm@10.26.0`，但主项目未声明统一包管理器。

## 9. 主要风险与改进建议

### 9.1 配置加载不一致

`ConfigModule` 固定读取 `.env.development`，但仓库只有 `.env.production`。同时 `MongooseModule.forRoot()` 使用的是 `process.env.MONGODB_URI`，不是 `ConfigService`。

影响：

- 本地启动时可能读不到预期环境变量。
- 生产环境如果没有外部注入变量，会回退到默认 MongoDB 地址和默认 JWT 密钥。

建议：

- 根据 `NODE_ENV` 动态选择 `.env.${NODE_ENV}`。
- 所有配置统一通过 `ConfigService` 获取。
- 增加 Joi/Zod 环境变量校验。

### 9.2 微信登录 JWT payload 不一致

普通登录签发 payload 包含 `userId`，但微信登录 `generateJwt()` 签发的是 `sub`，没有 `userId`。`JwtStrategy.validate()` 只读取 `payload.userId`。

影响：

- 微信扫码登录返回的 token 访问受保护接口时，`req.user.userId` 可能为 `undefined`。
- 简历押题、支付、用户信息等依赖 `userId` 的接口会出现异常或数据归属错误。

建议：

- 统一 JWT payload，例如始终包含 `userId`。
- `JwtStrategy.validate()` 兼容 `payload.userId ?? payload.sub`。

### 9.3 支付缺少异步回调入口

支付 Provider 支持 `notifyUrl`，但 `PaymentController` 目前只暴露创建订单和主动查单，没有看到支付宝/微信支付回调处理接口。

影响：

- 如果前端不轮询，支付成功不会自动落账。
- 支付平台异步通知无法验签、入库和触发权益发放。

建议：

- 增加 `/payment/callback/alipay` 和 `/payment/callback/wechat`。
- 回调中完成验签、幂等状态转换、权益发放和审计日志。

### 9.4 MetricsInterceptor 未全局启用

`MetricsInterceptor` 只是作为 provider 注册，未使用 `APP_INTERCEPTOR` 或 `@UseInterceptors()`。

影响：

- `/metrics` 能输出指标注册表，但 HTTP 请求计数和耗时可能没有实际写入。

建议：

- 将 `MetricsInterceptor` 注册为全局 `APP_INTERCEPTOR`。
- 注意与 `ResponseInterceptor` 的顺序。

### 9.5 MCP 工具调用了不存在的后端路由

`mcp/src/api.ts` 调用了：

- `/interview/resume/quiz/history`
- `/interview/resume/quiz/result/:resultId`

但 `InterviewController` 当前没有对应路由。

影响：

- `get_resume_quiz_history`
- `get_resume_quiz_result_detail`

这两个 MCP 工具会失败。

建议：

- 在 `InterviewController` 补齐历史列表和详情接口。
- 或调整 MCP 工具调用已有接口。

### 9.6 微信二维码状态使用进程内 Map

`WechatService.qrCodeStore` 是内存 `Map`。

影响：

- 服务重启后二维码状态丢失。
- 多实例部署时扫码回调可能打到另一台实例，导致登录失败。
- 过期二维码主要在轮询时清理，缺少定时清理。

建议：

- 使用 Redis 存储二维码状态。
- 设置 TTL。
- 多实例共享状态。

### 9.7 微信消息入口缺少签名校验

`/wechat/validateToken` 直接解析 XML 并处理事件，没有看到 `signature/timestamp/nonce` 校验。

影响：

- 可能被伪造微信回调请求。

建议：

- 按微信公众号规则校验签名。
- 区分首次服务器验证和消息推送处理。

### 9.8 `continue-conversation` 未加鉴权和归属校验

`POST /interview/continue-conversation` 没有 `@UseGuards(JwtAuthGuard)`。

影响：

- 知道 `sessionId` 的调用方可能继续任意会话。
- 会话在内存中，缺少用户归属检查。

建议：

- 添加 JWT 鉴权。
- 会话数据中校验 `userId`。

### 9.9 简历 URL 解析存在 SSRF 风险

`DocumentParserService` 接收 URL 后直接用 axios 下载。虽然限制了文件扩展名和大小，但没有看到域名白名单、协议限制、内网 IP 拦截。

影响：

- 如果接口暴露给外部，可能被利用访问内网地址或元数据服务。

建议：

- 只允许 HTTPS。
- 限制到可信 OSS/CDN 域名。
- 拦截 localhost、内网 IP、链路本地地址。
- 校验实际响应 Content-Type。

### 9.10 测试覆盖不足

当前测试主要是默认脚手架级别，无法保护支付、权益、AI 面试和 SSE 这些核心路径。

建议优先补：

- `PaymentService.finalizePaymentSuccess()` 幂等和权益发放测试。
- `InterviewService.executeResumeQuiz()` 扣费失败回滚测试。
- 微信 JWT payload 测试。
- 模拟面试暂停/恢复测试。
- MCP 路由契约测试。

## 10. 建议迭代优先级

| 优先级 | 建议 | 原因 |
| --- | --- | --- |
| P0 | 修复微信 JWT payload 不一致 | 直接影响扫码登录后的受保护接口 |
| P0 | 补齐支付回调和验签 | 支付落账不能只依赖前端主动查单 |
| P0 | 环境变量校验与 `.env` 加载策略 | 防止默认密钥、默认数据库误上生产 |
| P1 | 启用 MetricsInterceptor | 让监控数据真实可用 |
| P1 | 修复 MCP 缺失路由 | 避免 MCP 工具不可用 |
| P1 | 把二维码状态迁移到 Redis | 支持重启和多实例 |
| P1 | 加强简历 URL 下载安全 | 降低 SSRF 风险 |
| P2 | 补核心业务单测/集成测试 | 保护复杂业务回归 |
| P2 | 统一包管理器和 lockfile | 提升可复现安装与部署稳定性 |

## 11. 总结

该项目已经具备较完整的 AI 面试 SaaS 后端雏形：NestJS 模块边界清晰，AI 面试链路、支付权益链路、微信扫码登录、Prometheus 监控和 MCP 封装都有落点。核心业务复杂度主要集中在 `InterviewService` 和 `PaymentService`：前者负责 SSE、AI 编排、会话恢复和报告生成；后者负责套餐校验、订单状态机、幂等发货和交易流水。

当前更像“功能已打通、工程化待补强”的阶段。最值得优先处理的是鉴权一致性、支付回调、配置校验、MCP 路由契约和核心业务测试。这些问题解决后，项目的可部署性、可观测性和长期维护性会明显提升。
