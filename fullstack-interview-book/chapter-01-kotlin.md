# 第一章：Kotlin 语言深度面试

本章整理 Kotlin 协程（Coroutine）、流（Flow）、通道（Channel）等高频面试题。

## 1. 协程基础与调度

### 1.
**题目**：`launch` 和 `async` 的区别是什么？

**考察要点**：返回值、异常传播、适用场景、结构化并发。

**参考答案**：`launch` 用于“只关心任务完成，不关心返回值”的场景，返回 `Job`；`async` 用于“需要结果”的并发任务，返回 `Deferred<T>`。`launch` 中未捕获异常会立即向父协程传播，而 `async` 更像把异常延迟到 `await()` 时再抛出。面试里要强调：如果只是并发做两件事但不需要结果，用 `launch` 更清晰；如果要聚合多个结果，用 `async + await`。但 `async` 不能滥用，否则代码会变成“披着协程外衣的 Future 风格”。

### 2.
**题目**：为什么不建议在业务代码里随意使用 `GlobalScope`？

**考察要点**：生命周期泄漏、取消链断裂、结构化并发。

**参考答案**：`GlobalScope` 创建的是顶层协程，不受页面和业务对象生命周期约束，页面销毁后任务仍可能继续执行。风险在于取消链断裂、资源泄漏和异常难追踪。结构化并发强调任务必须挂靠到明确父作用域，因此生产代码更推荐 `viewModelScope`、`lifecycleScope` 或注入的业务作用域。

### 3.
**题目**：`CoroutineScope` 和 `CoroutineContext` 分别解决什么问题？

**考察要点**：作用域职责、上下文元素、组合关系。

**参考答案**：`CoroutineScope` 可以理解为“协程启动器”，它本质上持有一个 `CoroutineContext`，并规定了后续启动任务的默认上下文。`CoroutineContext` 是一个可组合的键值集合，里面常见元素有 `Job`、`Dispatcher`、`CoroutineName` 和 `CoroutineExceptionHandler`。前者回答“任务归谁管”，后者回答“任务在哪跑、出了问题谁处理”。面试中如果你能说出 `scope + Dispatchers.IO` 实际是生成了新的上下文组合。

### 4.
**题目**：`Dispatchers.Main`、`IO`、`Default` 的区别是什么？

**考察要点**：线程池特性、CPU/IO 密集、切换成本。

**参考答案**：`Dispatchers.Main` 绑定主线程，适合更新 UI；`Dispatchers.IO` 适合数据库、文件、网络等阻塞型任务；`Dispatchers.Default` 适合排序、解析、图像处理等 CPU 密集型工作。关键不是死记“什么 API 放哪”，而是看任务是否阻塞线程、是否需要并行计算。此外频繁 `withContext` 切换也有调度成本。

```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao,
    private val io: CoroutineDispatcher = Dispatchers.IO,
) {
    suspend fun refresh(): List<User> = withContext(io) {
        val remote = api.fetchUsers()
        dao.insertAll(remote)
        remote
    }
}
```

### 5.
**题目**：什么是结构化并发（Structured Concurrency）？

**考察要点**：父子关系、取消传播、异常传播、资源管理。

**参考答案**：结构化并发要求新协程必须在某个作用域中启动，并且其生命周期受父协程控制。这样父协程在结束前会等待子协程完成，取消时会自动向下传播，异常也能沿着层级往上冒泡。它解决了传统回调和线程池里“任务飞出去就找不到”的问题。面试时最好举例：页面触发两个并发请求，任一失败时可以整体取消，页面销毁时也能统一停止。

### 6.
**题目**：`coroutineScope` 和 `supervisorScope` 的差异是什么？

**考察要点**：兄弟协程失败影响、适用场景。

