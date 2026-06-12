# 第三章：Android 架构与组件

Android 架构面试通常不考“你知道多少名词”，而考“你如何把状态、数据、依赖和模块组织起来”。本章围绕 MVVM、MVI、Clean Architecture、依赖注入（Dependency Injection）、Room、DataStore、WorkManager、Paging 3 和多模块架构展开。

```text
UI(Compose/View)
    |
 ViewModel  <---- one-shot event / state reducer
    |
 UseCase / Reducer
    |
 Repository
  /      \
Room    Network
```

## 1. MVVM 与状态建模

### 1.
**题目**：为什么 Android 项目里 MVVM 仍然是主流架构？

**考察要点**：关注点分离、生命周期、测试性。

**参考答案**：MVVM 的优势在于把界面渲染逻辑、业务状态和数据访问层解耦。ViewModel 天然适配 Android 生命周期，能跨配置变更保留状态；View 只负责订阅状态和转发用户事件。相比 MVP，MVVM 更适合配合 Flow/Compose 做响应式状态管理。它之所以主流，不是因为“最先进”，而是因为在复杂度、学习成本和工程落地之间取得了较好平衡。

### 2.
**题目**：请描述 `ViewModel + StateFlow + Repository` 的典型数据流。

**考察要点**：单向数据流、状态暴露、异步加载。

