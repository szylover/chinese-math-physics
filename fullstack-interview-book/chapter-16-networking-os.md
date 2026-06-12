# 第十六章：计算机网络与操作系统

这一章的特点是：题很多，但真正拉开差距的不是“背定义”，而是你能不能把协议、内核、排障经验串成一个闭环。面试官最喜欢追问两类候选人：一类只会说名词，另一类能把“浏览器发请求 -> 负载均衡 -> 服务处理 -> 网络传输 -> 内核调度 -> 日志排查”完整讲清楚。后者通常更容易拿到高评级。

---

## 一、网络部分高频面试题

### 1. HTTP/1.1、HTTP/2、HTTP/3 有什么区别？

**答题框架：**

| 维度 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| 连接模型 | 文本协议，长连接但基本串行 | 二进制分帧（binary framing），多路复用（multiplexing） | 基于 QUIC，运行在 UDP 之上 |
| 队头阻塞 | 应用层明显 | TCP 层仍可能队头阻塞 | 通过独立流减少传输层队头阻塞 |
| 头部压缩 | 无 | HPACK | QPACK |
| 握手成本 | TCP + TLS | TCP + TLS | QUIC 内建 TLS 1.3 |
| 服务器推送 | 无 | 支持 Server Push，但实际使用下降 | 不强调 Server Push |

**展开解释：**

- HTTP/1.1 支持 keep-alive，但一个连接上的多个请求通常仍受限于串行或浏览器并发连接数。
- HTTP/2 的核心价值是把多个请求拆成帧（frame），在一条 TCP 连接里并发复用多个流（stream）。
- HTTP/3 不再依赖 TCP，而是建立在 QUIC（Quick UDP Internet Connections）之上，把重传、拥塞控制、TLS 都搬到用户态协议栈里，减少连接建立与迁移成本。

**面试加分点：**

- HTTP/2 虽然能多路复用，但只要底层 TCP 丢包，整个连接还是会受影响，这就是“传输层队头阻塞”。
- HTTP/3 非常适合移动网络切换场景，例如 Wi-Fi 切 4G 时连接迁移更平滑。

### 2. HTTP/2 的多路复用为什么比 HTTP/1.1 更高效？

HTTP/1.1 常见问题是：

1. 浏览器为并行请求建立多条 TCP 连接，带来握手与拥塞窗口重复预热成本。
2. 一个请求慢，可能阻塞同连接后续请求。

HTTP/2 把每个请求拆成多个帧并附带流 ID，在连接内交错传输，服务端和客户端按流重组，因此资源利用率更高。  
**但要注意**：HTTP/2 不是“没有队头阻塞”，而是把应用层队头阻塞降下来，TCP 丢包问题仍在。

### 3. HTTP/2 的 Header Compression 是什么？

HTTP/2 使用 HPACK（Header Compression for HTTP/2）压缩请求头。现实中大量头部字段重复，例如 `cookie`、`user-agent`、`accept-language`。HPACK 通过：

- 静态表（static table）
- 动态表（dynamic table）
- Huffman 编码

减少重复传输开销。面试时可以说：**这对移动端、弱网和头部较大的 API 请求尤其有价值。**

### 4. HTTP/3 为什么选择 UDP？

不是因为 UDP 更可靠，而是因为 QUIC 希望摆脱 TCP 的历史包袱：

- 在用户态快速迭代协议栈；
- 自定义重传与拥塞控制策略；
- 连接 ID 支持迁移；
- 0-RTT/1-RTT 建连更友好。

所以真正的答案不是“HTTP/3 使用 UDP”，而是“HTTP/3 通过 QUIC 在 UDP 之上补齐可靠性、顺序控制、加密与流控能力”。

### 5. 详细讲一下 HTTPS 与 TLS 1.3 握手过程

HTTPS（HTTP Secure）本质是 **HTTP + TLS（Transport Layer Security）**。  
TLS 1.3 握手可以按下面顺序回答：

1. **ClientHello**  
   客户端发送支持的 TLS 版本、cipher suites、随机数、SNI、ALPN、公钥交换参数等。
2. **ServerHello**  
   服务端选择协议版本、密码套件，返回随机数和密钥交换参数。
3. **证书链（certificate chain）**  
   服务端发送站点证书、中间证书，客户端验证域名、有效期、签名链、吊销状态。