**参考答案**：`coroutineScope` 遵循普通父子失败传播规则，一个子协程失败会取消同级兄弟任务，最终让整个作用域失败；`supervisorScope` 则更宽松，某个子协程失败不会自动取消其他兄弟。前者适合“全有或全无”的聚合任务，后者适合“部分失败可容忍”的并发场景，比如首页多个卡片并行加载。很多候选人只会背区别，真正加分点是能说出业务语义：不是语法不同，而是失败策略不同。

### 7.
**题目**：`Job` 和 `SupervisorJob` 有什么不同？

**考察要点**：失败隔离、ViewModel 场景。

**参考答案**：普通 `Job` 下，子任务异常会向上失败，再由父任务取消其他孩子；`SupervisorJob` 则让子任务彼此隔离，一个失败不必拖垮整个父作用域。`viewModelScope` 内部就用了监督式策略，因为页面上多个异步任务经常互不依赖。需要注意的是，`SupervisorJob` 只隔离“子任务之间”的失败，不会让异常凭空消失；如果不消费异常，仍然会出现在日志或顶层处理器里。

### 8.
**题目**：协程取消（Cancellation）为什么说是“协作式”的？

**考察要点**：取消检查点、挂起函数、CPU 死循环。

**参考答案**：协程不是抢占式中断线程，而是在挂起点或显式检查点感知取消状态，所以叫协作式。大多数挂起函数如 `delay`、`withContext`、`receive` 都会检查取消；但如果你写的是纯 CPU 循环，没有调用挂起函数，就必须手动用 `ensureActive()`、`yield()` 或检查 `isActive`。面试里常见追问是“为什么我 cancel 了任务还没停”，答案通常就是代码没有给取消任何检查机会。

### 9.
**题目**：`withTimeout` 和 `withTimeoutOrNull` 的区别与坑点是什么？

**考察要点**：超时异常、资源释放、finally。

**参考答案**：`withTimeout` 超时会抛 `TimeoutCancellationException`，而 `withTimeoutOrNull` 会返回 `null`。它们本质都属于取消，因此资源释放仍应放在 `finally`。另外，超时不代表底层阻塞 API 一定立刻停止，是否真正中断还取决于底层库是否支持取消。

### 10.
**题目**：`CoroutineExceptionHandler` 什么时候有用，什么时候没用？

**考察要点**：顶层协程、子协程、`async` 例外。

**参考答案**：`CoroutineExceptionHandler` 主要处理未被捕获的顶层异常，最常见于根 `launch`。对 `async` 来说，异常通常在 `await()` 时暴露，因此 handler 不是主控制手段。它更像日志和兜底机制，业务恢复仍应通过 `try/catch`、返回值或状态建模完成。

## 2. Flow 与响应式流

### 13.
**题目**：冷流（Cold Flow）和热流（Hot Flow）的区别是什么？

**考察要点**：启动时机、是否共享、典型类型。

**参考答案**：冷流（Cold Flow）是“有收集者才执行”，每次 `collect` 都会重新触发上游逻辑；热流（Hot Flow）则独立于收集者存在，生产和消费可以解耦。普通 `flow {}` 默认是冷流，`StateFlow`、`SharedFlow`、`Channel` 转流后通常体现为热源。冷流适合数据库查询、一次性请求；热流适合状态广播、事件分发、持续订阅。

### 14.
**题目**：`StateFlow`、`SharedFlow` 和 `LiveData` 怎么比较？

**考察要点**：状态语义、生命周期感知、回放机制。

**参考答案**：`StateFlow` 适合表示“当前状态”，必须有初始值，永远持有最新值；`SharedFlow` 更通用，可以配置重放（replay）和缓冲，适合事件流或多播；`LiveData` 天然生命周期感知，但协程生态和操作符能力弱于 Flow。现在主流 Android 新项目更常见的是 `ViewModel + StateFlow + collectAsStateWithLifecycle`。面试时不要简单说 `StateFlow` 替代 `LiveData`，而要强调：前者更统一于 Kotlin 协程模型，后者仍在旧项目和 XML 架构里常见。

