# 第十三章：Node.js 性能与安全

高水平的 Node.js 后端工程师，不只是“能把接口跑起来”，而是能解释为什么延迟升高、如何定位内存泄漏、如何构建缓存层、如何抵御安全攻击、如何在服务退出时不丢请求。性能与安全往往也是面试里最能区分“写过 Demo”和“带过生产系统”的部分。

## 13.1 性能分析（Profiling）

### 面试题 1
**题目**：Node.js 性能排查时，为什么不要一上来就“拍脑袋优化”？

**考察要点**：先测量、再定位、最后优化。

**参考答案**：性能优化必须基于证据，而不是凭感觉改代码。很多系统真正瓶颈可能在数据库、序列化、外部 API、锁竞争、GC，而不是你怀疑的那一段循环。标准流程应是：先建立指标与压测场景，再用 profiling 工具定位热点，再做有针对性的优化，最后回归验证。

### 面试题 2
**题目**：clinic.js 的 `doctor`、`flame`、`bubbleprof` 分别适合什么问题？

**考察要点**：不同可视化工具解决不同瓶颈。

**参考答案**：`doctor` 适合先做总体诊断，判断是 CPU、I/O 还是事件循环问题；`flame` 用于生成火焰图（flame graph），定位 CPU 热点函数；`bubbleprof` 更适合分析异步调用链和等待关系。面试中如果只会背工具名但说不出各自用途，说明实践不足。

### 面试题 3
**题目**：0x 火焰图（flame graph）怎么看？

**考察要点**：横轴不是时间、宽度代表采样占比。

**参考答案**：火焰图每个方块代表函数调用栈中的一帧，宽度反映采样中该函数占用 CPU 的比例；越宽说明越耗时。横轴通常不是时间线，而是栈聚合后的分布。读图时先找最宽的“平顶山”，再回溯其调用路径，而不是盯着最上层的小块。

### 面试题 4
**题目**：`perf_hooks` 能做什么？

**考察要点**：PerformanceObserver、mark/measure、histogram。

**参考答案**：`perf_hooks` 是 Node 内建性能观测工具，可以用 `performance.mark()` / `performance.measure()` 给关键路径打点，用 `PerformanceObserver` 收集结果，还能用 histogram 统计延迟分布。它比简单的 `console.time()` 更适合长期观测。

```ts
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.duration);
  }
});
obs.observe({ entryTypes: ['measure'] });

performance.mark('db-start');
// ... query
performance.mark('db-end');
performance.measure('db-query', 'db-start', 'db-end');
```

### 面试题 5
**题目**：`--inspect` 和 Chrome DevTools 在 Node.js 性能分析里怎么用？

**考察要点**：CPU profile、heap snapshot、timeline。

**参考答案**：启动服务时带 `--inspect`，然后在 Chrome DevTools 中连接 Node 进程，即可抓 CPU Profile、Heap Snapshot、Allocation Timeline。它特别适合复现某个具体场景下的热点函数或内存增长路径。线上一般不会直接开给所有流量，但预发或压测环境非常有用。

### 面试题 6
**题目**：`process.cpuUsage()` 和 `process.memoryUsage()` 有什么价值？

**考察要点**：轻量级指标。

**参考答案**：这两个 API 很轻量，适合做基础监控。`process.cpuUsage()` 可看用户态/内核态 CPU 累积时间；`process.memoryUsage()` 能看 `rss`、`heapUsed`、`heapTotal`、`external` 等。它们不能替代深度分析工具，但非常适合做趋势告警与粗粒度诊断。

## 13.2 事件循环监控

### 面试题 7
**题目**：如何衡量事件循环延迟（Event Loop Delay）？

**考察要点**：定时器漂移、监控 API。

**参考答案**：最常见方法是定期设置一个短延迟定时器，测量实际触发时间与理论触发时间的差值。若主线程被阻塞，定时器会明显“漂移”。Node 还提供更标准的 `monitorEventLoopDelay()` API，适合生产监控。

### 面试题 8
**题目**：`monitorEventLoopDelay()` 应该怎么理解？

**考察要点**：延迟分布监测、P99 视角。

