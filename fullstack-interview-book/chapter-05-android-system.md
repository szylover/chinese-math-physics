# 第五章：Android 系统原理

系统原理是 Android 高级面试最能拉开差距的一章。它既考你对框架 API 的理解，也考你是否知道 Activity、Binder、Looper、View、Gradle 背后的运行机制。本章按生命周期、IPC、消息循环、渲染、触摸、ANR、类加载和构建系统展开。

```text
Input -> Activity/Window -> ViewRootImpl
      -> Measure -> Layout -> Draw -> GPU -> Screen

Process
  └─ Main Thread
      ├─ Looper
      ├─ MessageQueue
      └─ Handler callbacks
```

## 1. 生命周期与状态恢复

### 1.
**题目**：请完整描述 Activity 生命周期及典型状态转换。

**考察要点**：`onCreate`、`onStart`、`onResume`、`onPause`、`onStop`、`onDestroy`。

**参考答案**：Activity 创建时通常依次经历 `onCreate -> onStart -> onResume`，进入前台可交互状态；当被部分遮挡或失去焦点时会先走 `onPause`，完全不可见则进入 `onStop`；再次返回前台会经过 `onRestart -> onStart -> onResume`；最终销毁时调用 `onDestroy`。面试时要强调：生命周期不是“固定单线流程”，系统可能因配置变更、回收或多窗口环境触发不同路径。

### 2.
**题目**：配置变更（Configuration Changes）为什么会导致 Activity 重建？

**考察要点**：资源重载、语言/方向变化、错误规避。

**参考答案**：屏幕方向、字体、语言、深色模式等配置变化可能影响资源选择和布局，因此默认策略是销毁并重建 Activity，让界面以新的配置完整加载。虽然可以通过 `configChanges` 自行接管部分变化，但滥用会把系统本该帮你做的资源刷新复杂化。面试里常见加分点是：应优先保证状态可恢复，而不是试图逃避重建。

### 3.
**题目**：进程死亡（Process Death）和简单旋转重建有什么本质不同？

**考察要点**：内存回收、状态持久性、SavedStateHandle。

**参考答案**：旋转重建时，应用进程通常还在，ViewModel 可能保留；进程死亡则意味着内存中对象全部消失，只剩系统保存的有限状态。`SavedStateHandle` 可以帮助 ViewModel 跨进程重建恢复关键参数，但不能替代数据库或磁盘持久化。面试高频误区是把 `ViewModel` 当成“永不丢失的状态仓库”。

### 4.
**题目**：Fragment 生命周期和 Activity 的关系如何理解？

**考察要点**：宿主依赖、view lifecycle、嵌套生命周期。

**参考答案**：Fragment 生命周期受宿主 Activity 和 FragmentManager 管理，但它还有额外的 View 生命周期：`onCreateView` 到 `onDestroyView` 之间才适合持有 ViewBinding、Adapter 等界面资源。很多泄漏问题都源于只记住 Fragment 生命周期，忘了 View 生命周期更短。回答时如果能主动提到 `viewLifecycleOwner`，说明经验比较实战。

### 5.
**题目**：FragmentManager 和 Back Stack 是如何工作的？

**考察要点**：事务、回退栈、状态保存。

**参考答案**：FragmentManager 负责维护 Fragment 实例、事务变更和回退栈。一次 `commit` 可以包含 add/replace/remove 等操作，若加入 back stack，系统会在返回时逆向执行事务恢复先前界面。高分答案会补充：`commitNow`、状态保存后提交异常、嵌套 FragmentManager 都是常见坑点。

## 2. IPC 与 Binder

### 6.
**题目**：Binder 机制为什么是 Android IPC 的核心？

**考察要点**：性能、对象引用语义、驱动支持。

**参考答案**：Binder 是 Android 自研 IPC 机制，核心优势是高性能、支持引用语义和系统级权限控制。它通过内核驱动管理进程间对象引用，客户端拿到的是代理对象（Proxy），服务端对应 Stub，方法调用看起来像本地调用，实则完成序列化、线程切换和跨进程通信。面试时要说出：Binder 不是简单 socket 封装，而是 Android 框架的服务总线基础。

### 7.
**题目**：一次 Binder 调用大致经历哪些步骤？

**考察要点**：Proxy、Parcel、驱动、线程池、Stub。

**参考答案**：客户端调用代理对象方法后，参数会写入 `Parcel`；随后通过 Binder 驱动把事务发送到目标进程；服务端 Binder 线程池中的某个线程读取事务并交给 Stub 解包；Stub 调用真正服务实现后，再把结果写回 `Parcel` 返回客户端。答题时如果能提到“客户端线程会阻塞等待返回，因此不能在主线程做慢 Binder 调用”，会非常加分。

### 8.
**题目**：AIDL、Messenger、ContentProvider、BroadcastReceiver 分别适合什么 IPC 场景？

