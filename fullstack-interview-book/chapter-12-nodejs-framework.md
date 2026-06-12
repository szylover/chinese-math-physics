# 第十二章：Node.js 框架与中间件

Node.js 后端面试不会只问“你会不会用框架”，而是会追问：为什么选它、它的中间件模型是什么、验证和序列化如何做、依赖注入（Dependency Injection）是否真正理解、认证授权如何落地、ORM 与数据库访问模式如何取舍。本章按主流框架与基础设施展开。

## 12.1 框架对比总览

```text
+-----------+----------------+----------------+------------------+
| Framework | Middleware     | TS Support     | Best Use Case    |
+-----------+----------------+----------------+------------------+
| Express   | Linear         | Medium         | Legacy / Simple  |
| Koa       | Onion          | Medium         | Clean async flow |
| Fastify   | Hook + Plugin  | Strong         | High performance |
| NestJS    | Module + DI    | Very strong    | Large backends   |
+-----------+----------------+----------------+------------------+
```

| Feature | Express | Koa | Fastify | NestJS |
|---|---|---|---|---|
| 性能 | 中等 | 中等偏高 | 高 | 中等偏高 |
| 中间件模型 | 线性（linear） | 洋葱模型（onion model） | Hook + Plugin | 基于平台适配器 |
| TypeScript 支持 | 一般 | 一般 | 较强 | 非常强 |
| 生态 | 最成熟 | 较成熟 | 快速增长 | 企业级完整 |
| 学习曲线 | 低 | 中 | 中 | 高 |

### 面试题 1
**题目**：Express、Koa、Fastify、NestJS 应该如何选型？

**考察要点**：团队规模、性能、工程复杂度、TS 需求。

**参考答案**：小型项目、历史项目改造、快速搭服务时，Express 成本最低；喜欢简洁 `async/await` 中间件模型且团队有较强自律，可选 Koa；追求吞吐、低开销 JSON API、重视 schema 校验与序列化，可优先 Fastify；团队较大、模块边界复杂、需要依赖注入、统一规范、测试体系、微服务扩展时，NestJS 往往最稳。面试中不要说“哪个最好”，而要说“哪个最适合当前团队与场景”。

## 12.2 Express

### 面试题 2
**题目**：Express 的中间件链（middleware chain）是怎样工作的？

**考察要点**：线性模型、`req`/`res`/`next`。

**参考答案**：Express 采用线性链式中间件模型，一个请求进入后依次经过中间件，每个中间件通过 `req`、`res`、`next` 参与处理。调用 `next()` 才会进入下一个中间件；若直接响应，则链路终止。它简单易懂，但复杂系统里容易出现调用顺序分散、跨切面逻辑难统一的问题。

```ts
import express from 'express';
import crypto from 'node:crypto';
const app = express();

app.use((req, _res, next) => {
  req.headers['x-request-id'] ??= crypto.randomUUID();
  next();
});

app.use((req, res) => {
  res.json({ ok: true, requestId: req.headers['x-request-id'] });
});
```

### 面试题 3
**题目**：Express 的错误处理中间件为什么要写四个参数？

**考察要点**：`(err, req, res, next)` 签名识别。

**参考答案**：Express 通过函数签名区分普通中间件和错误处理中间件。只有写成 `(err, req, res, next)` 这种四参数形式时，Express 才会在出现错误时把请求路由到它。很多候选人知道“要这么写”，但不知道这是框架识别机制的一部分。

### 面试题 4
**题目**：Express 中如何处理异步错误？

**考察要点**：Promise rejection、包装函数、统一错误出口。

**参考答案**：原生 Express 对 `async` handler 抛出的异常处理并不如现代框架自然，常见做法是写一个 `asyncHandler` 包装器，把 Promise rejection 交给 `next(err)`。核心目标是把所有错误收敛到统一错误处理中间件，避免每个路由重复 `try/catch`。

```ts
const asyncHandler = <T extends express.RequestHandler>(fn: T): express.RequestHandler =>
  (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
```

### 面试题 5
**题目**：Express Router 的价值是什么？