**参考答案**：它会返回一个 histogram，记录事件循环延迟的统计分布。相比只看平均值，更应关注 P95/P99，因为尾延迟更能反映用户真实体验。若 P99 event loop delay 长期偏高，说明有阻塞型代码、GC 抖动或宿主机资源争用。

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  console.log('p99(ms)=', histogram.percentile(99) / 1e6);
  histogram.reset();
}, 10000);
```

### 面试题 9
**题目**：如何检测“阻塞事件循环”的代码？

**考察要点**：长同步任务、诊断路径。

**参考答案**：先看 event loop delay 是否升高，再结合 CPU Profile 找热点函数；若 CPU 不高但延迟高，可能是同步 I/O、长 GC、巨型 JSON 序列化、正则灾难回溯等。经验上，任何超过几十毫秒的同步计算都值得警惕，尤其在高并发接口中。

### 面试题 10
**题目**：什么是事件循环利用率（Event Loop Utilization, ELU）？

**考察要点**：忙碌时间占比。

**参考答案**：ELU 衡量事件循环忙碌程度，可以帮助判断主线程是否接近饱和。如果 ELU 很高且延迟上升，说明主线程压力大；若 ELU 不高但响应慢，瓶颈可能在下游依赖。它比单纯 CPU 使用率更贴近 Node.js 的运行特点。

## 13.3 内存泄漏（Memory Leak）

### 面试题 11
**题目**：Node.js 常见内存泄漏原因有哪些？

**考察要点**：闭包、全局变量、事件监听器、定时器、缓存。

**参考答案**：高频原因包括：  
- 闭包意外持有大对象；  
- 全局变量或单例 Map 持续增长；  
- `EventEmitter` 监听器未移除；  
- 定时器未清理；  
- 请求上下文被缓存；  
- 大对象数组长期保留；  
- 错误的 LRU 实现。  
“JavaScript 有 GC 所以不会泄漏”是典型误解；只要对象仍可达，GC 就不会回收。

### 面试题 12
**题目**：如何使用 heapdump 分析内存泄漏？

**考察要点**：生成多个快照、比较 retained size。

**参考答案**：先在内存持续增长场景下抓多份 heap snapshot，再在 DevTools 中比较对象数量与 retained size 变化，重点看哪些对象不断增长、谁在引用它们。单独看一份快照很难判断“增长趋势”，对比快照才更有效。

### 面试题 13
**题目**：`--max-old-space-size` 是什么？调大就等于解决内存问题吗？

**考察要点**：堆上限调优与治标不治本。

**参考答案**：`--max-old-space-size` 用来调整 V8 老生代堆上限，例如 `node --max-old-space-size=4096 app.js`。它能延缓 OOM，但不能根治泄漏。若问题是对象一直不可回收，堆再大也只是“更晚爆炸”。正确做法是先分析内存增长根因，再决定是否需要调堆上限。

### 面试题 14
**题目**：`WeakRef` 和 `FinalizationRegistry` 能解决什么问题？

**考察要点**：弱引用、缓存辅助、不能滥用。

**参考答案**：`WeakRef` 允许你持有不阻止 GC 的弱引用，`FinalizationRegistry` 允许在对象被回收后注册清理通知。它们适合某些缓存、资源映射等高级场景，但不可依赖其时机做关键业务逻辑，因为回收时间不确定。面试里最好的回答是：知道它们能做什么，也知道它们不能替代显式资源管理。

### 面试题 15
**题目**：给你一个线上 Node 服务，内存不断上涨，你的排查流程是什么？

**考察要点**：方法论。

**参考答案**：先确认是 `rss` 涨还是 `heapUsed` 涨；如果是堆涨，抓 heap snapshot 看对象增长路径；如果是 `external` 涨，要查 Buffer/native addon；结合流量与接口分布找高风险入口；对缓存、监听器、队列、定时器做专项检查；最后在压测环境复现并回归验证。面试官看重的是流程完整性，而不是你背了几个工具名。

## 13.4 缓存（Caching）

### 面试题 16
**题目**：进程内缓存（In-memory Cache）什么时候合适？什么时候危险？

**考察要点**：低延迟与一致性限制。

**参考答案**：进程内缓存如 `Map`、LRU cache，优点是极快、实现简单，适合小体量热点数据；缺点是实例间不共享、重启即丢、容易造成内存膨胀。因此它更适合做短期热点缓存，而不是作为主缓存层。面试回答要主动提“多实例一致性问题”。

### 面试题 17
**题目**：什么是 LRU 缓存？为什么比简单 `Map` 更适合生产？

**考察要点**：淘汰策略。

**参考答案**：LRU（Least Recently Used）会优先淘汰最近最少使用的数据，能够在固定容量下尽量保留热点项。简单 `Map` 若只增不删，很容易变成泄漏源。生产环境里缓存几乎都要有容量上限、TTL 和淘汰策略。

### 面试题 18
**题目**：Redis 在 Node.js 后端中常见的用途有哪些？

**考察要点**：数据结构、pub/sub、分布式协调。

**参考答案**：Redis 可做缓存、分布式锁、会话存储、排行榜、限流计数器、消息发布订阅（pub/sub）、延迟队列辅助、幂等键存储等。它的 value 不只是字符串，还包括 hash、list、set、zset、stream 等。面试里若能结合具体数据结构说应用场景，会非常加分。

### 面试题 19
**题目**：Redis 的 Lua 脚本和 Cluster Mode 各解决什么问题？

**考察要点**：原子性与水平扩展。

**参考答案**：Lua 脚本适合把多个 Redis 操作封装成原子逻辑，例如“检查次数并递增”；Cluster Mode 用于把 key 分布到多个分片节点，提高容量和吞吐。要注意：Lua 脚本在 Cluster 中最好操作同一 hash slot 的 key，否则会遇到跨槽限制。

### 面试题 20
**题目**：HTTP 缓存头 `Cache-Control`、`ETag`、条件请求分别是什么？

**考察要点**：强缓存与协商缓存。

**参考答案**：`Cache-Control` 用于定义缓存策略，如 `max-age`、`public`、`private`；`ETag` 是资源版本标识；客户端带 `If-None-Match` 发起条件请求时，若资源未变化，服务器返回 `304 Not Modified`。这能减少带宽与服务器 CPU 消耗。面试中可总结：强缓存优先，过期后再走协商缓存。

### 面试题 21
**题目**：CDN 缓存对后端系统意味着什么？

**考察要点**：边缘节点卸载流量。

**参考答案**：CDN 把静态资源甚至部分动态内容缓存到边缘节点，大幅降低源站压力和跨地域延迟。对后端而言，这意味着需要设计可缓存资源、合理设置缓存键、处理版本更新与失效策略。很多性能优化不是“把 Node 再压榨 10%”，而是“别让请求打到 Node”。

### 面试题 22
**题目**：缓存失效（Cache Invalidation）为什么难？常见策略有哪些？

**考察要点**：TTL、主动删除、双写一致性。

**参考答案**：难点在于“数据变了，所有缓存副本都得及时正确更新”，但分布式环境里时序复杂。常见策略包括 TTL 过期、更新数据库后主动删除缓存、延迟双删、消息通知失效、写穿（write-through）等。没有万能方案，要根据一致性要求选择。

## 13.5 安全（Security）

### 面试题 23
**题目**：什么是 XSS（Cross-Site Scripting）？有哪些类型？

**考察要点**：stored、reflected、DOM-based。

**参考答案**：XSS 是攻击者把恶意脚本注入页面，在受害者浏览器执行。常见类型有：存储型（stored）、反射型（reflected）、DOM 型（DOM-based）。后端面试里你要能说出：防御重点不是“过滤 `<script>`”这么简单，而是输出编码、模板自动转义、输入清洗、CSP、`httpOnly` cookie 等组合拳。

### 面试题 24
**题目**：如何防御 CSRF（Cross-Site Request Forgery）？

**考察要点**：CSRF token、SameSite、幂等设计。

**参考答案**：若系统基于 cookie 自动带认证信息，就要防 CSRF。常见做法有：表单或请求头中加入 CSRF token、启用 `SameSite` cookie、对敏感操作做二次确认。若完全采用 Authorization header 中的 Bearer token，CSRF 风险会低很多，但 XSS 风险仍要防。

### 面试题 25
**题目**：SQL 注入和 NoSQL 注入分别怎么防？

**考察要点**：参数化查询、输入约束、ORM 边界。

**参考答案**：SQL 注入核心防御是参数化查询，不拼接原始用户输入；ORM 虽有保护，但写原始 SQL 时仍可能踩坑。NoSQL 注入则常见于把用户输入直接拼进 Mongo 查询对象，例如传入 `$ne`、`$gt` 这类操作符。解决思路是：白名单校验字段、限制查询结构、对 DTO 做严格验证。

### 面试题 26
**题目**：什么是 SSRF（Server-Side Request Forgery）？Node.js 服务如何防？

**考察要点**：后端代发请求的风险、URL 校验、allowlist。

**参考答案**：SSRF 是攻击者诱导服务器去访问本不该访问的地址，比如云 metadata、内网服务、管理面板。Node.js 服务常见高风险点是“用户传一个 URL，服务端帮忙下载/截图/抓取”。防御包括：严格校验协议与主机名、DNS 解析后校验目标 IP、禁止私网地址、维护 allowlist、限制重定向次数。

### 面试题 27
**题目**：如何实现限流（Rate Limiting）？

**考察要点**：token bucket、sliding window、Redis 原子计数。

**参考答案**：常见算法有令牌桶（token bucket）、漏桶、固定窗口、滑动窗口（sliding window）。分布式服务通常用 Redis 做计数与过期控制。设计时要考虑按用户、按 IP、按接口、按租户等多维度限流，并明确超限后的响应码与降级策略。

```ts
// 伪代码：滑动窗口
// key = rate:{userId}:{minuteBucket}
// INCR + EXPIRE，通过聚合最近多个 bucket 计算总次数
```

### 面试题 28
**题目**：为什么要重视依赖安全？`npm audit`、Snyk、lockfile 各有什么作用？

**考察要点**：供应链安全。

**参考答案**：Node.js 生态依赖链很深，一个小包出问题也可能影响生产。`npm audit` 用于检测已知漏洞；Snyk 等平台可做更持续的依赖扫描与修复建议；`package-lock.json` / `pnpm-lock.yaml` 保证安装结果可重复，减少“本地没问题、线上拉到不同版本出问题”的风险。锁文件不只是性能文件，更是安全与可重复构建的一部分。

### 面试题 29
**题目**：CORS（Cross-Origin Resource Sharing）要点有哪些？什么是预检请求（preflight request）？

**考察要点**：跨域头、OPTIONS、凭证限制。

**参考答案**：浏览器在跨域情况下会检查服务端是否返回允许的 `Access-Control-*` 头。对于带自定义头、非简单方法等请求，浏览器会先发 `OPTIONS` 预检请求确认是否允许。配置 CORS 时最忌讳“全开一切”，尤其不能把 `Access-Control-Allow-Origin: *` 与 `Allow-Credentials: true` 乱配在一起。

### 面试题 30
**题目**：Helmet.js 的作用是什么？

**考察要点**：安全响应头。

**参考答案**：Helmet.js 是 Express/Fastify 常用安全中间件集合，用于设置一组安全头，例如 `Content-Security-Policy`、`X-Frame-Options`、`X-Content-Type-Options` 等。它不是万能防火墙，但能帮你快速建立基本安全基线。

## 13.6 韧性与退出流程

### 面试题 31
**题目**：什么是断路器模式（Circuit Breaker Pattern）？

**考察要点**：失败快速返回、避免级联故障。

**参考答案**：当下游服务持续失败时，断路器会从“关闭”切到“打开”，短时间内直接拒绝请求或走降级逻辑，不再继续把流量打到已经有问题的下游；过一段时间进入半开状态，少量探测恢复情况。它的目标是避免线程、连接、重试风暴把整个系统拖垮。

### 面试题 32
**题目**：Node.js 服务为什么必须做优雅关闭（Graceful Shutdown）？

**考察要点**：SIGTERM、停止接新请求、排空连接。

**参考答案**：在容器编排和滚动发布环境中，服务经常收到 `SIGTERM`。如果进程立刻退出，正在处理的请求会中断，消息可能丢失，事务可能处于危险状态。正确流程是：收到信号后停止接收新请求，等待在途请求完成，关闭数据库连接、消息消费者与定时任务，然后再退出。

```ts
import http from 'node:http';