4. **密钥协商**  
   TLS 1.3 默认采用 (EC)DHE，双方根据密钥交换材料推导会话密钥。
5. **Finished**  
   双方发送握手摘要校验消息，证明握手过程未被篡改。
6. **应用数据加密传输**  
   后续 HTTP 数据通过协商出的对称密钥进行加密。

**为什么 TLS 1.3 更快？**

- 删掉了很多旧算法和冗余往返；
- 强制前向保密（forward secrecy）；
- 支持 0-RTT（有重放风险，适合幂等请求）。

### 6. 什么是证书链？浏览器如何验证？

站点证书通常不是根证书直接签发，而是：

根 CA -> 中间 CA -> 站点证书

浏览器会：

1. 校验证书域名是否匹配；
2. 检查有效期；
3. 校验证书链签名是否能一路追溯到受信任根；
4. 检查吊销信息（CRL/OCSP）；
5. 检查扩展字段，如 key usage。

**常见面试坑**：不要说“服务端发根证书给浏览器”。根证书通常预置在系统或浏览器信任库中。

### 7. 什么是 HSTS？

HSTS（HTTP Strict Transport Security）告诉浏览器：  
“这个域名在一段时间内只能走 HTTPS，不要再发 HTTP 请求。”

典型头部：

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

作用：

- 防止 SSL stripping（降级攻击）；
- 避免用户手输 `http://` 时被劫持。

### 8. TCP 三次握手为什么不是两次？

标准回答：

1. 客户端发 SYN，表示“我想建立连接”；
2. 服务端回 SYN + ACK，表示“我收到了，也同意建立，并确认我也能发”；
3. 客户端回 ACK，表示“我收到你的确认，双方收发能力都确认完毕”。

为什么不是两次？因为服务端无法确认自己的 SYN+ACK 是否被客户端收到。如果没有第三次确认，就无法确保双方对连接状态一致。

### 9. TCP 四次挥手为什么通常是四次？

TCP 是全双工：

- A 说“我没数据发了”（FIN）；
- B 回 ACK，表示知道了；
- 等 B 也发完数据，再发 FIN；
- A 回 ACK。

之所以不是三次，是因为“确认对方关闭发送方向”和“自己关闭发送方向”是两个动作，不一定同时发生。

### 10. TIME_WAIT 为什么存在？

TIME_WAIT 主要有两个作用：

1. **确保最后一个 ACK 能到达对方**，若对方未收到 FIN 的确认，还能重传。
2. **让旧连接中的延迟报文在网络中自然消失**，避免污染新连接。

面试延伸：

- TIME_WAIT 通常出现在主动关闭连接的一方；
- 大量短连接服务若 TIME_WAIT 很多，不一定是 bug，可能是业务模型导致；
- 优化要结合连接复用、keep-alive、反向代理、端口范围，不要只会调内核参数。

### 11. TCP 流量控制与滑动窗口是什么？

流量控制（flow control）解决的是**接收方来不及处理**的问题。  
接收端通过 TCP 窗口大小告诉发送端：“我还能接收多少字节”，发送端据此限制未确认数据量。

这与拥塞控制不同：

- 流量控制：保护接收端；
- 拥塞控制：保护网络。

### 12. TCP 拥塞控制有哪些阶段？

高频关键词必须会：

- 慢启动（slow start）
- 拥塞避免（congestion avoidance）
- 快重传（fast retransmit）
- 快恢复（fast recovery）

可按下面说：

1. 初始时拥塞窗口（cwnd）较小，指数增长；
2. 到达慢启动阈值（ssthresh）后转为线性增长；
3. 若检测到丢包，说明网络可能拥塞；
4. 收到 3 个重复 ACK 时触发快重传，不必等超时；
5. 根据算法进入快恢复或重新慢启动。

### 13. UDP 的特点与典型使用场景

UDP（User Datagram Protocol）特点：

- 无连接；
- 不保证可靠、顺序、去重；
- 首部开销小；
- 时延低。

典型场景：

- 音视频实时通信；
- DNS 查询；
- 游戏状态同步；
- QUIC。

面试加分点：UDP 不可靠不代表“不能做可靠传输”，QUIC 就是在 UDP 之上自己实现可靠机制。

### 14. WebSocket 握手过程是什么？