**考察要点**：路由分层、模块化组织。

**参考答案**：`express.Router()` 允许把一组相关路由及其专属中间件封装成子路由模块，例如 `/users`、`/orders`、`/admin` 分开维护。这样比所有接口都堆在 `app.ts` 更可扩展。面试中可进一步说明：Router 只是组织结构，不等于完整的模块系统，复杂项目仍需额外引入 service/repository 分层。

## 12.3 Koa

### 面试题 6
**题目**：Koa 的洋葱模型（Onion Model）是什么意思？

**考察要点**：进入时自上而下、返回时自下而上。

**参考答案**：Koa 中间件基于 `async/await`，每个中间件都可以在 `await next()` 前后写逻辑，请求进入时层层向内，响应返回时层层向外，像洋葱一样。它特别适合日志、耗时统计、事务边界、统一响应包装等横切逻辑。

```ts
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  ctx.set('x-cost-ms', String(Date.now() - start));
});
```

### 面试题 7
**题目**：Koa 的 `context` 对象和 Express 的 `req/res` 有什么区别？

**考察要点**：统一上下文抽象。

**参考答案**：Koa 把 Node 的 request/response 再封装为 `ctx`，把请求与响应常用能力聚合在一个对象上，例如 `ctx.request`、`ctx.response`、`ctx.state`、`ctx.body`。Express 更直接暴露 `req`/`res`，灵活但分散。Koa 的 `ctx` 风格更利于编写统一中间件。

### 面试题 8
**题目**：Koa 相比 Express 的现代化优势是什么？

**考察要点**：`async/await`、更简内核、更少历史包袱。

**参考答案**：Koa 更“薄”、更现代，天然围绕 `async/await` 设计，没有 Express 那么多历史兼容包袱。对于强调中间件可组合性和清晰异步控制流的团队，Koa 更优雅。但它的“少内置”意味着你要自己做更多工程整合。

## 12.4 Fastify

### 面试题 9
**题目**：Fastify 为什么通常比 Express 更快？

**考察要点**：低开销、schema 驱动、序列化优化。

**参考答案**：Fastify 的快不是“魔法”，而是因为它从设计上减少运行时开销：  
1. 基于 schema 做请求校验与响应序列化；  
2. 插件封装和生命周期设计清晰；  
3. 避免很多动态层层包装；  
4. 对 JSON 序列化路径有专门优化。  
所以它特别适合结构化 API 服务。

### 面试题 10
**题目**：Fastify 的插件系统（Plugin System）有什么价值？

**考察要点**：封装、作用域、复用。

**参考答案**：Fastify 插件是其核心组织单元。路由、数据库、鉴权、工具函数都可以通过插件注册，并形成作用域边界。一个插件里声明的 decorator、hook、route 可以只在局部可见，这比“全局挂满一切”的做法更可维护。

### 面试题 11
**题目**：Fastify 为什么强调 schema-based validation and serialization？

**考察要点**：输入校验、输出约束、性能收益。

**参考答案**：很多服务只重视“请求校验”，但忽略“响应序列化”。Fastify 通过 schema 同时定义输入与输出：一方面能更早拦截非法数据，另一方面能让序列化路径可优化、响应结构更稳定。这也是它吞吐高的原因之一。

```ts
fastify.get('/users/:id', {
  schema: {
    params: {
      type: 'object',
      required: ['id'],
      properties: { id: { type: 'string' } }
    },
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          name: { type: 'string' }
        }
      }
    }
  }
}, async (request) => {
  return { id: (request.params as { id: string }).id, name: 'alice' };
});
```

### 面试题 12
**题目**：Fastify 的 decorators 和 hooks 分别适合做什么？

**考察要点**：能力注入与生命周期切面。

**参考答案**：decorator 用于给 `fastify` 实例、request 或 reply 注入复用能力，例如 `request.user`、`fastify.db`；hooks 用于生命周期切面，例如 `onRequest`、`preHandler`、`onSend`。前者更像能力扩展，后者更像流程插桩。

## 12.5 NestJS 深入

