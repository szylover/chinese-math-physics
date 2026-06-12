# 第二章：Jetpack Compose 面试

Jetpack Compose 是 Android 官方推荐的声明式 UI（Declarative UI）框架。Compose 面试不再只考“你会不会写控件”，而是会深挖组合（Composition）、重组（Recomposition）、状态（State）、副作用（Side Effects）和性能模型。本章按高频面试主线展开，并穿插 Kotlin 代码与架构图。

```text
State -> Compose Runtime -> Recomposition -> UI Tree Update
   ^            |                |                 |
   +------------+---- Snapshot ---+----------------+
```

## 1. Compose 核心模型

### 1.
**题目**：声明式 UI 和命令式 UI（Imperative UI）的根本区别是什么？

**考察要点**：状态驱动、渲染方式、维护成本。

**参考答案**：命令式 UI 强调“如何一步步修改界面”，开发者手动调用 `setText`、`setVisibility`、增删 View；声明式 UI 则强调“给定当前状态，界面应该长什么样”。Compose 通过状态驱动 UI 重绘，开发者主要描述界面函数，而不是手工维护一堆可变 View。面试时要指出优势并非“代码更短”，而是状态来源更清晰、局部更新更容易推理，代价则是需要理解重组与稳定性。

### 2.
**题目**：`@Composable` 注解到底意味着什么？

**考察要点**：编译器插件、函数改写、运行时协议。

**参考答案**：`@Composable` 并不是普通标记注解，它会被 Compose Compiler Plugin 改写，使函数能够参与组合树构建、跳过判断和状态记忆。你看到的是普通 Kotlin 函数，编译后其实多了运行时需要的参数和调用协议。面试里如果能说到“Compose 不是 XML 替换品，而是一套编译器 + runtime 协议”，通常会让回答显得更专业。

### 3.
**题目**：什么是 Composition、Slot Table 和 Recomposition？

**考察要点**：UI 树、位置记忆、局部更新。

**参考答案**：Composition 是一次把 `@Composable` 函数执行后生成 UI 结构和状态关联信息的过程。Compose Runtime 会把函数调用位置信息保存在 Slot Table 中，用来记忆状态和判断后续是否可以跳过。Recomposition 则是在某些状态发生变化后，重新执行受影响的那部分组合逻辑。关键不是“整个页面重绘”，而是“尽量局部、可跳过、带记忆地重组”。

### 4.
**题目**：什么叫智能重组（Smart Recomposition）？

**考察要点**：跳过、参数稳定性、最小刷新。

**参考答案**：智能重组的核心是“只重组真正受影响的节点”。如果一个 `Composable` 的输入参数没有变化，且类型被判断为稳定（stable），Compose 就可能跳过这次执行。它并不是魔法，而是依赖编译器分析、运行时快照系统以及你是否遵守不可变数据设计。面试高频坑点是：把大对象、可变集合直接传入 UI，会破坏可跳过性。

### 5.
**题目**：什么是稳定性（Stability）？

**考察要点**：`@Stable`、`@Immutable`、跳过依据。

**参考答案**：稳定性描述的是一个类型在 Compose 看来是否“值变化可预测”。不可变数据类最容易被视为稳定；如果对象内部状态变化无法被 runtime 感知，就可能导致过度重组或错误跳过。`@Immutable` 用于显式声明对象不可变，`@Stable` 则表示该类型虽可变，但其变化符合 Compose 对可观察性的预期。答题时别把注解当银弹，错误标注会掩盖问题。

## 2. 状态管理

### 6.
**题目**：`remember` 和 `mutableStateOf` 分别扮演什么角色？

**考察要点**：状态记忆、状态持有、重组触发。

**参考答案**：`mutableStateOf` 创建的是可观察状态容器，值变化会通知 Compose；`remember` 则负责把对象记忆在当前组合位置上，避免每次重组都重新创建。两者经常一起出现：`remember { mutableStateOf(...) }`。面试时最好指出：`remember` 记住的是“对象”，不是“永不销毁”；当对应组合节点移除时，状态也会被释放。

### 7.
**题目**：`rememberSaveable` 和 `remember` 的区别是什么？

