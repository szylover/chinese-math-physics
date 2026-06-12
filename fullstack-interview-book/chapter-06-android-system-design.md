# 第六章：Android 系统设计题

系统设计题考察的不是“有没有标准答案”，而是你能否在有限时间内把需求、约束、架构、关键数据结构、线程模型和取舍讲清楚。本章给出 4 个高频 Android 设计案例，每个案例都按面试表达方式展开。

## 案例一：设计一个高性能图片加载框架（仿 Glide/Coil）

**题目**：请设计一个高性能图片加载框架，要求支持三级缓存、生命周期感知、图片变换和并发加载。

**考察要点**：缓存层级、请求去重、生命周期绑定、解码与内存控制。

**参考答案**：

### 1）需求拆解
- 功能性：支持 URL、本地文件、资源 ID；支持占位图、错误图、圆角/裁剪等变换；支持内存和磁盘缓存。
- 非功能性：列表快速滚动不卡顿；页面销毁后请求自动取消；弱网下避免重复下载；内存占用可控。

### 2）高层架构
```text
ImageView/Composable Request
          |
     RequestManager  <---- LifecycleOwner / ViewTree
          |
      Engine / Dispatcher
     /        |         \
Memory L1   Memory L2   Disk Cache
     \        |         /
          Decoder / Transformer
                 |
               Result
```

### 3）组件设计
- `RequestManager`：面向 UI 暴露 `load(url).placeholder().into(target)`，并根据 Activity/Fragment 生命周期自动暂停、恢复、取消请求。
- `Engine`：负责请求去重，相同 key 的并发请求合并到同一个任务，避免重复下载与解码。
- 一级缓存可放强引用 LRU，二级缓存可放弱引用或 bitmap pool，磁盘缓存存原图或变换结果。
- `Transformer` 负责圆角、裁剪、模糊等流水线，cache key 需要把变换参数编码进去。

### 4）并发与内存策略
可以用 Kotlin 协程（Coroutine）构建加载流水线：网络下载在 `Dispatchers.IO`，解码和部分 CPU 处理在 `Dispatchers.Default`。同时需要根据目标尺寸做采样，避免原图解码；对活跃请求使用弱引用跟踪，页面销毁后自动取消。

```kotlin
data class ImageRequest(
    val data: String,
    val width: Int,
    val height: Int,
    val transformations: List<String> = emptyList(),
)

class ImageEngine(
    private val memoryCache: LruCache<String, Bitmap>,
    private val diskCache: DiskCache,
    private val loader: suspend (String) -> ByteArray,
) {
    suspend fun execute(request: ImageRequest): Bitmap = coroutineScope {
        val key = buildKey(request)
        memoryCache.get(key)?.let { return@coroutineScope it }
        diskCache.get(key)?.let { bytes ->
            return@coroutineScope decode(bytes, request).also { memoryCache.put(key, it) }
        }
        val bytes = loader(request.data)
        val bitmap = decode(bytes, request)
        diskCache.put(key, bytes)
        memoryCache.put(key, bitmap)
        bitmap
    }
}
```

### 5）关键取舍
- 强缓存命中高，但更占内存；弱引用更灵活，但回收不可控。
- 缓存变换结果能提速，但会膨胀磁盘占用。
- 生命周期自动取消能省资源，但需要准确绑定请求宿主。

### 6）面试表达建议
回答时一定要先讲 key 设计，因为缓存正确性取决于“数据源 + 尺寸 + 变换 + 配置”的联合 key；再讲请求合并与取消，否则框架在滚动场景下很难真正高性能。

## 案例二：设计一个推送通知系统

**题目**：请设计一个 Android 推送通知系统，要求支持 FCM/HMS、多厂商兼容、消息持久化与失败重试。

**考察要点**：推送接入、到达链路、后台限制、通知策略。

**参考答案**：

### 1）需求拆解
- 功能性：接收远程消息，展示通知，支持点击跳转、分组、渠道管理。
- 非功能性：消息尽量不丢失；弱网和进程被杀时可恢复；Android 12+ 背景限制下仍符合平台规范。

### 2）高层架构
```text
Server
  |
FCM / HMS / OEM Push
  |
PushReceiver / FirebaseMessagingService
  |
Persist Queue(Room) ---> Retry Worker(WorkManager)
  |
Notification Builder -> Channel -> System UI
```

### 3）核心设计
- 抽象 `PushProvider` 接口，屏蔽 FCM、HMS、厂商通道差异。
- 收到透传消息后，先快速落地到 Room，再决定是否立即展示通知或延后拉取详情。
- 对于需要确认送达的业务消息，可记录 `messageId`、状态和重试次数，配合 WorkManager 做补偿。
- 通知层面要规划 `NotificationChannel`、分组、静音策略，避免所有消息都打一类高优先级。

```kotlin
@Entity
data class PendingPush(
    @PrimaryKey val id: String,
    val payload: String,
    val retryCount: Int = 0,
    val status: String = "pending",
)

class PushRepository(
    private val dao: PushDao,
) {
    suspend fun persist(id: String, payload: String) {
        dao.upsert(PendingPush(id = id, payload = payload))
    }
}
```

### 4）长连接 vs 轮询
移动端通常优先利用系统推送通道而不是自己维持长连接，因为后者在功耗、保活和平台限制上代价很高。轮询则实现简单，但实时性差、耗电高。优秀回答会指出：业务如果强依赖实时性，应评估前台场景、WebSocket 和服务端消息合并，而不是默认后台常驻长连接。

### 5）Android 12+ 限制
新版本对后台启动 Activity、前台服务和通知权限更严格，因此收到消息后应优先走规范通知唤起用户，而不是强行拉起页面。耗时任务交给 WorkManager，短逻辑尽量在消息回调里快速完成。