WebSocket 初始阶段走 HTTP Upgrade：

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ...
Sec-WebSocket-Version: 13
```

服务端若支持，会返回 `101 Switching Protocols`。此后连接升级为全双工长连接。

### 15. WebSocket 帧格式、心跳、与 SSE 的区别

**帧格式关键点：**

- FIN：是否最后一帧；
- opcode：文本、二进制、ping、pong、close；
- mask：浏览器到服务端的帧需要掩码；
- payload length：负载长度。

**心跳为什么需要？**

- 代理、LB、NAT 可能回收空闲连接；
- 业务需要探测对端是否在线。

**WebSocket vs SSE（Server-Sent Events）**

| 维度 | WebSocket | SSE |
|---|---|---|
| 通信方向 | 双向 | 服务端单向推送 |
| 传输格式 | 自定义帧 | 文本事件流 |
| 重连 | 应用层处理 | 浏览器原生支持 |
| 场景 | IM、协作、游戏 | 通知、日志流、价格推送 |

### 16. DNS 解析过程怎么讲？

从浏览器输入域名开始，一般会经过：

1. 浏览器缓存；
2. 操作系统缓存；
3. 本地 hosts；
4. 本地 DNS 解析器；
5. 若无缓存，请求递归解析器；
6. 递归解析器再按迭代方式问根域名服务器、TLD 服务器、权威 DNS；
7. 返回结果并缓存。

**递归（recursive） vs 迭代（iterative）**

- 递归：你帮我查到底；
- 迭代：你告诉我下一站去哪查。

### 17. DNS 缓存有哪些层级？

- 浏览器缓存
- OS DNS cache
- 本地解析器缓存
- 运营商 DNS 缓存
- CDN 边缘解析缓存

面试追问时可说 TTL（Time To Live）决定缓存生存时间，但现实中也可能有最小 TTL、预取、负缓存策略。

### 18. 什么是 DNS over HTTPS（DoH）？

DoH 是把 DNS 请求通过 HTTPS 发送，目标是：

- 提升隐私；
- 降低明文 DNS 被监听、篡改的风险；
- 更容易穿透某些网络限制。

但它也带来企业审计、网络治理、排障可观测性上的挑战。

### 19. CDN 是如何工作的？

CDN（Content Delivery Network）核心是把内容分发到靠近用户的边缘节点（edge node）。

典型流程：

1. 用户访问域名；
2. DNS 或调度系统把用户导向最近/最优边缘节点；
3. 边缘节点命中缓存则直接返回；
4. 未命中则回源（origin pull）到源站拉取；
5. 回写缓存并返回用户。

### 20. CDN Cache Hit 与 Cache Miss 有什么区别？

- **Cache Hit**：命中边缘缓存，延迟低、源站压力小；
- **Cache Miss**：边缘没有内容，需要回源，延迟更高。

面试里可补充：

- 命中率不是越高越好，要结合实时性；
- 静态资源适合长缓存 + 版本号；
- 动态接口一般靠边缘计算、API 缓存、KV 缓存，而不是传统文件缓存。

### 21. 什么是边缘计算（Edge Computing）？

边缘计算是把一部分逻辑放到离用户更近的位置执行，例如：

- 图片裁剪；
- A/B 路由；
- Token 预校验；
- Header 注入；
- 地理位置就近策略。

对前端和 Node.js 候选人，面试官常会问 Cloudflare Workers、Vercel Edge Functions、CDN 边缘鉴权等实际应用。

### 22. REST、GraphQL、gRPC 如何选型？

| 维度 | REST | GraphQL | gRPC |
|---|---|---|---|
| 资源表达 | 强 | 中 | 弱 |
| 获取字段灵活性 | 低 | 高 | 中 |
| 传输格式 | 常见 JSON | JSON | Protobuf |
| 浏览器友好 | 高 | 高 | 原生较弱 |
| 类型约束 | 中 | 强 Schema | 很强 |
| 内部服务通信 | 常见 | 较少 | 很常见 |

**选择建议：**

- 对外开放 API：REST 最稳；
- 前端定制化字段多、聚合需求强：GraphQL；
- 微服务高性能内部通信：gRPC。

### 23. 负载均衡算法有哪些？

高频算法：

- 轮询（round-robin）
- 加权轮询（weighted round-robin）
- 最少连接（least connections）
- IP hash
- 一致性哈希（consistent hashing）

**一致性哈希为什么重要？**

当节点增减时，只影响少量 key 重映射，适合缓存集群、分布式 KV、会话路由。

### 24. CORS 是什么？为什么浏览器需要它？

CORS（Cross-Origin Resource Sharing）是浏览器上的安全机制，用于在**同源策略（same-origin policy）**下控制跨域请求。

同源要求协议、域名、端口都相同。  
注意：**CORS 是浏览器限制，不是服务端限制。**

### 25. 什么是预检请求（Preflight Request）？

当请求不是“简单请求”时，浏览器会先发一个 `OPTIONS`：

- 问服务端是否允许这个 Origin；
- 是否允许这些方法和自定义头；
- 是否允许携带凭证。

常见响应头：

```http
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