**考察要点**：进程外保存、配置变更、Saver。

**参考答案**：`remember` 只能在当前组合生命周期内保留值，旋转屏幕、进程被杀后不一定能恢复；`rememberSaveable` 会把可保存数据写入 `Bundle`，因此可以跨配置变更甚至部分进程重建场景恢复。复杂对象需要自定义 `Saver`。要提醒面试官：`rememberSaveable` 不是数据库，也不适合放大对象，它解决的是短期 UI 恢复而不是持久化。

```kotlin
@Composable
fun Counter() {
    var count by rememberSaveable { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("count = $count")
    }
}
```

### 8.
**题目**：`derivedStateOf` 的使用价值是什么？

**考察要点**：派生状态、减少重复计算、精准依赖。

**参考答案**：当某个 UI 值可由其他状态推导而来，且你希望只在依赖真正变化时重新计算，就可以用 `derivedStateOf`。典型例子是列表滚动位置映射成“是否显示返回顶部按钮”。它的重点不是“缓存”，而是把依赖关系显式声明出来，避免在每次重组里重复做昂贵计算。过度使用也不好，因为它本身也有维护成本。

### 9.
**题目**：`snapshotFlow` 解决了什么问题？

**考察要点**：Compose State 转 Flow、观察快照读取。

**参考答案**：`snapshotFlow` 可以把 Compose Snapshot 中读取到的状态变化转换为 Flow，方便与协程操作符体系结合。比如把 `LazyListState.firstVisibleItemIndex` 转成滚动事件流，再做节流或埋点。它适合“从 Compose 状态桥接到协程世界”，但不适合替代普通状态读取。面试时要强调：它只会在读取的快照状态变化时发射。

### 10.
**题目**：为什么 Compose 强调状态提升（State Hoisting）？

**考察要点**：单一数据源、可测试性、复用性。

**参考答案**：状态提升就是把状态从子组件中拿出来，由更高层统一持有，再把当前值和事件回调下发。这样做可以建立单一数据源（Single Source of Truth），让组件更可复用、更易测试，也方便与 ViewModel 集成。面试里可以用 `value + onValueChange` 作为标准接口模式举例，这是 Compose 组件设计的基本功。

## 3. 副作用与协程

### 11.
**题目**：`LaunchedEffect` 适合做什么？

**考察要点**：组合感知协程、key 变化、取消重启。

**参考答案**：`LaunchedEffect` 用于在组合生命周期内启动协程，例如首次加载、监听某个 key 变化后重新拉取数据。它的协程会和对应组合节点绑定，key 变化时旧协程取消、新协程重启。回答时要点出：它不应该承担长期业务状态持有，长期任务更适合放在 ViewModel。

### 12.
**题目**：`SideEffect`、`DisposableEffect` 的区别是什么？

**考察要点**：提交后执行、资源注册释放、生命周期。

**参考答案**：`SideEffect` 会在成功重组并提交后执行，适合把 Compose 内部状态同步给外部对象，例如更新 analytics 属性。`DisposableEffect` 则适合“注册-注销”模式，比如添加监听器、订阅生命周期；当 key 变化或节点离开组合时，会先调用 `onDispose` 收尾。高频面试点是：副作用一定要选和生命周期匹配的 API，而不是全塞进 `LaunchedEffect`。

### 13.
**题目**：`rememberCoroutineScope` 的定位是什么？

**考察要点**：事件回调启动协程、组合绑定、与 `LaunchedEffect` 对比。

**参考答案**：`rememberCoroutineScope` 返回一个和当前组合节点生命周期绑定的 `CoroutineScope`，适合在点击事件等回调里启动协程。它和 `LaunchedEffect` 的区别在于：前者由你手动在事件中触发，后者是组合驱动。比如 Snackbar 显示、滚动到指定位置，通常更适合 `rememberCoroutineScope`。如果把一次性页面初始化逻辑放这里，反而会丢失声明式语义。

### 14.
**题目**：`rememberUpdatedState` 为什么经常被忽略？

**考察要点**：捕获最新 lambda、避免副作用重启。