### 6）关键取舍
- 直接展示通知快，但可能缺少最新详情；先落库再异步补齐更稳。
- 高优先级推送能提高到达率，但滥用会被系统和厂商限制。
- 多通道接入增加覆盖率，但也增加测试矩阵和故障复杂度。

## 案例三：设计一个离线优先的数据同步架构

**题目**：请设计一个离线优先（Offline-first）的数据同步架构，要求支持本地优先读取、冲突解决和乐观更新。

**考察要点**：单一真实源、同步队列、冲突策略、弱网体验。

**参考答案**：

### 1）需求拆解
- 功能性：用户离线可读写，网络恢复后自动同步；支持列表、详情和草稿编辑。
- 非功能性：本地读取快；冲突可解释；失败可重试；UI 对同步状态可感知。

### 2）高层架构
```text
UI -> ViewModel -> Repository
                 /          \
           Room(Local SSOT)  Sync Engine
                 ^               |
                 |          WorkManager Queue
                 +------ API / Delta Sync
```

### 3）核心原则
- Room 作为单一真实源（SSOT），UI 永远读本地，不直接绑定网络响应。
- 用户修改先写本地并标记为 `pending_sync`，UI 立即更新，实现乐观体验。
- 后台同步引擎扫描待同步队列，按照顺序推送到服务端，成功后更新本地状态。

```kotlin
@Entity
data class NoteEntity(
    @PrimaryKey val id: String,
    val content: String,
    val updatedAt: Long,
    val syncState: String,
)

suspend fun updateNote(id: String, content: String) {
    dao.updateContent(id, content, System.currentTimeMillis(), "pending")
    syncScheduler.enqueue(id)
}
```

### 4）冲突解决
简单场景可用 Last Write Wins（LWW），实现低成本但可能覆盖用户修改；复杂协作场景可以引入字段级合并、版本号比较，甚至借鉴 CRDT（Conflict-free Replicated Data Type）思想。面试里不要求你把 CRDT 数学细节讲透，但最好说明：冲突策略必须和业务一致，评论、文档、任务状态并不适合同一种策略。

### 5）网络与重试
通过网络状态监控决定是否触发同步；真正执行则交给 WorkManager，并结合指数退避和幂等接口设计。关键是 message id / mutation id，要确保重试不会造成服务端重复写入。

### 6）关键取舍
- 本地优先读写体验最好，但同步复杂度明显上升。
- 乐观更新提升体验，但失败回滚需要清晰提示。
- LWW 简单，协同编辑则往往需要更细粒度模型。

## 案例四：设计一个组件化/模块化架构

**题目**：请设计一个大型 Android 组件化/模块化架构，要求支持模块解耦、跨模块导航、依赖注入和构建性能优化。

**考察要点**：模块边界、依赖图、导航协议、构建治理。

**参考答案**：

### 1）需求拆解
- 功能性：支持多个业务团队并行开发；模块可独立测试；公共能力统一复用。
- 非功能性：构建速度可控；依赖边界清晰；跨模块调用不形成环依赖。

### 2）高层架构
```text
app
├─ feature-home
├─ feature-order
├─ feature-profile
├─ core-ui
├─ core-network
├─ core-database
└─ common-kernel
```

推荐依赖方向：`app -> feature -> core -> common`，feature 之间不直接互相依赖。

### 3）跨模块导航
可以定义统一 `Navigator` 接口或 destination 协议，由 app 壳层负责真正注册路由和 NavGraph。业务模块只暴露“我能去哪”和“我需要什么参数”，而不依赖对方实现类。这能避免 feature-home 直接 import feature-order 的页面。

### 4）跨模块依赖注入
Hilt 可在 core 层提供通用基础设施，在 feature 模块中声明各自的 ViewModel、UseCase 和仓库实现。若存在接口在 common/domain、实现在 feature/data，可通过 `@Binds` 做上浮绑定。关键是让编译依赖和运行依赖方向一致。

```kotlin
interface OrderEntry {
    fun route(orderId: String): String
}

@Module
@InstallIn(SingletonComponent::class)
abstract class OrderEntryModule {
    @Binds
    abstract fun bindOrderEntry(impl: OrderEntryImpl): OrderEntry
}
```

### 5）构建性能优化
- 使用 convention plugins 管理公共构建逻辑。
- 减少不必要的 API 级依赖，优先 `implementation`。
- 对资源和 KSP/KAPT 成本较高模块重点治理。
- 按团队边界与业务边界拆分，不要过度碎片化。

### 6）Feature Toggle
每个 feature 模块可暴露 capability 配置，通过远程开关或本地构建开关控制启用。这样灰度发布、区域功能差异和实验能力更容易实现，但也需要治理开关膨胀。

### 7）关键取舍
- 模块越细，解耦越强，但构建脚本和协调成本越高。
- 中心化路由易统一治理，但可能形成平台瓶颈。
- DI 统一后可维护性更好，但初期学习成本更高。

## 本章要点

1. 系统设计题先讲需求和约束，再讲架构和关键组件，不要一上来就堆技术名词。
2. 图片加载、推送、离线同步、模块化都是 Android 高频综合题，回答时要兼顾平台限制。
3. 设计题真正考察的是取舍：性能、内存、复杂度、构建速度、可维护性往往无法同时最优。
4. Kotlin 协程、Room、WorkManager、Hilt、Compose 等现代能力应当自然融入方案，而不是孤立罗列。
5. 练习系统设计时，建议按“需求—架构图—关键流程—失败场景—Tradeoff”固定模板表达。

## 延伸阅读

- Glide、Coil、WorkManager、Room、Hilt 官方文档与源码
- Android 大型应用模块化实践文章
- Offline-first、CRDT、Push delivery 相关工程实践资料