```text
Client -> Controller -> Service -> Repository/ORM -> Database
            |             |
         Guard/Pipe   Interceptor
            |             |
        Exception Filter (cross-cutting)
```

### 面试题 13
**题目**：NestJS 为什么适合大型后端项目？

**考察要点**：模块化、DI、装饰器、统一约定。

**参考答案**：NestJS 的核心价值在于“约束下的工程化”：模块系统清晰，依赖注入统一，控制器、服务、守卫、管道、拦截器、异常过滤器各司其职，TypeScript 体验完整，非常适合多人协作与长期维护。它牺牲了部分“裸写灵活性”，换来的是一致性与可测试性。

### 面试题 14
**题目**：`@Module()` 的 `imports`、`providers`、`exports` 分别是什么意思？

**考察要点**：模块依赖图与可见性。

**参考答案**：`imports` 表示当前模块依赖哪些其他模块；`providers` 是本模块内部由 IoC 容器管理的依赖，例如 service、repository；`exports` 表示把本模块中的哪些 provider 暴露给其他模块使用。理解 `exports` 很重要，它决定了依赖是“模块内私有”还是“模块间共享”。

### 面试题 15
**题目**：NestJS 的 Controller 和 Service 应该如何分工？

**考察要点**：路由适配层 vs 业务层。

**参考答案**：Controller 负责协议层适配：路由、参数获取、响应状态码、DTO 映射；Service 负责业务逻辑：校验业务规则、调用仓储、协调事务。面试中如果你把数据库访问、业务判断、第三方调用全写在 Controller 里，会被认为缺乏分层意识。

### 面试题 16
**题目**：什么是依赖注入（Dependency Injection, DI）？NestJS 的 IoC 容器做了什么？

**考察要点**：控制反转、对象生命周期、可测试性。

**参考答案**：依赖注入就是对象不自己 new 依赖，而是由外部容器提供。NestJS 的 IoC 容器会根据 provider 定义创建实例、解决依赖关系、管理作用域。它的好处是解耦、便于测试、便于替换实现，比如把真实数据库仓储换成 mock。

### 面试题 17
**题目**：Guard（守卫）适合放什么逻辑？

**考察要点**：认证与授权前置。

**参考答案**：Guard 最适合处理“是否允许进入路由”的问题，例如 JWT 校验、角色权限判断、租户隔离检查。它返回布尔值或抛异常，位于真正业务逻辑之前，因此很适合做访问控制。

### 面试题 18
**题目**：Pipe（管道）在 NestJS 中的职责是什么？

**考察要点**：验证与转换。

**参考答案**：Pipe 负责把“原始输入”变成“业务可用输入”，典型场景是参数校验和类型转换。例如 URL 里的 `id` 从字符串转成数字，或通过 `class-validator` 校验 DTO 字段合法性。Guard 关注“能不能进”，Pipe 关注“进来后参数对不对”。

```ts
export class CreateUserDto {
  @IsEmail()
  email!: string;

  @Length(8, 32)
  password!: string;
}
```

### 面试题 19
**题目**：`class-validator` 和 `class-transformer` 在 NestJS 中通常怎么配合？

**考察要点**：DTO 校验与实例转换。

**参考答案**：`class-transformer` 把普通对象转换为 DTO 类实例，`class-validator` 再基于装饰器进行验证。通常通过全局 `ValidationPipe` 开启，例如 `transform: true`、`whitelist: true`，前者负责类型转换，后者负责移除未声明字段，降低脏数据注入风险。

### 面试题 20
**题目**：Interceptor（拦截器）和 Middleware 的区别是什么？

**考察要点**：框架层抽象深度、前后处理、响应映射。

**参考答案**：Middleware 更接近底层请求管线，通常与具体路由处理结果耦合较弱；Interceptor 在 Nest 抽象层次更高，能拿到 handler 执行前后的时机，非常适合做日志、缓存、统一响应包裹、超时控制等。若你需要“修改 controller 返回值”，Interceptor 通常比 Middleware 更合适。

