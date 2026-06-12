# 第十一章：Node.js 核心原理

Node.js 面试里最容易拉开差距的，不是会不会写 `Express`，而是你能不能把 **事件循环（Event Loop）**、**libuv**、**V8**、**模块系统（Module System）**、**流（Stream）**、**Worker Threads** 与 **Cluster** 串成一张完整的运行时地图。下面按高频面试题展开。

```text
+---------------------- Application ----------------------+
| Express / Fastify / NestJS / Custom Business Logic      |
+------------------------- Node.js -----------------------+
| Timers | Microtasks | Module Loader | Buffer | Stream   |
+------------------------- libuv -------------------------+
| Event Loop | Thread Pool | TCP/UDP | FS | DNS | Crypto  |
+-------------------------- OS ---------------------------+
| epoll(Linux) | kqueue(BSD/macOS) | IOCP(Windows)        |
+--------------------------- V8 --------------------------+
| Ignition | TurboFan | Heap | GC | Hidden Class | IC     |
+---------------------------------------------------------+
```

## 11.1 事件循环（Event Loop）

### 面试题 1
**题目**：Node.js 为什么能用单线程处理大量并发连接？

**考察要点**：事件驱动、非阻塞 I/O、事件循环与线程池分工。

**参考答案**：Node.js 的 JavaScript 执行主线程通常只有一个，但它并不是“所有工作都在一个线程里完成”。真正的关键是：JavaScript 主线程只负责执行回调与业务逻辑；I/O 监听由 libuv 和操作系统完成；部分无法直接走 OS 异步接口的任务会被投递到线程池。这样主线程不必同步等待磁盘、网络、加解密等耗时操作，因此即使只有一个 JS 主线程，也能维持大量并发连接。面试中最好补一句：Node.js 擅长 **I/O 密集型（I/O-bound）** 场景，不擅长长时间 CPU 密集型计算。

### 面试题 2
**题目**：请详细说明 Node.js 事件循环的 6 个阶段。

**考察要点**：timers、pending callbacks、idle/prepare、poll、check、close callbacks。

**参考答案**：Node.js 的事件循环可以简化为六个阶段：  
1. **timers**：执行到期的 `setTimeout()`、`setInterval()` 回调。  
2. **pending callbacks**：执行某些系统级回调，例如部分 TCP 错误回调。  
3. **idle/prepare**：Node.js 内部使用，开发者通常不直接介入。  
4. **poll**：获取新的 I/O 事件，并执行与 I/O 相关的回调；如果队列为空，会在这里等待。  
5. **check**：执行 `setImmediate()` 回调。  
6. **close callbacks**：执行如 `socket.on('close')` 这类关闭回调。  
常见误区是把事件循环仅理解成“定时器队列”。实际上它是 libuv 调度 I/O、定时器和回调的总控结构。

### 面试题 3
**题目**：每个阶段里到底会执行什么？

**考察要点**：能否把抽象阶段和具体 API 对应起来。

**参考答案**：`setTimeout`/`setInterval` 在 timers；某些延迟到下一轮处理的系统回调在 pending callbacks；poll 阶段最重要，文件、网络等大多数 I/O 回调在这里被取出；`setImmediate` 在 check；连接关闭类回调在 close callbacks。`process.nextTick()` 与 `Promise.then()` 不属于六阶段本身，它们属于更高优先级的微任务调度机制，会在阶段切换间隙被清空。

### 面试题 4
**题目**：`setTimeout(fn, 0)` 为什么不是立刻执行？

**考察要点**：最小延迟、事件循环时机、poll 阶段影响。

**参考答案**：`setTimeout(fn, 0)` 表示“最早在 0ms 后进入可执行状态”，不等于马上运行。它至少要等当前调用栈清空，并等事件循环回到 timers 阶段。若前面还有大量同步代码、`nextTick`、Promise 微任务，或 poll 阶段里还有 I/O 回调未处理，它都要继续等待。因此定时器是“到期可调度”，不是“到期立即执行”。

### 面试题 5
**题目**：`process.nextTick()`、`Promise.then()`、`setImmediate()`、`setTimeout(0)` 的执行顺序如何比较？

**考察要点**：nextTick 队列、微任务队列、check 阶段、timers 阶段。