**考察要点**：接口复杂度、并发模型、数据共享。

**参考答案**：AIDL 适合结构化、双向、并发较强的服务接口；Messenger 基于 Handler，适合消息式、串行化程度高的通信；ContentProvider 适合跨进程数据共享与 CRUD；BroadcastReceiver 适合事件通知和系统广播响应。面试里应体现“按通信模型选技术”，而不是把所有跨进程问题都往 AIDL 上套。

## 3. Handler / Looper / MessageQueue

### 9.
**题目**：为什么主线程需要 Looper？

**考察要点**：事件循环、消息分发、UI 串行模型。

**参考答案**：主线程要处理输入事件、生命周期回调、布局绘制、Handler 消息等大量异步任务，因此需要 Looper 持续从 MessageQueue 取消息并分发。没有 Looper，主线程只能一口气执行完初始化代码就结束，无法成为长期运行的事件循环线程。面试里要补充：Android 选择单线程 UI 模型，是为了避免复杂的并发 UI 访问问题。

### 10.
**题目**：Handler、Looper、MessageQueue 的关系是什么？

**考察要点**：生产者、循环器、队列。

**参考答案**：Looper 是循环器，持续从 MessageQueue 读取消息；MessageQueue 是按时间排序的消息队列；Handler 是消息的发送者和分发入口。发送时，Handler 把 Message 放入目标线程的队列；处理时，Looper 取出消息再回调到 Handler。把三者理解成“投递、排队、消费”的关系就很清楚。

### 11.
**题目**：`HandlerThread` 解决了什么问题？

**考察要点**：自带 Looper 的后台线程、串行任务。

**参考答案**：普通线程默认没有 Looper，不能直接处理 Handler 消息；`HandlerThread` 启动后会自动创建 Looper，因此适合串行后台任务，如日志写盘、相机串行命令、数据库队列等。它比线程池更适合需要明确消息顺序的场景。面试时可顺带指出：现代协程并不会完全替代它，某些与旧框架集成场景仍然常见。

### 12.
**题目**：什么是 `IdleHandler`，它适合做什么？

**考察要点**：空闲任务、时机控制、不能滥用。

**参考答案**：`MessageQueue.IdleHandler` 会在消息队列暂时空闲时得到回调，适合做低优先级、可延后工作，例如预加载、轻量缓存清理、统计打点等。它不保证实时执行，也不能做重任务，否则会反过来破坏主线程空闲时间。面试高分点是指出：IdleHandler 是调度机会，不是后台线程。

## 4. View 渲染与触摸分发

### 13.
**题目**：请描述 View 渲染管线：measure → layout → draw。

**考察要点**：尺寸计算、位置确定、绘制阶段。

**参考答案**：`measure` 阶段决定每个 View 想要和最终能得到的尺寸；`layout` 阶段确定子 View 的最终位置；`draw` 阶段则依次绘制背景、内容、子 View 和前景。这个流程通常由 `ViewRootImpl` 在一次遍历中驱动。面试里如果能补充“requestLayout 可能引发 measure/layout/draw，invalidate 主要触发 draw”，就更完整。

### 14.
**题目**：`requestLayout()` 和 `invalidate()` 的区别是什么？

**考察要点**：重新布局 vs 重新绘制。

**参考答案**：当尺寸或位置可能变化时，应调用 `requestLayout()`，它会请求重新走测量和布局；当只是内容外观变化但尺寸不变时，用 `invalidate()` 请求重绘即可。错误使用会带来性能浪费，比如仅改文字颜色却频繁 requestLayout。这个问题是面试里判断你是否真正理解 View 系统的重要题。

### 15.
**题目**：硬件加速（Hardware Acceleration）带来了什么改变？

**考察要点**：GPU 渲染、显示列表、兼容问题。

**参考答案**：硬件加速把大量绘制工作交给 GPU，并通过显示列表等机制提升渲染性能。它让大多数普通绘制更流畅，但某些自定义绘制 API、混合模式、阴影和离屏缓冲场景也可能出现兼容或性能差异。优秀回答应体现双面性：硬件加速不是“自动更快”，而是需要理解渲染成本迁移到了哪里。

### 16.
**题目**：触摸事件分发链路是什么？

**考察要点**：`dispatchTouchEvent`、`onInterceptTouchEvent`、`onTouchEvent`。

**参考答案**：事件从 Activity/Window 进入顶层 ViewGroup 的 `dispatchTouchEvent`，父容器可在 `onInterceptTouchEvent` 中决定是否拦截；若不拦截，事件继续下发给子 View；最终由目标 View 的 `onTouchEvent` 处理。MOVE 序列中一旦父容器决定拦截，后续事件会转交父容器。面试时最好把 DOWN 事件的重要性说出来：没有正确消费 DOWN，后续序列通常拿不到。

### 17.
**题目**：滚动冲突通常如何解决？