```kotlin
data class UserUiState(
    val loading: Boolean = false,
    val users: List<String> = emptyList(),
    val error: String? = null,
)

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState

    private val _events = MutableSharedFlow<String>(extraBufferCapacity = 1)
    val events: SharedFlow<String> = _events
}
```

### 15.
**题目**：`stateIn` 和 `shareIn` 的作用是什么？

**考察要点**：冷流转热流、共享上游、启动策略。

**参考答案**：这两个 API 用来把冷流提升为共享热流，避免多个收集者重复执行上游。`stateIn` 产出 `StateFlow`，适合 UI 状态；`shareIn` 产出 `SharedFlow`，适合广播型数据。核心追问一般是 `SharingStarted`：`WhileSubscribed` 可以降低无订阅时的资源消耗，`Eagerly` 会立即启动上游，`Lazily` 则首个订阅者出现才开始。答题时最好补充一句：它们经常配合 `viewModelScope` 使用，因为热流必须依附一个持续存在的作用域。

### 16.
**题目**：`flowOn` 到底改变了什么？

**考察要点**：上游上下文、下游不变、线程切换边界。

**参考答案**：`flowOn` 只影响它上游的执行上下文，不影响下游收集者。因此 `flow { emit(load()) }.flowOn(IO).collect { render(it) }` 的含义是：上游在 IO 执行，下游 `collect` 仍在当前协程上下文。很多人误以为 `flowOn` 会像 Rx 一样全局切换线程，这是错误的。面试时如果能说出“`flowOn` 内部通过 channel 边界切分上下游协程”，通常会很加分。

### 17.
**题目**：`map`、`filter`、`combine`、`zip` 的区别是什么？

**考察要点**：逐项变换、筛选、双流组合、时序差异。

**参考答案**：`map` 做逐项变换，`filter` 做条件筛选。`zip` 是严格一一配对，两边都各来一个值才组合一次；`combine` 则基于双方最新值，任一侧更新都可重新产出结果。搜索条件联动、登录态合成更适合 `combine`，强调顺序配对时更适合 `zip`。

### 18.
**题目**：`buffer`、`conflate`、`collectLatest` 有什么区别？

**考察要点**：背压（Backpressure）、丢弃策略、取消旧任务。

**参考答案**：`buffer` 让上游和下游通过缓冲区解耦，提高吞吐；`conflate` 只保留最新值，适合高频状态刷新；`collectLatest` 则在新值到来时取消旧处理逻辑。三者都在解决慢收集者问题，但丢值策略和取消语义完全不同，不能混着用。

### 19.
**题目**：Flow 中常见的异常处理方式有哪些？

**考察要点**：`catch`、`onCompletion`、`retryWhen`、边界。

**参考答案**：`catch` 处理的是它上游抛出的异常，通常用于把异常转成兜底值或错误状态；`onCompletion` 更像 finally，可以同时观察正常结束和异常结束；`retryWhen` 适合做有限次数重试和退避策略。高频陷阱是：`catch` 放在链路末尾并不能捕获下游 `collect` 代码里的异常。工程实践里，经常在 Repository 层把异常转成 `Result` 或领域错误，再由 ViewModel 映射成 UI 状态。

### 20.
**题目**：什么时候应该用 `callbackFlow`？

**考察要点**：桥接回调、取消释放、`awaitClose`。

**参考答案**：当你需要把传统回调 API、监听器或 SDK 事件桥接成 Flow 时，`callbackFlow` 很合适。它允许你在内部调用 `trySend` 发射事件，并在 `awaitClose` 中注销监听器，避免泄漏。答题时一定要强调：`callbackFlow` 不是为了“任何时候都能发 Flow”，而是为了解决回调源头的生命周期收尾问题。如果忘了在 `awaitClose` 中 remove listener，是典型面试陷阱。