const server = http.createServer(app);
server.listen(3000);

process.on('SIGTERM', () => {
  server.close(() => {
    // close db / redis / mq
    process.exit(0);
  });
});
```

### 面试题 33
**题目**：什么是 connection draining？

**考察要点**：长连接排空。

**参考答案**：connection draining 指服务下线前不再接新连接，但保留已有连接一段时间，让存量请求自然完成。对 keep-alive、WebSocket、SSE 等场景尤其重要。若没有 draining，发布期间会出现大量瞬时失败。

## 本章要点

- 性能优化先测量，再定位，再优化。  
- `clinic.js`、0x、DevTools、`perf_hooks` 是 Node.js 性能排查核心工具链。  
- 事件循环指标要看 delay、ELU 和尾延迟。  
- 内存泄漏排查核心是：趋势、快照对比、引用链。  
- 缓存设计要同时考虑命中率、一致性、失效策略和内存成本。  
- 安全面试高频是 XSS、CSRF、SQL/NoSQL Injection、SSRF、CORS、限流、依赖安全。  
- 生产系统必须具备断路器与优雅关闭能力。  

## 延伸阅读

- Node.js 官方文档：`perf_hooks`、诊断、内存调优  
- clinic.js、0x 官方仓库文档  
- OWASP Top 10 与 NodeGoat  
- Redis 官方文档与缓存一致性相关文章  
