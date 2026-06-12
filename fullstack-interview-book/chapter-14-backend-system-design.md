# 第十四章：后端系统设计题

本章聚焦 Node.js 后端高频系统设计题。系统设计题的关键，不是把名词堆满，而是按照“需求分析 → 架构 → 数据模型 → API → 扩展性 → 关键代码”的顺序稳定展开。每个案例都给出面试表达模板与 TypeScript/Node.js 关键实现片段。

## 案例一：设计一个高并发短链接服务

**题目**：设计一个类似 bit.ly 的高并发短链接服务，要求支持短链生成、跳转、点击统计、限流和热点缓存。

**考察要点**：ID 生成、读多写少架构、缓存策略、统计异步化、分库分表。

**参考答案**：  
### 1）需求分析
- 功能需求：创建短链、按短码跳转、过期控制、自定义别名、点击统计。  
- 非功能需求：高读吞吐、低跳转延迟、高可用、可观测、抗刷。  

### 2）高层架构
```text
Client -> API Gateway -> Short URL Service -> MySQL
                      -> Redis Cache
                      -> MQ -> Analytics Worker -> OLAP / MySQL
```

### 3）核心设计
- **ID 生成**：可用 Snowflake 生成全局唯一 ID，再做 Base62 编码得到短码。  
- **存储**：MySQL 存长短链映射；Redis 缓存热点短码。  
- **跳转状态码**：永久链接用 `301`，临时营销活动更适合 `302`。  
- **统计**：点击计数、referrer、geo、UA 通过消息队列异步写入，避免影响主跳转链路。  
- **限流**：按用户或 API key 限制创建频率，防止恶意刷库。  

### 4）数据模型
- `short_urls(id, short_code, long_url, user_id, expire_at, created_at)`  
- `click_events(id, short_code, ts, ip_hash, referrer, country, ua)`  

### 5）API 设计
- `POST /api/short-urls`：创建短链  
- `GET /:code`：跳转  
- `GET /api/short-urls/:code/stats`：查询统计  

### 6）扩展性
- 热点短链直接命中 Redis；  
- 长尾数据走 MySQL；  
- 统计链路异步化；  
- 数据量增大后按 `short_code hash` 分库分表。  

```ts
function base62Encode(num: bigint): string {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  if (num === 0n) return '0';
  let n = num;
  let out = '';
  while (n > 0n) {
    out = chars[Number(n % 62n)] + out;
    n /= 62n;
  }
  return out;
}
```

### 高频追问
**题目**：如果某个短链突然成为全网热点怎么办？  
**考察要点**：缓存、热点保护、降级。  
**参考答案**：提前将热点链接预热到 Redis，设置本地进程缓存，必要时把统计异步化且允许轻微延迟；若仍有流量尖峰，可对异常来源做限流和验证码保护。

## 案例二：设计一个实时消息推送系统

**题目**：设计一个支持聊天、房间订阅、在线状态、断线重连的实时消息推送系统。

**考察要点**：WebSocket/SSE 选择、连接管理、水平扩展、消息投递语义。

**参考答案**：  
### 1）需求分析
- 功能：单聊、群聊、频道订阅、在线/离线、输入中状态、消息确认。  
- 非功能：低延迟、海量连接、至少一次（at-least-once）投递、可扩展。  

### 2）协议选择
- WebSocket：双向实时通信，聊天首选。  
- SSE：服务端单向推送，适合通知流。  
- Long Polling：兼容兜底方案。  

### 3）架构
```text
Client -> WS Gateway -> Redis Adapter / MQ -> Message Service -> DB
                      -> Presence Service
```

### 4）关键设计
- **连接管理**：心跳包 + 超时检测；客户端断线后指数退避重连。  
- **房间订阅**：节点内存保存 socket-room 映射，跨节点用 Redis Adapter 广播。  
- **消息持久化**：消息先落库再 ack，保障至少一次投递。  
- **Presence**：在线状态放 Redis，TTL 自动过期。  

### 5）API / 事件设计
- `ws:connect`、`room:join`、`message:send`、`message:ack`、`presence:update`  

```ts
type ChatMessage = {
  id: string;
  roomId: string;
  senderId: string;
  content: string;
  createdAt: string;
};
```

### 6）扩展性
- 水平扩展时使用 sticky session，或让连接固定到某网关节点并通过 Redis pub/sub 广播；  
- 大规模消息分发可引入 Kafka/RabbitMQ；  
- 慢消费者要隔离，防止拖垮整个房间广播。  

### 高频追问
**题目**：如何避免消息重复？  
**考察要点**：幂等、消息 ID、ack。  
**参考答案**：服务端生成全局消息 ID，客户端按消息 ID 去重；ack 超时后可重发，但接收方和存储层都要做幂等校验。

## 案例三：设计一个多租户 SaaS 平台

**题目**：设计一个支持多个企业租户的 SaaS 平台，要求租户数据隔离、配置可定制、支持计费。

**考察要点**：租户隔离、权限模型、性能隔离、配置中心。

**参考答案**：  
### 1）隔离策略
- **Shared DB + tenant_id**：成本最低，适合中小租户，但应用层必须强制加租户过滤。  
- **Schema per tenant**：隔离更强，运维复杂度中等。  
- **DB per tenant**：隔离最强，成本最高，适合大客户。  

### 2）高层架构
```text
Gateway -> NestJS API -> Tenant Resolver -> App Services
                               |              |
                           Tenant Config    Billing/Metering
                               |              |
                             PostgreSQL / Redis / MQ
```