**考察要点**：外部拦截、内部拦截、方向判断。

**参考答案**：常见策略有外部拦截法和内部拦截法。外部拦截法由父容器在 `onInterceptTouchEvent` 中根据方向、距离、阈值判断是否接管；内部拦截法则由子 View 在合适时机调用 `requestDisallowInterceptTouchEvent` 请求父容器暂不拦截。答题时如果能结合 ViewPager/RecyclerView、横竖向滑动案例，会更有说服力。

## 5. ANR、类加载与构建系统

### 18.
**题目**：ANR 的常见触发阈值是什么？

**考察要点**：Activity 5s、BroadcastReceiver 10s、Service 20s。

**参考答案**：经典面试答案是：前台 Activity 输入/启动响应超时常见阈值约 5 秒，BroadcastReceiver 通常约 10 秒，Service 某些超时场景约 20 秒。但真正重要的是理解本质：主线程或关键组件线程长时间无法响应系统期望。面试里最好补充：不同版本和场景下细节会变，但“阻塞关键线程”这一根因不变。

### 19.
**题目**：遇到 ANR 时如何分析 `traces.txt`？

**考察要点**：主线程栈、锁竞争、Binder 阻塞。

**参考答案**：首先看发生 ANR 时主线程栈顶在做什么，是 IO、锁等待、Binder 调用、数据库查询还是无限循环。再看是否有其他线程持有关键锁，或 Binder 线程池是否被打满。优秀回答通常会把 traces、日志、Perfetto 时间线结合起来，而不是只盯着一个文件。

### 20.
**题目**：`PathClassLoader` 和 `DexClassLoader` 的区别是什么？

**考察要点**：应用类加载、动态加载、代码来源。

**参考答案**：`PathClassLoader` 通常用于加载应用安装包和系统类路径中的 dex；`DexClassLoader` 则支持从外部路径加载 dex/jar/apk，更适合动态加载场景。面试里应说明：动态加载带来灵活性，但也伴随安全、兼容和平台限制问题，现代 Android 已越来越谨慎对待这类能力。

### 21.
**题目**：什么是 MultiDex，为什么会出现它？

**考察要点**：65K 方法数限制、主 dex、启动影响。

**参考答案**：由于单个 dex 的方法引用数上限约为 65,536，应用过大时就需要 MultiDex 把代码拆到多个 dex 中。早期设备还需要特别处理主 dex 中必须提前可见的类。现在虽然大多数项目都能平稳使用 MultiDex，但方法数膨胀仍会影响构建、安装和启动表现，所以它不是“开了就完事”。

### 22.
**题目**：Gradle 的 init、configuration、execution 三个阶段分别做什么？

**考察要点**：构建生命周期、任务图、配置成本。

**参考答案**：init 阶段确定参与构建的项目；configuration 阶段执行各模块 build 脚本并创建任务图；execution 阶段只执行最终需要的任务。现代 Gradle 优化的重点之一就是减少 configuration 成本，例如使用 configuration avoidance、缓存和合理插件设计。面试答题时如果能把“为什么配置阶段慢”讲出来，会很加分。

### 23.
**题目**：`buildSrc` 和 convention plugins 应该怎么选？

**考察要点**：构建逻辑复用、隔离、可维护性。

**参考答案**：`buildSrc` 简单直观，但改动后常导致更广泛的构建失效和重新编译；convention plugins 更适合把构建逻辑模块化、显式化，便于大型项目维护。现在官方更推荐使用独立的 convention plugins 管理公共构建约定。面试里要体现的是“构建逻辑也是代码，需要治理”。

### 24.
**题目**：Build Variants 和 AGP（Android Gradle Plugin）在工程化里扮演什么角色？

**考察要点**：多环境、多渠道、插件能力。

**参考答案**：Build Variants 通过 buildType、productFlavor 组合出开发、测试、预发、生产等不同构建版本，支持多环境配置与差异资源；AGP 则把 Android 构建流程接入 Gradle，负责资源处理、dex、打包、签名等大量能力。优秀回答会提到：变体过多会显著增加配置和 CI 成本，因此需要有意识控制组合爆炸。

## 本章要点

1. 生命周期问题的本质是“系统随时可能回收、重建或切换状态”。
2. Binder、Looper、ViewRootImpl 构成了 Android 应用运行时的核心骨架。
3. 触摸、渲染、ANR 三大类问题，最终都能落回线程模型和事件流转。
4. 类加载与 Gradle 代表的是 Android 工程化与运行时边界，高级面试经常会深挖。
5. 回答系统原理题时，务必尽量从“机制 + 典型问题 + 实战定位”三个层次组织答案。

## 延伸阅读

- Android Framework 源码：ActivityThread、ViewRootImpl、Looper、Binder
- 官方文档：Processes and Threads、Background Work、Gradle for Android
- Perfetto 与 ANR 调试相关文章