**参考答案**：当副作用内部需要用到外部最新参数，但你又不想因为参数变化而重启整个副作用时，`rememberUpdatedState` 非常有用。典型例子是启动一个长生命周期的超时任务，完成后调用最新的 `onTimeout` 回调。没有它时，开发者常常在 key 中塞太多参数，导致不必要的取消和重启。这个点很能区分“会写 Compose”和“理解 Compose 副作用模型”的候选人。

### 15.
**题目**：`produceState` 的使用场景是什么？

**考察要点**：协程产出 State、异步加载包装。

**参考答案**：`produceState` 用来把异步数据生产过程包装成 `State<T>`，内部可以安全地调用挂起函数并更新 `value`。它适合简单的数据加载桥接，比如从仓库拿一份配置后直接驱动 UI。复杂业务里，通常仍然建议在 ViewModel 中维护 `StateFlow`，因为那样状态来源更集中。面试回答最好体现边界感：Compose 管 UI 层异步桥接，不要吞掉整个业务架构。

## 4. 布局、修饰符与环境

### 16.
**题目**：自定义 `Layout` 的测量（measure）和摆放（placement）流程是什么？

**考察要点**：Constraints、Measurable、Placeable。

**参考答案**：自定义 `Layout` 时，父布局先给出 `Constraints`，子元素作为 `Measurable` 接收约束并返回 `Placeable`，然后父布局在 `layout(width, height)` 中调用 `place`/`placeRelative` 完成摆放。Compose 的布局流程和 View 的 measure/layout 很像，但 API 更函数式。回答时如果能指出“先测量、后决定自身尺寸、再摆放子项”，就是正确主线。

### 17.
**题目**：`SubcomposeLayout` 为什么强大，也为什么昂贵？

**考察要点**：延迟子组合、依赖测量结果再组合、性能成本。

**参考答案**：`SubcomposeLayout` 允许你在测量阶段再去组合部分内容，因此很适合 Tab、Scaffold、复杂自适应布局这类“先知道尺寸再决定内容”的场景。它强大在于打破了一次性组合限制，但代价是实现更复杂、额外组合成本更高。面试时不要只说“高级布局”，而要说明它通常只在标准布局解决不了问题时使用。

### 18.
**题目**：为什么说 `Modifier` 链的顺序会影响结果？

**考察要点**：装饰链、输入区域、绘制顺序。

**参考答案**：`Modifier` 本质上是一个从外到内、逐层包裹的节点链，不同修饰符影响测量、绘制、点击区域和语义树的时机不同。比如 `padding().background()` 和 `background().padding()` 的视觉范围不一样；`clickable().padding()` 与 `padding().clickable()` 的可点击区域也不同。这个问题经常用于考察候选人是否真正调试过 Compose UI。

### 19.
**题目**：`CompositionLocal` 适合用来传什么，不适合传什么？

**考察要点**：隐式依赖、主题/环境、滥用风险。

**参考答案**：`CompositionLocal` 适合传递主题、颜色、间距、权限提供者、导航器这类“跨层级、低频变化、环境型依赖”。它不适合承载页面核心业务状态，因为那会让数据来源变得隐式且难以追踪。面试里最好表明态度：它是依赖注入补充，不是万能全局变量。

## 5. 动画与导航

### 20.
**题目**：`animate*AsState` 的特点是什么？

**考察要点**：单值动画、声明式、易用性。

**参考答案**：`animate*AsState` 适合把某个目标值平滑过渡到另一个值，例如透明度、颜色、偏移量。它最大的优点是 API 简洁，直接根据状态变化自动驱动动画。缺点是对复杂联动场景控制力有限，因此更适合局部、轻量动画。答题时加一句“它本质上还是状态驱动”，会让逻辑更完整。

### 21.
**题目**：`AnimatedVisibility` 和 `Crossfade` 的区别是什么？

**考察要点**：显隐动画、内容切换、适用场景。

**参考答案**：`AnimatedVisibility` 关注的是一个内容块进出场，可配置 enter/exit 动画；`Crossfade` 更偏两个状态内容之间的淡入淡出切换。前者适合展开面板、错误提示、底部按钮显隐；后者适合加载态与结果态切换。面试中如果能补充“复杂状态切换可以考虑 `AnimatedContent` 或 `updateTransition`”，会更完整。