### 3）关键点
- 请求进入后先解析 `tenantId`，可来自子域名、JWT claim、API key。  
- 所有 repository 默认带租户过滤，避免越权读写。  
- 租户级配置放独立配置表或配置服务。  
- 计费通过 usage metering 异步汇总 API 调用量、存储量、席位数。  
- 为防 noisy neighbor，可做租户级限流、任务队列隔离、资源配额。  

```ts
@Injectable()
export class TenantContext {
  constructor(@Inject(REQUEST) private readonly req: Request) {}
  get tenantId(): string {
    return this.req.headers['x-tenant-id'] as string;
  }
}
```

### 4）数据模型
- `tenants(id, name, plan, status)`  
- `users(id, tenant_id, email, role)`  
- `usage_records(id, tenant_id, metric, value, ts)`  

### 5）追问
**题目**：如何防止开发者漏写 `tenant_id` 条件？  
**考察要点**：框架约束。  
**参考答案**：在 NestJS + ORM 层统一封装租户仓储基类，所有查询 API 默认注入租户条件；对高危表增加数据库层策略或视图，不能只靠“开发自觉”。

## 案例四：设计一个分布式任务调度系统

**题目**：设计一个支持 cron、延迟任务、重试、死信队列的分布式调度系统。

**考察要点**：注册与调度分离、分布式锁、Worker Pool、重试机制。

**参考答案**：  
### 1）需求拆解
- 功能：任务注册、Cron 调度、延迟执行、优先级、失败重试、监控告警。  
- 非功能：单任务不重复执行、可横向扩展、支持幂等。  

### 2）架构
```text
Scheduler API -> Task Registry DB
Scheduler Leader -> Redis/DB Lock -> BullMQ Queue -> Workers
                                           |
                                       DLQ / Metrics
```

### 3）关键设计
- 任务定义与执行器注册分离；  
- 定时扫描器只负责把到期任务投递到队列；  
- 使用 Redis Redlock 或数据库乐观锁保证同一批任务不被多个 scheduler 重复投递；  
- Worker 按队列消费，失败按指数退避重试，超阈值进死信队列（DLQ）。  

```ts
import { Queue, Worker } from 'bullmq';

const queue = new Queue('jobs');
await queue.add('send-email', { userId: 'u1' }, { attempts: 5, backoff: { type: 'exponential', delay: 1000 } });

new Worker('jobs', async (job) => {
  // execute job
});
```

### 4）监控
- 任务成功率、重试率、延迟、堆积长度、DLQ 数量；  
- 任务执行日志按 job id 追踪；  
- 对关键任务配置告警。  

### 5）追问
**题目**：任务重复执行怎么办？  
**考察要点**：幂等性。  
**参考答案**：分布式系统里“绝不重复”很难保证，所以业务执行器必须具备幂等性，例如以业务键去重、状态机防重、唯一索引约束。

## 案例五：设计一个 API 网关

**题目**：设计一个统一的 API Gateway，要求负责路由、鉴权、限流、日志、熔断和安全头。

**考察要点**：边车能力、跨服务通用逻辑、性能与可维护性平衡。

**参考答案**：  
### 1）职责边界
- 路由转发与负载均衡  
- JWT / API Key 校验  
- 限流、CORS、安全头  
- 请求/响应转换  
- 日志、指标、trace id 注入  
- 下游熔断与超时控制  

### 2）架构
```text
Client -> API Gateway -> User Service
                     -> Order Service
                     -> Payment Service
                     -> Metrics / Logs / Traces
```

### 3）关键设计
- 路由配置可动态下发；  
- JWT 在网关做签名校验与基础 claim 检查，下游只做细粒度授权；  
- 限流可按 userId、IP、route 维度组合；  
- 使用断路器保护慢下游；  
- 统一加 `CORS`、`Helmet` 安全头。  

```ts
import Fastify from 'fastify';
import crypto from 'node:crypto';
import helmet from '@fastify/helmet';

const app = Fastify();
await app.register(helmet);

app.addHook('onRequest', async (req, reply) => {
  req.headers['x-request-id'] ??= crypto.randomUUID();
  // validate jwt / api key / rate limit
});
```

### 4）扩展性
- 网关本身无状态，方便水平扩展；  
- 配置中心统一管理路由与限流策略；  
- 指标上报 Prometheus，日志接 ELK/ClickHouse。  

### 5）追问
**题目**：为什么不把所有业务逻辑都放在网关？  
**考察要点**：边界与职责。  
**参考答案**：网关应该承载通用能力，而不是业务编排中心。业务逻辑堆在网关会造成耦合、发布风险和性能瓶颈。网关要“薄而强”，业务服务要“专而深”。

## 本章要点

- 系统设计先讲需求，再讲架构，再落到数据与扩展性。  
- 短链接服务本质是读多写少 + 热点缓存 + 异步统计。  
- 实时推送系统关键是连接管理、投递语义、横向扩展。  
- 多租户 SaaS 的核心是隔离策略与平台化约束。  
- 分布式调度系统要把“调度”和“执行”解耦，并依赖幂等保证一致性。  
- API 网关负责通用能力，不应吞掉业务边界。  

## 延伸阅读

- Designing Data-Intensive Applications  
- BullMQ、Redis、Kafka、RabbitMQ 官方文档  
- NestJS、Fastify 官方文档  
- Cloudflare、NGINX、Envoy、Kong 等网关产品设计资料  