**参考答案**：通常优先级可以概括为：当前同步代码执行完后，先清空 `process.nextTick()` 队列，再清空 Promise 微任务队列，然后事件循环继续推进；`setTimeout(0)` 在 timers 阶段，`setImmediate()` 在 check 阶段。若它们都从顶层脚本注册，`setTimeout(0)` 和 `setImmediate()` 的先后并不绝对；但如果在 I/O 回调内部注册，通常 `setImmediate()` 会先于 `setTimeout(0)`。

```ts
import fs from 'node:fs';

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  Promise.resolve().then(() => console.log('promise'));
  process.nextTick(() => console.log('nextTick'));
});

// 常见输出：
// nextTick
// promise
// immediate
// timeout
```

### 面试题 6
**题目**：`process.nextTick()` 和 Promise 微任务有什么差异？

**考察要点**：nextTick queue 优先级更高、饥饿风险。

**参考答案**：两者都会在当前阶段结束后尽快执行，但 `process.nextTick()` 队列优先级高于 Promise 微任务。Node.js 会在每次阶段切换前先清空 nextTick，再清空 Promise 微任务。所以如果递归地塞入大量 `nextTick`，连 Promise 回调和 I/O 都可能被饿死。面试里建议说：除非需要兼容 Node 内部风格或非常明确的时序控制，否则业务代码更建议优先使用 Promise 微任务。

### 面试题 7
**题目**：什么是微任务（Microtask）和宏任务（Macrotask）？

**考察要点**：分类与 Node.js 语境中的差异。

**参考答案**：微任务通常指在当前执行单元结束后、事件循环进入下一阶段前需要立即清空的任务，例如 Promise 回调；Node.js 还额外有 `process.nextTick()` 队列。宏任务是更宽泛的事件循环任务，比如定时器、I/O、`setImmediate()`。浏览器面试喜欢问“微任务/宏任务”，Node.js 面试更喜欢追问“Node 的 nextTick 为什么甚至比 Promise 更早”。

### 面试题 8
**题目**：什么是事件循环饥饿（Event Loop Starvation）？

**考察要点**：过量同步任务、递归 nextTick、长计算。

**参考答案**：事件循环饥饿是指某类任务长期占据主线程，导致其他回调得不到执行。例如：  
- 大量同步 `while` 循环；  
- 递归 `process.nextTick()`；  
- Promise 链中塞入超长 CPU 计算；  
- 单次 JSON 大对象序列化/反序列化。  
结果是请求超时、健康检查失败、延迟飙升。解决思路包括：切片计算、改用 Worker Threads、把 I/O 结果流式处理而不是一次性载入。

### 面试题 9
**题目**：如何解释 poll 阶段是事件循环的“核心”？

**考察要点**：I/O 拉取、阻塞等待、与 timers/check 的关系。

**参考答案**：poll 阶段负责取回 I/O 事件并执行相关回调，还决定事件循环是继续忙碌还是等待。若 poll 队列不为空，Node 会持续处理；若为空，它会判断是否有 `setImmediate()`，有则转向 check，没有则可能等待新的 I/O 到来。这就是为什么在 I/O 回调里注册 `setImmediate()` 常常先执行：因为 poll 结束后就直接进入 check。

### 面试题 10
**题目**：如何用代码验证事件循环顺序？

**考察要点**：通过最小实验解释机制。

**参考答案**：可以组合同步代码、微任务、定时器与 I/O，观察输出顺序。面试时不要只背答案，最好说明“顺序依赖注册位置”。例如顶层脚本与 I/O 回调内部的注册结果不完全一样。

```ts
console.log('A');
setTimeout(() => console.log('B'), 0);
setImmediate(() => console.log('C'));
Promise.resolve().then(() => console.log('D'));
process.nextTick(() => console.log('E'));
console.log('F');

// 常见：A F E D B/C
// B 与 C 在顶层场景下不保证绝对顺序
```

## 11.2 libuv 与底层 I/O

### 面试题 11
**题目**：libuv 在 Node.js 中扮演什么角色？

**考察要点**：跨平台异步 I/O 抽象层。

**参考答案**：libuv 是 Node.js 的跨平台异步 I/O 库，负责事件循环、线程池、定时器、异步文件系统、网络 I/O 封装等。可以把它理解成“Node 运行时调度内核”。JavaScript 代码调用 Node API 后，很多实际调度工作并不在 V8 里完成，而是在 libuv 中完成。

### 面试题 12
**题目**：Node.js 线程池默认多大？可以调整吗？

**考察要点**：默认 4，`UV_THREADPOOL_SIZE`。