### 26. 携带 Cookie 的跨域请求为什么不能把 `Allow-Origin` 写成 `*`？

因为带凭证（credentials）时，规范要求 `Access-Control-Allow-Origin` 必须是明确域名，不能通配。  
这是 CORS 非常高频的细节题。

### 27. 从输入 URL 到页面展示，中间发生了什么？

这是网络综合题，建议按链路回答：

1. URL 解析；
2. DNS 解析；
3. 建立 TCP/QUIC 连接；
4. TLS 握手；
5. 发送 HTTP 请求；
6. 经过 CDN、LB、网关；
7. 应用服务处理，访问缓存/数据库；
8. 返回 HTTP 响应；
9. 浏览器解析 HTML/CSS/JS；
10. 构建 DOM/CSSOM、布局、绘制；
11. 资源继续并发加载与执行。

---

## 二、操作系统部分高频面试题

### 28. 进程、线程、协程有什么区别？

| 项目 | 进程（Process） | 线程（Thread） | 协程（Coroutine） |
|---|---|---|---|
| 资源隔离 | 强 | 弱，共享进程内资源 | 更弱，通常用户态调度 |
| 切换开销 | 大 | 中 | 小 |
| 调度者 | OS | OS | 运行时 / 用户态框架 |
| 崩溃影响 | 通常隔离 | 可能影响整个进程 | 通常影响所在线程/作用域 |

ASCII 图可这样讲：

```text
Process A
├─ Thread 1
│  ├─ Coroutine a
│  ├─ Coroutine b
├─ Thread 2
│  ├─ Coroutine c
└─ Heap / Code / FDs shared by threads
```

对 Kotlin 候选人，可补充 `suspend`、结构化并发（structured concurrency）；对 Node.js 候选人，可补充 event loop + promise 不是线程。

### 29. 上下文切换（Context Switch）发生了什么？

上下文切换时，系统通常需要：

- 保存当前执行现场：寄存器、程序计数器、栈指针；
- 切换地址空间或页表（进程切换时）；
- 刷新或影响缓存/TLB；
- 恢复下一个执行单元的上下文。

所以切换不是免费操作。  
面试里如果问“为什么线程太多反而变慢”，就要提上下文切换、锁竞争、缓存失效。

### 30. 死锁的四个必要条件是什么？

死锁（deadlock）四个必要条件：

1. 互斥（mutual exclusion）
2. 请求并保持（hold and wait）
3. 不可剥夺（no preemption）
4. 循环等待（circular wait）

破坏任一条件即可预防死锁。

### 31. 如何预防、避免、检测死锁？

**预防**：从制度上打断必要条件，例如固定加锁顺序。  
**避免**：如银行家算法（Banker’s Algorithm），分配前判断是否仍处安全状态。  
**检测**：允许死锁发生，再通过资源分配图、超时、监控检测并恢复。  

工程实践中，最常见的是：

- 统一锁顺序；
- 尽量减少锁持有时间；
- 使用超时；
- 避免嵌套锁；
- 对数据库事务设置合理隔离与重试策略。

### 32. 什么是银行家算法？

银行家算法的核心思想是：  
系统分配资源前，先模拟分配结果，检查是否仍存在一个“安全序列（safe sequence）”，让所有进程最终都能完成。如果没有，就拒绝这次分配。

面试中不一定要求手写，但你至少要知道它是“死锁避免”而不是“死锁检测”。

### 33. 虚拟内存是什么？

虚拟内存（virtual memory）让每个进程看到独立、连续的地址空间，背后由操作系统和 MMU（Memory Management Unit）映射到物理内存。

好处：