### 面试题 21
**题目**：异常过滤器（Exception Filter）解决什么问题？

**考察要点**：统一错误格式、协议适配。

**参考答案**：Exception Filter 负责把分散抛出的异常收敛成统一响应结构，例如 `{ code, message, requestId }`。它还能区分业务异常、验证异常、系统异常，对外屏蔽内部堆栈信息。大型项目若没有统一异常出口，线上排错和前后端协作都会很痛苦。

### 面试题 22
**题目**：自定义装饰器（Custom Decorator）和 `@SetMetadata` 的使用场景是什么？

**考察要点**：声明式编程、与 Guard 配合。

**参考答案**：自定义装饰器可以把重复的元数据声明抽象出来，例如 `@Roles('admin')` 本质上可通过 `@SetMetadata('roles', ['admin'])` 实现，再由 Guard 读取元数据进行权限判断。这样可以把“权限声明”放在路由定义处，代码更直观。

### 面试题 23
**题目**：NestJS 如何支持微服务（Microservices）？

**考察要点**：Transport、消息模式、统一编程模型。

**参考答案**：NestJS 提供多种 transport 适配器，如 TCP、Redis、NATS、RabbitMQ、Kafka 等。开发者可以沿用 Controller/Handler 风格处理消息，降低从 HTTP 服务扩展到消息驱动架构的学习成本。面试里可强调：NestJS 解决的是“编程模型统一”，不是替你自动完成分布式治理。

## 12.6 ORM 与数据库模式

| Feature | Prisma | TypeORM | Drizzle | Sequelize |
|---|---|---|---|---|
| 类型安全 | 很强 | 中等 | 很强 | 较弱 |
| Migrations | 成熟 | 成熟 | 较灵活 | 成熟 |
| 性能 | 较好 | 中等 | 较好 | 中等 |
| 学习曲线 | 中 | 中 | 中 | 低 |
| Raw SQL 支持 | 支持 | 支持 | 很强 | 支持 |

### 面试题 24
**题目**：Prisma、TypeORM、Drizzle、Sequelize 怎么比较？

**考察要点**：类型安全、抽象层、团队习惯。

**参考答案**：Prisma 的开发体验和类型安全很突出，适合追求生产力的 TS 团队；TypeORM 概念传统、资料多，但在复杂场景里坑也较多；Drizzle 更接近“类型安全 SQL builder”，适合想保留 SQL 控制力的团队；Sequelize 历史最久、上手容易，但 TypeScript 体验相对弱。面试回答要避免“只看热度”，而应说明“类型安全 vs SQL 控制力 vs 学习成本”的平衡。

### 面试题 25
**题目**：什么是连接池（Connection Pooling）？为什么数据库服务必须关注它？

**考察要点**：连接复用、上限控制、避免连接风暴。

**参考答案**：数据库连接的建立成本较高，连接池通过复用连接减少握手开销，并限制并发连接数。Node.js 服务通常实例多、并发高，如果每个请求都新建连接，很快会把数据库打挂。面试里建议补充：池太小会排队，池太大又会压垮数据库，所以要结合数据库最大连接数与应用实例数一起规划。

### 面试题 26
**题目**：读写分离和只读副本（Read Replica）适合什么场景？

**考察要点**：读多写少、最终一致性权衡。

**参考答案**：当业务读远多于写时，可以把查询压力导向只读副本，主库负责写入。这样能显著提升读吞吐，但要接受复制延迟带来的最终一致性问题。比如用户刚更新资料，马上读取时若打到从库，可能读到旧值。面试时要主动提这个一致性代价。

### 面试题 27
**题目**：事务（Transaction）和隔离级别（Isolation Level）怎么解释？

**考察要点**：ACID、脏读、不可重复读、幻读。

**参考答案**：事务用于把多个数据库操作封装成一个原子单元。隔离级别决定并发事务之间互相可见的程度，常见问题有脏读、不可重复读、幻读。面试中最重要的是说明：隔离级别越高并不总是越好，因为它通常伴随更高锁开销与更低并发。

### 面试题 28
**题目**：什么是 N+1 查询问题？如何解决？