**参考答案**：libuv 线程池默认大小是 4，可以通过环境变量 `UV_THREADPOOL_SIZE` 调整，最大一般是 128。线程池并不是越大越好，过大可能导致上下文切换与内存占用增加。实际调优要看任务类型，比如大量 `crypto.pbkdf2`、`fs.readFile` 或 `zlib` 压缩时可以适当扩容。

### 面试题 13
**题目**：哪些操作会走线程池，哪些直接使用操作系统异步能力？

**考察要点**：fs、dns、crypto、zlib vs TCP/UDP/pipes。

**参考答案**：通常 `fs`、部分 `dns`、`crypto`、`zlib` 会走 libuv 线程池，因为这些操作在不少平台上没有统一高效的原生异步接口；而 TCP、UDP、管道（pipes）这类网络 I/O 更多依赖操作系统事件通知机制，如 epoll、kqueue、IOCP。高频误区是“Node.js 所有异步都靠线程池”，这是错误的。

### 面试题 14
**题目**：请解释 `epoll`、`kqueue`、`IOCP` 的区别。

**考察要点**：平台差异与统一抽象。

**参考答案**：`epoll` 是 Linux 下高性能 I/O 多路复用机制，`kqueue` 主要用于 BSD/macOS，`IOCP` 是 Windows 的完成端口模型。Node.js 开发者通常不直接使用这些 API，但要知道 libuv 把它们抽象成统一接口，所以你写的网络代码大体能跨平台运行。面试里如果被追问，强调“底层实现不同，上层事件驱动接口一致”即可。

## 11.3 V8 引擎（V8 Engine）

### 面试题 15
**题目**：V8 是如何执行 JavaScript 的？

**考察要点**：解析、字节码、解释执行、JIT 优化。

**参考答案**：V8 会先把 JavaScript 源码解析成抽象语法树，再生成字节码，由 **Ignition** 解释器执行。热点代码被识别后，会交给 **TurboFan** 做即时编译（JIT, Just-In-Time Compilation）优化，生成更高效的机器码。因此 V8 不是“一上来全部编译成机器码”，而是先解释执行，再对热点路径优化。

### 面试题 16
**题目**：什么情况下 V8 会去优化代码？又为什么会反优化（deopt）？

**考察要点**：热点函数、类型稳定性、隐藏类变化。

**参考答案**：当某段函数被频繁调用，V8 会把它当作热点代码交给 TurboFan 优化。优化依赖“类型与对象形状比较稳定”的假设。如果一个对象一会儿有 `name` 字段，一会儿又动态添加 `age`、删除 `name`，隐藏类频繁变化，优化假设被打破，就可能触发 deopt，导致性能回退。面试中可总结为：写稳定形状的数据结构，比炫技式动态对象更利于 V8。

### 面试题 17
**题目**：V8 堆内存有哪些主要区域？

**考察要点**：new space、old space、large object space、code space。

**参考答案**：常见要记住：  
- **New Space**：新生代对象，生命周期短。  
- **Old Space**：老生代对象，存活时间长。  
- **Large Object Space**：大对象，通常不会频繁复制。  
- **Code Space**：存放已编译代码。  
如果对象在新生代中多次 GC 后仍存活，就会晋升到老生代。理解这点有助于分析“为什么某些缓存对象导致老生代持续膨胀”。

### 面试题 18
**题目**：Minor GC 和 Major GC 的区别是什么？

**考察要点**：Scavenge、Mark-Sweep-Compact。

**参考答案**：**Minor GC** 主要针对新生代，常见算法是 **Scavenge**，速度快，适合回收大量短命对象；**Major GC** 主要针对老生代，通常涉及 **Mark-Sweep-Compact**，成本更高，停顿也更明显。Node.js 服务若频繁 Full GC，往往意味着老生代压力大，需要检查缓存、闭包引用、监听器泄漏或大对象保留。

### 面试题 19
**题目**：什么是增量标记（Incremental Marking）？

**考察要点**：降低停顿时间。

**参考答案**：增量标记是把一次完整的标记过程拆成多个小步骤，与 JavaScript 执行交替进行，减少单次长暂停。它不能消灭 GC 停顿，但能显著改善尾延迟。对后端服务来说，GC 不仅是吞吐问题，更是 P99/P999 延迟问题。

### 面试题 20
**题目**：什么是隐藏类（Hidden Classes）？

**考察要点**：对象形状与属性访问优化。