- 进程隔离；
- 便于内存管理；
- 可按需加载；
- 支持交换（swap）。

### 34. 分页（Paging）与 TLB 是什么？

分页把虚拟地址和物理地址都分成固定大小页。访问时通过页表完成地址转换。  
TLB（Translation Lookaside Buffer）是页表映射的高速缓存，用于降低地址转换开销。

**面试加分表达**：

- 没命中 TLB 时需要查页表，代价更高；
- 若页也不在内存中，还会触发缺页异常（page fault）。

### 35. 页面置换算法为什么常提 LRU？

当物理内存不足时，要把某些页换出。  
LRU（Least Recently Used）假设“最近最少使用”的页未来也更不可能被访问。它是很经典的近似局部性原则。

面试时可以说：

- 理想 LRU 很难完全低成本实现；
- 现实系统常用近似算法，如 Clock。

### 36. 常见 I/O 模型有哪些？

高频对比：

| 模型 | 特点 |
|---|---|
| 阻塞 I/O（blocking I/O） | 调用后一直等 |
| 非阻塞 I/O（non-blocking I/O） | 立即返回，需要轮询 |
| I/O 多路复用（I/O multiplexing） | 用一个线程监听多个 FD |
| 异步 I/O（async I/O） | 内核完成后主动通知 |

### 37. select、poll、epoll 有什么区别？

**select**

- FD 数量上限较低；
- 每次都要复制/遍历 FD 集合。

**poll**

- 去掉固定上限；
- 但仍需线性扫描。

**epoll**

- 事件驱动；
- 通过回调式就绪通知减少无效扫描；
- Linux 高并发网络编程常用。

面试追问时可以补充：

- `epoll` 有 LT（level-triggered）和 ET（edge-triggered）两种模式；
- Node.js、Nginx、Redis 等都大量利用多路复用思想。

### 38. 阻塞、非阻塞、同步、异步这几个概念怎么区分？

很多候选人会混淆。简单记：

- **阻塞/非阻塞**：调用返回前，线程是否被卡住；
- **同步/异步**：结果完成后，谁来通知你。

例如：

- `read()` 一直等：阻塞同步；
- 非阻塞 socket + 轮询：非阻塞同步；
- `epoll`：同步多路复用；
- 真正的 async I/O：异步。

### 39. Node.js 为什么单线程还能处理高并发？

因为“JavaScript 执行线程单线程”不等于“整个系统单线程”。  
Node.js 依赖：

- event loop；
- 非阻塞 I/O；
- 内核多路复用；
- libuv 线程池处理部分阻塞任务。

所以高并发来自**不阻塞主线程等待 I/O**，而不是 CPU 计算 magically 变快。

### 40. Kotlin 协程为什么轻量？

因为协程切换通常发生在用户态，不需要像线程那样由 OS 完整调度；同时挂起点只保存必要状态。  
但要注意：协程轻量不代表 CPU 任务就能无限开；如果全是计算密集，仍然受核心数约束。

---

## 三、Linux 排障命令：面试与实战都常考

### 41. 进程排查常用哪些命令？

#### `ps`