**参考答案**：典型流程是：UI 触发意图，ViewModel 调用 Repository，Repository 再协调本地数据库和远程接口，最后 ViewModel 把结果映射为 `StateFlow<UiState>`。UI 层只收集状态，不直接碰数据源。优秀回答应该点出：Repository 不只是“网络请求类”，它是数据来源协调者；ViewModel 不应该泄漏可变状态，而应暴露只读 `StateFlow`。

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repo: ArticleRepository,
) : ViewModel() {
    private val _uiState = MutableStateFlow(ArticleUiState())
    val uiState: StateFlow<ArticleUiState> = _uiState

    fun refresh() = viewModelScope.launch {
        _uiState.update { it.copy(loading = true) }
        runCatching { repo.refresh() }
            .onSuccess { list -> _uiState.update { it.copy(loading = false, items = list) } }
            .onFailure { e -> _uiState.update { it.copy(loading = false, error = e.message) } }
    }
}
```

### 3.
**题目**：UI 状态为什么推荐用 `data class + sealed class` 建模？

**考察要点**：不可变、穷尽性、可测试性。

**参考答案**：`data class` 适合表达页面长期状态，例如 loading、列表数据、错误信息；`sealed class` 适合表达离散状态或结果类型，例如 `Loading/Success/Error`。二者结合后，状态结构更清晰，测试时也能对每种分支做穷尽断言。面试时要强调：不要把所有东西塞进一堆可空字段，那会让状态语义模糊、组合非法状态变多。

### 4.
**题目**：单次事件（Toast、Navigation）为什么不适合直接放进 `StateFlow`？

**考察要点**：重放、副作用、事件消费。

**参考答案**：`StateFlow` 会持有最新值，新订阅者会立即收到，因此它更适合“当前状态”，不适合“一次性事件”。如果把导航事件放进 `StateFlow`，配置变更后可能重复消费。更稳妥的方式是 `SharedFlow`、`Channel` 或明确的 event wrapper。答题时可以补充：理想情况下，UI 事件应尽量由状态推导，只有真正一次性的副作用才使用事件流。

### 5.
**题目**：MVI（Model-View-Intent）的核心流程是什么？

**考察要点**：Intent、Reducer、State、单向数据流。

**参考答案**：MVI 强调单向数据流：用户发出 Intent，系统经过处理后产出 Result，再通过 Reducer 折叠为新 State，View 根据 State 渲染。它把“状态如何变化”表达得比传统 MVVM 更明确，特别适合复杂交互和可回放调试。代价是样板代码更多，对团队建模能力要求更高。

### 6.
**题目**：MVVM 和 MVI 应该如何比较？

**考察要点**：复杂度、状态透明度、学习成本。

**参考答案**：MVVM 更灵活，适合大多数业务页面；MVI 更严格，强调不可变状态和 reducer，适合交互复杂、状态机明显的页面。很多团队其实采用“MVVM 外壳 + MVI 内核”：对外仍是 ViewModel，对内用 intent/reducer 管状态。面试时不要把两者说成二选一，更好的回答是结合团队规模、页面复杂度和调试需求分析取舍。

## 2. Clean Architecture 与依赖注入

### 7.
**题目**：Clean Architecture 的三层分别是什么？

**考察要点**：presentation、domain、data、依赖方向。

**参考答案**：表现层（presentation）负责 UI、ViewModel 与状态映射；领域层（domain）负责 use case、实体规则和业务编排；数据层（data）负责网络、本地存储与 Repository 实现。依赖方向应向内：外层依赖内层接口，而不是反过来。高分答案会指出：不是所有项目都必须强行三层拆到底，小项目过度分层反而增加样板成本。

### 8.
**题目**：Use Case 的价值是什么？

**考察要点**：业务复用、编排边界、测试隔离。

**参考答案**：Use Case 用来承载一个明确业务动作，例如“登录”“同步草稿”“提交订单”。它的价值不只是“多包一层”，而是把跨多个 Repository 的编排、校验和策略集中起来，让 ViewModel 更薄、更好测。如果业务只是简单透传 DAO 或 API，Use Case 可以不必每个都创建；关键是是否有独立业务语义。

### 9.
**题目**：Hilt 的 `Module`、`Component`、`Scope`、`Qualifier` 分别是什么？

**考察要点**：依赖提供、对象图、生命周期、同类型区分。

**参考答案**：`Module` 用来声明如何提供依赖；`Component` 表示依赖图及其生命周期边界；`Scope` 表示同一组件范围内对象复用策略；`Qualifier` 用来区分同类型但语义不同的依赖，比如不同 baseUrl 的 `OkHttpClient`。面试时要能说清楚：Hilt 不是运行时反射容器，而是基于 Dagger 的编译期生成方案，因此启动性能和类型安全更好。

### 10.
**题目**：`SingletonComponent`、`ActivityRetainedComponent`、`ViewModelComponent` 的差异是什么？

**考察要点**：生命周期匹配、对象共享粒度。

**参考答案**：`SingletonComponent` 随应用进程存在，适合数据库、网络客户端、全局配置；`ActivityRetainedComponent` 跨配置变更保留，适合与 Activity 相关但不想因旋转重建的对象；`ViewModelComponent` 则与单个 ViewModel 生命周期一致。真正的面试重点是“依赖应该绑定到最小合理生命周期”，而不是图省事全部做成单例。

### 11.
**题目**：Hilt 和 Compose 如何协同工作？

**考察要点**：`hiltViewModel()`、作用域、导航图。

**参考答案**：在 Compose 中最常见的模式是通过 `hiltViewModel()` 获取页面 ViewModel，再由 ViewModel 暴露 `StateFlow` 给 Composable 收集。若采用 Navigation Compose，还要注意 ViewModel 是绑定到 NavBackStackEntry 还是更高层图。优秀回答会提到：Composable 自身尽量不要直接注入太多业务依赖，而应通过参数接收状态和回调，保持可预览、可测试。

### 12.
**题目**：Hilt 和 Koin 的差异是什么？

**考察要点**：编译期/运行时、易用性、性能。

**参考答案**：Hilt 基于 Dagger，编译期生成依赖图，类型安全强、性能稳定，但学习曲线较高；Koin 更偏运行时 DSL，配置直观、上手快，但错误常在运行时暴露，大规模项目下性能和可追踪性通常不如 Hilt。面试里最好的回答不是站队，而是说明：团队规模小、快速验证时 Koin 很舒服；复杂 Android 商业项目里 Hilt 更主流。

## 3. Room 与本地存储

### 13.
**题目**：Room 的 `Entity`、`DAO`、`Database` 如何分工？

**考察要点**：表结构、访问接口、数据库入口。

**参考答案**：`Entity` 描述表结构，`DAO` 声明查询、插入、更新、事务等访问方法，`RoomDatabase` 则是整个数据库的入口，负责配置版本、导出 schema 和提供 DAO。Room 的价值不只是“SQLite 包装器”，而是它能在编译期校验 SQL 与返回类型匹配，减少运行时错误。

### 14.
**题目**：Room Migration 为什么是高频考点？

**考察要点**：线上升级安全、自动/手动迁移、schema 管理。

**参考答案**：因为数据库 schema 一旦上线，就涉及老用户升级路径。Room 支持自动迁移，但前提是变更足够简单；复杂变更如拆表、数据回填、字段合并通常仍需手写 Migration。面试时要说明：迁移不仅是改版本号，更要把导出的 schema 纳入版本管理，并对升级链做测试。

### 15.
**题目**：`TypeConverter` 用来解决什么问题？

**考察要点**：复杂类型持久化、自定义序列化。

**参考答案**：SQLite 天然只支持有限的基础类型，而业务模型经常包含日期、枚举、列表或嵌套对象。`TypeConverter` 允许你把这些类型转换为数据库可存储格式，再在读取时还原。面试里最好提醒：转换后的格式要考虑可迁移性和查询能力，比如把复杂对象全塞成 JSON 虽方便，但会牺牲 SQL 查询能力。

### 16.
**题目**：Room 中 1:N、M:N 关系通常怎么建模？

**考察要点**：外键、中间表、`@Relation`。

**参考答案**：1:N 常通过子表外键指向父表，再用 `@Relation` 在查询结果中组装；M:N 则需要中间交叉表，再通过 `Junction` 建立多对多关系。答题时要指出：Room 不会自动帮你优化大关系查询，复杂场景仍要关注 SQL 性能、索引和事务边界。

### 17.
**题目**：Room 和 Flow 集成时有什么优势？

**考察要点**：可观察查询、自动刷新、线程处理。

**参考答案**：Room 支持直接从 DAO 返回 `Flow<T>`，当底层表发生变化时，订阅者会自动收到新结果。这非常适合离线优先和 Compose 页面，因为数据库就是单一真实源。面试高分点是补充：Flow 返回并不代表一切都自动高效，复杂联表查询仍要谨慎，否则会因为频繁 invalidation 导致不必要计算。

## 4. DataStore、WorkManager、Paging 3

### 18.
**题目**：DataStore 的 Preferences 模式和 Proto 模式怎么选？

**考察要点**：类型安全、可演进、使用门槛。

**参考答案**：Preferences DataStore 类似键值存储，不需要提前定义 schema，适合轻量配置；Proto DataStore 基于 protobuf，类型安全更强、可演进性更好，适合结构化配置。面试里通常推荐：简单开关、主题、最近一次时间戳可以用 Preferences；业务配置、用户偏好模型更建议 Proto。

### 19.
**题目**：从 `SharedPreferences` 迁移到 DataStore 时要注意什么？

**考察要点**：迁移时机、单写源、线程安全。

**参考答案**：迁移时要确保只有一个新的写入源，避免 SharedPreferences 和 DataStore 并存导致状态漂移。DataStore 提供迁移 API，可以在首次读取时完成旧值导入。还要强调一点：不要把 DataStore 当数据库使用，它适合小型配置，不适合高频复杂查询。

### 20.
**题目**：WorkManager 的核心优势是什么？

**考察要点**：持久任务、系统兼容、约束调度。

**参考答案**：WorkManager 统一封装了不同 Android 版本上的后台任务能力，适合需要“即使应用退出后也尽量执行”的可延迟工作，如同步、日志上传、备份等。它支持网络、电量、充电等约束，并能在进程被杀后恢复。回答时要说清：它不是实时任务框架，不适合秒级立刻执行的前台交互。

### 21.
**题目**：链式任务（Chaining）、周期任务（Periodic Work）、加急任务（Expedited Work）分别适合什么场景？

**考察要点**：依赖顺序、重复调度、系统配额。

**参考答案**：链式任务适合“先下载再解析再上报”这类有依赖顺序的后台流程；周期任务适合周期性清理、同步，但不能期待精确准点；加急任务适合用户刚触发且需要尽快执行的工作，但受系统配额和 Android 版本限制。面试时提到“加急也不是无限制前台服务替代品”会更稳。

### 22.
**题目**：Paging 3 的 `PagingSource` 是什么？

**考察要点**：分页加载单元、key、刷新。

**参考答案**：`PagingSource<Key, Value>` 描述如何根据某个分页 key 加载一页数据，并返回 `LoadResult`。它负责分页协议，而不是直接承担 UI 状态。答题时要能说出 `prevKey/nextKey` 的作用，以及 `invalidate()` 会触发重新创建新的 PagingSource 实例。

### 23.
**题目**：`RemoteMediator` 解决了什么问题？

**考察要点**：本地缓存 + 远程同步、单一数据源。

**参考答案**：`RemoteMediator` 用于协调网络和本地数据库，让 Paging 列表优先从 Room 读取，再在后台决定何时补远程数据。它非常适合离线优先和大列表缓存场景。高频面试点是：页面最终观察的是本地数据库分页流，而不是直接观察网络结果，这样才能保证一致性和恢复能力。

### 24.
**题目**：PagingData 在 Compose 中通常如何使用？

**考察要点**：`collectAsLazyPagingItems`、加载态、错误态。

**参考答案**：Compose 中通常通过 `pager.flow.collectAsLazyPagingItems()` 把分页流接入 `LazyColumn`。然后基于 `loadState.refresh/append/prepend` 渲染首屏加载、尾部加载和错误重试 UI。面试时别只写 API，要指出“分页列表不是只有成功态”，滚动恢复、空态、重试交互都很重要。

## 5. 多模块架构

### 25.
**题目**：为什么大型 Android 项目要做多模块（Multi-module）？

**考察要点**：解耦、并行开发、构建性能、边界治理。

**参考答案**：多模块的核心收益有三点：按领域或功能解耦代码、支持多人并行开发、减少增量构建影响范围。它还能强迫团队显式梳理依赖边界，避免 app 模块变成巨型黑洞。面试里要补充现实代价：模块拆太碎会增加样板、版本管理和调试成本，所以模块化是治理手段，不是越多越好。

### 26.
**题目**：模块间导航和依赖应该如何设计？

**考察要点**：接口隔离、路由抽象、反向依赖。

**参考答案**：理想做法是让 feature 模块依赖稳定的 core/common 能力，而不是互相直接依赖。导航可以通过路由接口、导航服务或统一的 destination 协议解耦，避免 A 模块直接引用 B 模块实现类。若再结合 Hilt 提供跨模块接口绑定，就能形成“实现下沉、接口上浮”的结构。这个回答很能体现系统设计意识。

## 本章要点

1. MVVM 依然主流，但复杂页面常借鉴 MVI 的 reducer 与单向数据流。
2. Clean Architecture 的目标是业务边界清晰，而不是机械分层。
3. Hilt 的关键是生命周期匹配和编译期依赖图，不是只会写 `@Inject`。
4. Room、DataStore、WorkManager、Paging 3 共同构成现代 Android 数据与离线能力基础设施。
5. 多模块架构要兼顾解耦、构建效率和团队成本，避免“为拆而拆”。

## 延伸阅读

- Android Architecture Guide
- Hilt、Room、DataStore、WorkManager、Paging 3 官方文档
- Now in Android 开源项目的模块化架构实践