### 22.
**题目**：`updateTransition` 适合什么场景？

**考察要点**：多属性联动、状态机动画。

**参考答案**：当一个状态变化需要驱动多个属性协同动画时，`updateTransition` 比多个 `animate*AsState` 更合适。比如卡片展开时同时改变高度、颜色、阴影和旋转角度。它的优势在于所有动画共享同一状态机，语义更清晰，也更容易保证一致性。系统设计和高级 UI 面试中，这个点常被用来区分“只会堆 API”和“会建模动画状态”。

### 23.
**题目**：`InfiniteTransition` 的使用注意点是什么？

**考察要点**：无限动画、资源消耗、生命周期。

**参考答案**：`InfiniteTransition` 适合做 loading shimmer、呼吸灯等持续性动画，但要注意它会在组件存活期间持续占用帧资源。对于列表中大量 item 同时无限动画，要格外警惕卡顿。面试时最好体现节制原则：动画不是越多越好，应尽量只在少量可见节点启用。

### 24.
**题目**：Navigation Compose 的核心思想是什么？

**考察要点**：路由、`NavHost`、参数与 back stack。

**参考答案**：Navigation Compose 用 `NavHost` 定义路由图，用 `NavController` 管理导航栈，把页面切换从 Fragment 事务抽象为声明式路由跳转。它适合 Compose 单 Activity 架构，但参数建议尽量传轻量 ID，复杂对象交给共享 ViewModel 或数据层恢复。面试里如果你能提到“按 feature 模块拆图、避免全局路由表过大”，会体现工程经验。

## 6. 性能与互操作

### 25.
**题目**：如何避免不必要的重组？

**考察要点**：稳定参数、状态下沉、拆分函数、remember。

**参考答案**：常见手段包括：保持参数稳定、把大状态拆成小状态、避免在 `Composable` 中直接创建新对象、对昂贵计算使用 `remember`/`derivedStateOf`、为列表项提供稳定 key。还要注意 lambda 捕获和集合实例频繁变化，它们都会让 Compose 认为输入变了。面试中别只说“看 metrics”，还要说你如何从数据建模层面减少重组范围。

### 26.
**题目**：Compose Compiler Metrics 能看什么？

**考察要点**：restartable/skippable、稳定性报告、优化线索。

**参考答案**：开启 Compose Compiler Metrics 后，可以看到哪些函数是 restartable、skippable，哪些参数被判定为 unstable，从而定位重组优化方向。它不是给普通业务页面天天开着看的，而是用于性能分析和特定疑难页面诊断。高水平回答会强调：指标只是现象，真正要回到代码结构、不可变建模和参数传递策略上解决问题。

### 27.
**题目**：`AndroidView` 和 `ComposeView` 各自用来解决什么问题？

**考察要点**：View 互操作、渐进迁移、生命周期。

**参考答案**：`AndroidView` 允许在 Compose 中嵌入传统 View，适合地图、WebView、播放器等暂时难以完全 Compose 化的组件；`ComposeView` 则允许在传统 XML/View 体系里嵌入 Compose 内容，适合渐进式迁移。面试时要补充一个实践点：互操作不是“随便混用”，而是迁移过渡策略，过度嵌套会增加测量和生命周期复杂度。

```kotlin
@Composable
fun LegacyMapView(factory: (Context) -> View) {
    AndroidView(factory = factory)
}
```

## 本章要点

1. Compose 的核心是“状态驱动 UI + 编译器协助重组优化”。
2. `remember` 记住对象，`mutableStateOf` 负责可观察变化，`rememberSaveable` 负责短期恢复。
3. 副作用 API 的选择关键在生命周期：`LaunchedEffect`、`DisposableEffect`、`SideEffect` 各司其职。
4. 性能优化重点在稳定类型、合理拆分状态、减少无意义对象创建和重组范围。
5. Compose 与传统 View 能互操作，但目标应是清晰边界和渐进迁移，而不是无限混搭。

## 延伸阅读

- Android Developers：Thinking in Compose
- Compose Runtime 与 Snapshot 官方文档
- Compose Compiler Metrics 配置说明
- Navigation Compose、Animation、Custom Layout 官方指南