```kotlin
fun NetworkMonitor.observe(): Flow<Boolean> = callbackFlow {
    val listener = object : Listener {
        override fun onChanged(available: Boolean) {
            trySend(available)
        }
    }
    register(listener)
    awaitClose { unregister(listener) }
}
```

## 3. Channel 与并发通信

### 21.
**题目**：`Channel` 的几种容量类型分别适合什么场景？

**考察要点**：Rendezvous、Buffered、Unlimited、Conflated。

**参考答案**：Rendezvous 通道容量为 0，发送和接收必须“握手”，适合同步交接；Buffered 有固定缓冲，适合一般生产消费；Unlimited 几乎不施加背压，容易堆内存，只在你明确知道上游速度可控时使用；Conflated 只保留最新值，适合进度、位置、状态等“旧值无意义”场景。面试中要明确：选择容量其实是在选择背压策略，而不是只看“会不会阻塞”。

### 22.
**题目**：`Channel` 和 `SharedFlow` 应该如何选？

**考察要点**：一对一/一对多、关闭语义、热流替代。

**参考答案**：`Channel` 更偏消息队列语义，天然支持 `close`，常用于一对一消费或明确的生产者-消费者模型；`SharedFlow` 更偏广播语义，一份数据可以被多个收集者接收。很多 Android 场景下，UI 事件已经更推荐 `SharedFlow`，因为它和 Flow 操作符体系更统一。只有当你需要显式关闭、背压控制或严格队列语义时，`Channel` 才更有优势。

### 23.
**题目**：什么是 `produce`/`consume` 模式？

**考察要点**：协程内建生产者、资源关闭、流水线。

**参考答案**：`produce` 是一种快捷方式，可以在协程里创建并返回 `ReceiveChannel`；消费者侧可以循环读取，也可以用 `consumeEach`。这种模式很适合构建流水线，比如下载、解析、落库三段式处理。注意事项是：通道用完要及时关闭或取消，否则后台协程会继续存活。面试里如果你能把它和 Flow 对比，说出 Flow 更声明式、Channel 更偏主动消息传递，会显得思路清楚。

## 4. 语言特性与 Kotlin 2.x

### 25.
**题目**：`sealed class` 和 `sealed interface` 的价值是什么？

**考察要点**：封闭层次、`when` 穷尽、状态建模。

**参考答案**：密封类型的核心价值是“编译器知道所有子类型”，因此在 `when` 表达式里可以做穷尽检查。Android 中最常见用途就是建模 UI 状态、单次事件、领域结果。`sealed interface` 比 `sealed class` 更灵活，因为实现类还可以继承其他类。面试时最好给出状态建模案例，而不是只停留在语法层。

### 26.
**题目**：委托（Delegation）中的 `by lazy` 和 `by map` 各适合什么场景？

**考察要点**：懒加载、属性映射、反射替代。

**参考答案**：`by lazy` 用于第一次访问时再初始化，适合昂贵对象或延迟创建 UI/仓库实例；它还能配置线程安全策略。`by map` 则常用于把 Map 映射成对象属性，适合解析动态配置、参数容器或 DSL 结果。真正加分点在于指出：委托不是语法糖展示，而是把通用逻辑抽成可复用策略，比如缓存、观察、鉴权都可以借助自定义委托实现。

### 27.
**题目**：为什么 `inline` 会提升高阶函数性能？

**考察要点**：对象分配、lambda 调用、字节码展开。

**参考答案**：高阶函数通常会带来 lambda 对象分配和一次额外调用层；`inline` 让编译器把函数体和 lambda 在调用处展开，从而减少对象创建与栈帧开销。它对集合链式处理、锁封装、DSL builder 都很有价值。但面试还要补充副作用：过度内联会增大字节码体积，因此不是所有高阶函数都该 `inline`。如果被追问 `noinline`、`crossinline`，要能说明前者禁止某个参数内联，后者禁止非局部返回。

### 28.
**题目**：`reified` 解决了什么问题？

**考察要点**：类型擦除、泛型反射、内联限制。