**参考答案**：虽然 JavaScript 是动态语言，但 V8 会为具有相同属性布局的对象建立隐藏类，用来加速属性访问。如果同一批对象都按相同顺序初始化相同字段，V8 就更容易优化。反之，频繁动态增删属性会破坏隐藏类稳定性。

```ts
class User {
  name: string;
  age: number;
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}

// 比起先创建空对象再不断追加字段，更利于 V8 优化
```

### 面试题 21
**题目**：什么是内联缓存（Inline Caching, IC）？

**考察要点**：缓存属性访问路径。

**参考答案**：内联缓存会记住某个访问点过去看到的对象形状与查找路径。比如 `user.name` 多次访问同类对象时，V8 可以跳过很多通用查找逻辑，直接按缓存路径读取属性。隐藏类稳定，IC 命中率就高；对象形状混乱，IC 容易退化。

## 11.4 模块系统（Module System）

### 面试题 22
**题目**：CommonJS 和 ESM 有哪些核心差异？

**考察要点**：`require` vs `import`、加载时机、导出绑定、顶层 await。

**参考答案**：CommonJS 使用 `require` 和 `module.exports`，本质是运行时加载，导出是值拷贝语义的表现；ESM 使用 `import`/`export`，在语法层面静态可分析，导出是实时绑定（live binding）。ESM 支持顶层 `await`，更适合现代工具链与 Tree Shaking。Node.js 中两者还能共存，但互操作细节很多，比如默认导出映射、`type: "module"`、扩展名要求等。

### 面试题 23
**题目**：Node.js 的模块解析算法大致是怎样的？

**考察要点**：相对路径、包名、`package.json`、exports/main。

**参考答案**：当你 `require('./x')` 或 `import './x.js'` 时，Node 会先按相对/绝对路径查找；若是包名，则去 `node_modules` 中查找包目录，再读取 `package.json` 的 `exports`、`main`、`type` 等字段决定入口。ESM 的解析通常比 CommonJS 更严格，例如很多场景下要求显式扩展名。面试里回答时要强调：现代包优先考虑 `exports` 映射，而不是只背 `main`。

### 面试题 24
**题目**：Node.js 如何处理循环依赖（Circular Dependencies）？

**考察要点**：部分导出、缓存、初始化顺序。

**参考答案**：当模块 A 依赖 B，B 又依赖 A 时，Node 不会无限递归，而是利用模块缓存返回“当前已初始化的部分导出”。因此你可能拿到的是未完全初始化的对象。CommonJS 和 ESM 都能处理循环依赖，但 ESM 因为是实时绑定，语义更清晰一些。设计上应尽量避免深层循环依赖，可通过提取公共模块解决。

### 面试题 25
**题目**：什么是顶层 `await`（Top-level Await）？适合什么场景？

**考察要点**：仅 ESM 支持、模块初始化顺序。

**参考答案**：顶层 `await` 允许在 ESM 模块最外层直接等待异步操作，例如加载配置、初始化数据库连接。但它会影响依赖图中下游模块的执行时机，因此要谨慎使用。适合“模块初始化必须依赖异步结果”的场景，不适合随手把所有模块都改成顶层等待。

## 11.5 Buffer 与 Stream

### 面试题 26
**题目**：为什么 Node.js 需要 Buffer，而不是直接用字符串？

**考察要点**：二进制数据、网络协议、文件处理。

**参考答案**：字符串适合文本，不适合任意二进制字节序列。Node.js 作为后端运行时，经常要处理文件、TCP、压缩包、图片、音视频等二进制内容，因此需要 `Buffer`。`Buffer` 底层与 `ArrayBuffer` 相关，并针对 I/O 场景做了优化。

### 面试题 27
**题目**：Buffer 内部为什么会有池化（pooling）？

**考察要点**：小对象频繁分配的性能成本。

**参考答案**：如果每次读写几个字节都单独向系统申请内存，成本会很高。Node.js 对小 Buffer 分配采用池化思路，从预先分配的大块内存中切片，提高分配效率、降低碎片化。面试时可顺带说：大 Buffer 和小 Buffer 的分配路径不完全一样。

### 面试题 28
**题目**：流有哪四种类型？

**考察要点**：Readable、Writable、Duplex、Transform。

**参考答案**：  
- **Readable**：可读流，如文件读取、HTTP 请求体。  
- **Writable**：可写流，如文件写入、HTTP 响应。  
- **Duplex**：双工流，可读可写但两边独立，例如 TCP socket。  
- **Transform**：转换流，是一种特殊双工流，输出依赖输入，例如 gzip、JSON 行解析。  
这四类在面试里很常见，最好能结合业务举例。