```bash
ps -ef | grep java
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

用途：静态查看进程列表、父子关系、资源占用。

#### `top` / `htop`

- 看实时 CPU、内存、负载；
- 定位哪类线程或进程异常；
- `htop` 交互性更好。

#### `strace`

```bash
strace -p 12345
```

用途：追踪系统调用（system call），排查进程卡在 `futex`、`read`、`connect`、`epoll_wait` 等位置。

#### `lsof`

```bash
lsof -p 12345
lsof -i :8080
```

用途：看进程打开了哪些文件、端口、socket。

### 42. 网络排查常用命令有哪些？

#### `netstat` / `ss`

```bash
ss -ltnp
ss -ant | grep 443
```

看监听端口、连接状态、`ESTABLISHED`、`TIME_WAIT`、`CLOSE_WAIT`。

#### `tcpdump`

```bash
tcpdump -i eth0 port 443
```

抓包分析握手、重传、RST、TLS 行为。

#### `curl`

```bash
curl -I https://example.com
curl -v https://example.com/api
curl --resolve example.com:443:1.2.3.4 https://example.com
```

验证 HTTP 状态码、重定向、TLS、Header、域名解析覆盖。

#### `nslookup`

```bash
nslookup example.com
```

快速看 DNS 解析结果。面试中你也可提 `dig` 更常用。

### 43. 磁盘与日志排查命令会怎么答？

#### `df`

```bash
df -h
```

看文件系统空间。

#### `du`

```bash
du -sh *
du -sh /var/log/*
```

看目录占用。

#### `iostat`

```bash
iostat -xz 1
```

看磁盘 I/O 饱和度、队列、await。

#### `tail`

```bash
tail -f app.log
```

实时追日志。

#### `grep`

```bash
grep "ERROR" app.log | tail -100
```

筛选关键字。

#### `awk`

```bash
awk '{print $1, $9}' access.log | head
```

按列处理日志。

#### `sed`

```bash
sed -n '1,20p' app.log
```

按范围查看、替换文本。

### 44. 线上服务慢了，你会怎么排查？

这是综合题，建议按层次回答：

1. **先看现象**：超时、错误率、P99 延迟、局部还是全局。
2. **看资源**：CPU、内存、负载、磁盘、网络。
3. **看连接**：是否大量 `TIME_WAIT`、`CLOSE_WAIT`、连接池耗尽。
4. **看日志**：错误码、超时链路、重试风暴。
5. **看依赖**：DB、Redis、MQ、第三方 API。
6. **看变更**：最近发布、配置变更、证书到期、流量突增。
7. **必要时抓样本**：`strace`、`tcpdump`、火焰图、JFR、pprof。

### 45. 什么是 `CLOSE_WAIT` 很多？通常意味着什么？

`CLOSE_WAIT` 表示对端已经关闭连接，本端还没正确 close。  
如果大量堆积，常见原因：

- 应用代码泄漏连接；
- 某些异常分支没有关闭 socket；
- 上层连接池或框架处理不当。

这道题很适合体现“你不是只会背协议，而是真做过排障”。

### 46. CPU 飙高但流量没涨，怎么分析？

可能方向：

- 死循环；
- 热点锁自旋；
- 日志爆量；
- GC 频繁；
- 反序列化/正则回溯；
- 重试风暴；
- 某个批任务异常。

答题时最好结合工具：

- `top -H -p <pid>` 找热点线程；
- `jstack` / 火焰图定位 Java；
- Node 看 event loop lag、CPU profile；
- Android native 看 systrace / perf。

---

## 四、面试官最喜欢追问的综合题

### 47. 为什么说“网络问题”和“系统问题”往往是同一件事？

因为线上问题常跨层：  
一个慢请求，可能表面看是 API 超时，实际是：

- DNS 抖动；
- TLS 握手失败重试；
- 连接池耗尽；
- TCP 重传；
- 线程池阻塞；
- 磁盘打满导致日志卡住；
- 下游服务雪崩。

会把这些层次串起来讲的候选人，通常明显强于只会背 RFC 的人。

### 48. 如何把网络题答成工程题？

推荐模板：

1. **定义**：是什么；
2. **目的**：为了解决什么问题；
3. **机制**：它如何工作；
4. **权衡**：优点、缺点、代价；
5. **场景**：在哪些真实系统里用到；
6. **排障**：出了问题怎么观察与验证。

例如答 HTTP/3 时，不要只说“更快”，要说：

- 为什么更快；
- 在什么链路更明显；
- 它如何与 CDN、移动网络、连接迁移结合；
- 是否所有业务都值得切。

## 本章要点

- 网络题要抓住“解决了什么问题”而不是只背协议名词。
- HTTP/2 的核心是二进制分帧、多路复用、头部压缩；HTTP/3 的核心是 QUIC。
- HTTPS 必须会 TLS 1.3 握手、证书链、HSTS。
- TCP 高频点是三次握手、四次挥手、流量控制、拥塞控制、TIME_WAIT。
- WebSocket、DNS、CDN、CORS 都是前后端与后端面试的交叉高频。
- 操作系统重点是进程/线程/协程、上下文切换、死锁、虚拟内存、I/O 模型。
- Linux 命令不是背诵题，关键在于你知道“何时用、看什么、如何定位问题”。

## 延伸阅读

- RFC 9110 / 9113 / 9114（HTTP 语义、HTTP/2、HTTP/3）
- RFC 8446（TLS 1.3）
- 《TCP/IP 详解 卷一》
- 《深入理解计算机系统》
- Brendan Gregg 关于性能分析、火焰图与 Linux 可观测性的文章与演讲