**考察要点**：批量查询、join、预加载、DataLoader。

**参考答案**：先查 N 条主记录，再为每条记录各发一次从表查询，就会形成 N+1。典型解决方案：SQL join、批量 `IN` 查询、ORM 的 eager loading、GraphQL 场景下使用 DataLoader 聚合。同一请求里重复查同一实体，也是可通过请求级缓存优化的。

## 12.7 认证与授权

### 面试题 29
**题目**：JWT（JSON Web Token）结构是什么？

**考察要点**：header.payload.signature。

**参考答案**：JWT 由三部分组成：`header.payload.signature`。`header` 描述算法和类型，`payload` 放声明（claims），`signature` 用密钥或私钥签名，保证未被篡改。JWT 不是加密格式，默认只是 Base64URL 编码，所以敏感信息不应直接放 payload。

### 面试题 30
**题目**：为什么线上系统通常要同时使用 access token 和 refresh token？

**考察要点**：短期访问令牌、长期刷新令牌、旋转。

**参考答案**：access token 生命周期短，泄露后的风险窗口较小；refresh token 生命周期长，用于换取新的 access token。更安全的做法是 **token rotation**：每次刷新都签发新 refresh token，并使旧 token 失效。这样一旦旧 refresh token 被盗用，更容易发现异常并阻断。

### 面试题 31
**题目**：OAuth 2.0 中授权码模式（Authorization Code）和 PKCE 是什么关系？

**考察要点**：第三方登录、公开客户端安全。

**参考答案**：授权码模式适合把用户授权过程和令牌交换分开进行。PKCE 是对授权码模式的增强，尤其适合 SPA、移动端等公开客户端，它通过 `code_verifier` / `code_challenge` 防止授权码被截获后直接换 token。现代应用集成第三方登录时，PKCE 已基本成为标配。

### 面试题 32
**题目**：client credentials 模式适合什么场景？

**考察要点**：机器到机器（machine-to-machine）。

**参考答案**：client credentials 不涉及最终用户，适用于服务到服务调用，例如内部任务系统调用权限服务、统计服务调用配置中心。它代表的是“客户端自身身份”，不是用户身份，因此不能用来替代用户登录授权。

### 面试题 33
**题目**：Passport.js 的价值是什么？

**考察要点**：策略模式（strategy pattern）。

**参考答案**：Passport.js 把不同认证方式抽象成 Strategy，例如 local、JWT、Google OAuth、GitHub OAuth。好处是把认证流程标准化，易于扩展。缺点是抽象较老、黑盒感偏强，很多团队在 NestJS/Fastify 中会自己写更轻量的认证层。

### 面试题 34
**题目**：Session-based 和 token-based 认证怎么比较？

**考察要点**：状态存储、扩展性、撤销能力。

**参考答案**：Session 模式把登录状态保存在服务端，浏览器通过 cookie 带 session id，服务端可方便地失效会话，适合传统 Web；token 模式通常无状态，扩展性好，适合前后端分离和多端接入，但撤销与续期管理更复杂。面试里常见高分回答是：没有绝对优劣，要根据部署模型、跨域需求、会话控制需求来选择。

## 本章要点

- 框架选型看团队、复杂度、性能目标，不看“谁最流行”。  
- Express 胜在简单；Koa 胜在洋葱模型；Fastify 胜在 schema 驱动性能；NestJS 胜在工程化。  
- NestJS 高频考点是 Module、DI、Guard、Pipe、Interceptor、Exception Filter。  
- ORM 选型本质是类型安全、SQL 控制力、学习成本的权衡。  
- 数据库面试必须能说清连接池、读写分离、事务隔离、N+1。  
- 认证授权必须同时理解 JWT、refresh token、OAuth2、session/token 差异。  

## 延伸阅读

- Express、Koa、Fastify、NestJS 官方文档  
- Prisma / Drizzle / TypeORM 官方文档  
- OAuth 2.0 与 OpenID Connect 官方规范  
- Passport.js 与 JWT Best Practices 相关文章  