### 面试题 29
**题目**：什么是背压（Backpressure）？为什么重要？

**考察要点**：生产速度大于消费速度、内存暴涨风险。

**参考答案**：背压是指上游生产数据的速度超过下游消费能力。如果不控制，Readable 会不断推送数据，Writable 来不及处理，内存缓存不断堆积。Node.js 的流设计核心目标之一就是把背压内建到 API 中，避免一次性把大文件全读进内存。

### 面试题 30
**题目**：`pipe()` 是如何处理背压的？

**考察要点**：`write()` 返回值、暂停与恢复。

**参考答案**：当 `writable.write(chunk)` 返回 `false`，表示内部缓冲已接近上限，上游 Readable 会被暂停；等下游触发 `drain` 事件后，再恢复读取。`pipe()` 已经帮你处理好了这套暂停/恢复机制，所以绝大多数顺序流转场景优先用 `pipe()` 或 `pipeline()`。

### 面试题 31
**题目**：如何手动处理背压？

**考察要点**：`highWaterMark`、`drain`、流控。

**参考答案**：如果你不用 `pipe()`，就要自己检查 `write()` 的返回值，并在 `drain` 后继续写。`highWaterMark` 决定内部缓冲阈值，但它不是“最大内存上限”，只是触发背压的参考值。

```ts
import fs from 'node:fs';

const rs = fs.createReadStream('input.log', { highWaterMark: 64 * 1024 });
const ws = fs.createWriteStream('output.log');

rs.on('data', (chunk) => {
  const canContinue = ws.write(chunk);
  if (!canContinue) {
    rs.pause();
    ws.once('drain', () => rs.resume());
  }
});
```

### 面试题 32
**题目**：如何实现自定义 Transform 流？

**考察要点**：继承 `Transform`、实现 `_transform`。

**参考答案**：自定义 Transform 的关键是实现 `_transform(chunk, encoding, callback)`，把输入转换后推送给下游。适合日志脱敏、CSV 转 JSON、协议适配等场景。

```ts
import { Transform, pipeline } from 'node:stream';
import { promisify } from 'node:util';

const pipe = promisify(pipeline);

class UppercaseTransform extends Transform {
  _transform(chunk: Buffer, _encoding: BufferEncoding, callback: (err?: Error | null, data?: Buffer) => void) {
    callback(null, Buffer.from(chunk.toString().toUpperCase()));
  }
}
```

### 面试题 33
**题目**：为什么推荐 `pipeline()` 而不是手写多级 `pipe()`？

**考察要点**：统一错误处理与资源清理。

**参考答案**：`pipeline()` 会把多个流串起来，并在任意一环出错时统一销毁上下游资源，避免文件句柄、socket 或内存泄漏。手写 `a.pipe(b).pipe(c)` 看起来简洁，但错误处理容易遗漏。在生产环境里，`pipeline()` 更稳妥。

## 11.6 Worker Threads 与 Cluster

### 面试题 34
**题目**：`worker_threads`、`child_process`、`cluster` 有什么区别？

**考察要点**：线程 vs 进程、共享内存、隔离性。

**参考答案**：`worker_threads` 是同进程内多线程，适合 CPU 密集型任务，通信开销相对低，还支持 `SharedArrayBuffer` 共享内存；`child_process` 是创建独立进程，隔离性好，适合执行外部命令或需要彻底隔离的任务；`cluster` 本质上是多进程模型，多个 worker 共享同一服务端口，主要用于提高 Node HTTP 服务的多核利用率。

### 面试题 35
**题目**：什么场景应该用 Worker Threads，而不是 libuv 线程池？

**考察要点**：自定义 CPU 计算 vs 内置异步 API。

**参考答案**：如果任务已经由 Node 内置 API 封装且会自动走线程池，例如 `crypto.pbkdf2`、`zlib.gzip`、`fs.readFile`，一般不需要自己再包一层 Worker。若是你自己的 CPU 密集计算，比如图像处理、复杂规则引擎、报表聚合、密码学之外的大量数学计算，则更适合 Worker Threads。

### 面试题 36
**题目**：`SharedArrayBuffer` 和 `Atomics` 有什么用？

**考察要点**：线程间共享内存与同步。