**参考答案**：JVM 上泛型会发生类型擦除，所以普通泛型函数运行时拿不到 `T::class`。`inline + reified` 让类型信息在调用点具体化，因此可以直接写 `is T`、`T::class`，经常用于路由、JSON 解析、依赖查找。要提醒面试官你知道它的限制：`reified` 只能出现在内联函数中，因为它依赖编译期展开。

### 29.
**题目**：扩展函数（Extension Function）是不是“真的给类加了方法”？

**考察要点**：静态分发、可见性、成员优先。

**参考答案**：不是。扩展函数本质上是静态函数，编译器只是把接收者对象作为第一个参数传入，所以它无法访问类的私有成员，也不会参与真正的虚方法重写。调用哪个扩展函数，取决于变量的静态类型，而不是运行时实际类型。高频追问是“成员函数和扩展函数同名谁优先”，答案是成员优先。

### 30.
**题目**：`let`、`run`、`with`、`apply`、`also` 如何快速区分？

**考察要点**：接收者、返回值、副作用语义。

**参考答案**：区分它们有两个维度：上下文对象是 `this` 还是 `it`，返回的是对象本身还是 lambda 结果。`apply`/`also` 常用于对象配置，前者用 `this`，后者用 `it`；`run`/`with` 更适合在对象上下文中计算结果；`let` 经常配合空安全做链式处理。真正面试别背表格，而是说出编码意图：想“继续返回对象”就优先 `apply/also`，想“基于对象产出结果”就优先 `run/let/with`。

### 31.
**题目**：Kotlin 的空安全（Null Safety）有哪些常见模式？

**考察要点**：安全调用、Elvis、早返回、避免 `!!`。

**参考答案**：最常用的是安全调用 `?.`、Elvis 操作符 `?:`、`takeIf`、`requireNotNull` 与早返回。优秀代码不是“把 `!!` 换成 `?.`”，而是通过类型建模减少 nullable 传播，比如 Repository 保证非空，UI 层只消费合法状态。面试时可以强调：`null` 常常意味着状态不完整，很多时候应该用 `sealed class` 表达 Loading 或 Empty，而不是拿一个可空字段硬扛所有语义。

### 32.
**题目**：什么是智能转换（Smart Cast），它为什么有时会失效？

**考察要点**：不可变引用、并发修改、开放属性。

**参考答案**：当编译器能证明某个变量在检查后不会被改动时，就会自动把类型收窄，这就是智能转换。比如 `if (x is String) println(x.length)` 中，分支内 `x` 会被视为 `String`。它失效通常是因为变量是可变的、是开放属性、或者编译器无法证明检查后期间没有被其他代码修改。回答时点出“编译器是在做保守证明”会显得你理解的是类型系统，而不是表面语法。

### 33.
**题目**：Kotlin 2.x 里的 K2 编译器、上下文接收者（Context Receivers）和值类（Value Class）有哪些面试价值？

**考察要点**：编译性能、DSL 能力、类型安全轻量封装。

**参考答案**：K2 编译器的价值在于统一前端、提升分析速度并改进错误诊断。上下文接收者适合 DSL 和能力注入，让函数明确声明依赖上下文；值类则用更轻量的方式提供类型安全，适合 `UserId`、`Token` 这类语义化标识。回答时要补充：值类并非绝对零成本，装箱场景仍可能出现。

```kotlin
@JvmInline
value class UserId(val value: String)

context(StringBuilder)
fun line(text: String) {
    appendLine(text)
}
```

## 延伸阅读

- Kotlin 官方文档：Coroutines、Flow、Language Reference
- kotlinx.coroutines 官方设计文档
- Android Developers：Coroutines best practices
- JetBrains 关于 K2 编译器与 Kotlin 2.x 特性的发布说明

## 本章要点

1. 协程题重点讲结构化并发、取消传播与异常。
2. `StateFlow`、`SharedFlow`、`Channel` 的核心差异在语义与背压模型。