**参考答案**：普通 `postMessage` 默认是消息传递模型，而 `SharedArrayBuffer` 允许多个线程访问同一块内存。为了避免竞态，需要配合 `Atomics` 做原子操作与同步。它适合极致性能场景，但复杂度高，业务系统中大多数场景仍优先选择消息通信。

### 面试题 37
**题目**：`MessageChannel` / `MessagePort` 适合做什么？

**考察要点**：结构化通信、端口转移。

**参考答案**：`MessageChannel` 会创建一对互联端口，适合在线程之间建立更清晰的通信边界。例如一个主线程管理多个 worker 时，可以把某个 `MessagePort` 转移给特定 worker，实现专用控制通道。相比把所有消息都塞在同一个 `parentPort`，可维护性更高。

### 面试题 38
**题目**：如何设计 Worker Pool？

**考察要点**：预热、任务队列、复用、超时。

**参考答案**：不要每来一个任务就新建一个 Worker，因为线程启动也有成本。更合理的是预创建固定数量 Worker，维护任务队列，空闲时分配任务，超时时回收，必要时记录执行时间和失败率。Worker Pool 的本质和数据库连接池很像：通过复用减少昂贵初始化成本。

```ts
// 简化示意
type Job = { id: string; payload: number[] };
// 真实场景应包含队列、超时、健康检查与失败重试
```

### 面试题 39
**题目**：Cluster 的主从架构（master-worker architecture）是怎样的？

**考察要点**：主进程调度、多个 worker 共享端口。

**参考答案**：Cluster 会启动一个主进程和多个 worker 进程。主进程负责派生 worker、监听退出事件、分发连接或协调共享监听套接字；worker 负责真正处理 HTTP 请求。这样能让 Node 在多核 CPU 上跑多个进程，提升吞吐并降低单进程故障影响。

### 面试题 40
**题目**：Cluster 的负载均衡有哪些模式？

**考察要点**：round-robin vs OS-level。

**参考答案**：常见有两种思路：  
1. **Round-robin**：主进程把连接轮询分给 worker。  
2. **OS-level**：由多个 worker 共享监听 socket，操作系统负责调度。  
Node 默认策略在不同平台和版本中有过演进。面试中回答重点不是背实现细节，而是说明“连接分发可能由 Node 主进程做，也可能更多依赖 OS”。

### 面试题 41
**题目**：如何做优雅重启（Graceful Restart）？

**考察要点**：新 worker 先起、旧 worker 停止接新流量、等待存量请求完成。

**参考答案**：优雅重启的思路是：先拉起新 worker，确认健康后再让旧 worker 停止接收新连接；对于已有连接，给一定超时时间处理完成，然后再退出。这样能避免发布时请求中断。若服务有长连接，还要配合连接排空（connection draining）策略。

### 面试题 42
**题目**：为什么 WebSocket 场景常常需要 Sticky Session？

**考察要点**：长连接与状态粘性。

**参考答案**：HTTP 短请求可以随便轮询分发，但 WebSocket 建立后是长连接，后续消息必须落到持有该连接的同一实例，否则找不到对应 socket。因此多实例部署常需要 **粘性会话（Sticky Session）**，或者借助 Redis Adapter、消息总线在节点间广播，把连接状态做外部协调。

```ts
import cluster from 'node:cluster';
import os from 'node:os';
import http from 'node:http';

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on('exit', () => cluster.fork());
} else {
  http.createServer((_req, res) => {
    res.end(`worker ${process.pid}`);
  }).listen(3000);
}
```

## 本章要点

- 事件循环六阶段必须能结合 API 说清楚，而不是只背名字。  
- `process.nextTick()` 优先级高于 Promise 微任务，但滥用会造成饥饿。  
- libuv 不是“线程池本身”，而是 Node.js 的跨平台调度内核。  
- V8 的性能关键字是：Ignition、TurboFan、Hidden Classes、Inline Caching、GC 分代。  
- CommonJS 与 ESM 的差异，面试重点是加载模型、静态分析与互操作。  
- Buffer/Stream 的核心是二进制处理与背压控制。  
- CPU 密集任务优先考虑 Worker Threads；多核服务扩展可考虑 Cluster。  

## 延伸阅读

- Node.js 官方文档：Event Loop、Streams、Worker Threads、Cluster  
- libuv 官方文档  
- V8 Blog：Garbage Collection、TurboFan、Hidden Classes  
- “You Don’t Know JS Yet” 中关于异步与作用域的章节  
