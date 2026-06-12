# 第九章：React 与前端面试

React 面试的难点，不在于“会不会写组件”，而在于你能否解释：为什么状态更新是异步批处理（batching）、为什么 `useEffect` 容易踩闭包陷阱、为什么 Fiber（Fiber）要做优先级调度、为什么 Next.js App Router 能把服务端组件（React Server Components, RSC）和客户端组件拆开。本章按面试节奏组织内容：先讲核心原理，再讲 Hooks，再讲状态管理、Next.js、性能优化、测试、样式和可访问性。

---

## 一、React 全景图

```text
UI = f(state)
   │
   ▼
Render Phase
   │  生成新 Fiber 树
   ▼
Reconciliation
   │  对比差异
   ▼
Commit Phase
   │  更新 DOM / 触发生命周期
   ▼
Browser Paint
```

这张图很重要。很多面试题看似分散，比如 Virtual DOM、Fiber、Hooks、性能优化，实际上都可以回到“React 如何把状态变化变成稳定 UI 更新”这个主线。

---

## 二、高频面试题

### 题目 1：什么是 Virtual DOM（虚拟 DOM）？它解决了什么问题？

**考察要点**

- 是否能说出本质而不是神化
- 是否理解它不是“绝对更快”

**参考答案**

Virtual DOM 本质上是对 UI 的一层 JavaScript 对象表示。React 先基于状态生成新的虚拟树，再与旧树比较，最后把必要的变更提交到真实 DOM。它解决的核心问题不是“DOM 一定更快”，而是“声明式 UI 更新的可预测性”和“跨平台渲染抽象”。  

真实 DOM 操作昂贵的不只是 API 调用本身，还包括布局（layout）、绘制（paint）、样式计算等连带成本。React 通过中间层统一管理更新，使开发者不必手工维护 DOM 状态同步。

---

### 题目 2：React 的 diff（reconciliation）规则有哪些关键点？

**考察要点**

- 是否知道同层比较
- 是否知道 key 的作用

**参考答案**

React 的协调算法（reconciliation）主要基于两个经验规则：第一，不同类型的节点会被认为是完全不同的子树，直接卸载旧节点并挂载新节点；第二，同层节点通过 `key` 辅助复用。  

```tsx
{list.map(item => (
  <Row key={item.id} item={item} />
))}
```

如果 `key` 不稳定，例如使用数组索引，在插入、排序场景下可能导致节点状态错位。面试时最好明确说出：React 不做跨层最优树编辑，它靠约定换复杂度。

---

### 题目 3：为什么说 key 不是“为了消除 warning”，而是影响正确性？

**考察要点**

- 是否理解组件实例复用

**参考答案**

`key` 决定了 React 如何识别同级节点是否是“同一个实体”。如果 key 不稳定，React 可能把旧组件实例错误复用到新数据上，导致输入框内容错位、本地状态混乱、动画异常。  

因此 `key` 首先是正确性问题，其次才是性能问题。最佳实践是使用稳定、唯一、与业务实体绑定的 id。

---

### 题目 4：Fiber 架构出现的原因是什么？

**考察要点**

- 是否理解同步渲染的瓶颈

**参考答案**

早期 React 渲染过程基本是同步递归执行，一旦树足够大，长任务会阻塞主线程，影响输入响应。Fiber 的引入是为了把渲染工作拆成可中断、可恢复、可分优先级调度的单元。  

```text
Old Stack Reconciler: 一次递归到底，不可中断
Fiber Reconciler:     拆成小任务，可暂停/继续/丢弃
```

这为时间切片（time slicing）、并发渲染（concurrent rendering）、优先级更新打下了基础。

---

### 题目 5：Fiber 中的 render phase 和 commit phase 有什么区别？

**考察要点**

- 是否理解可中断与不可中断边界

**参考答案**

Render phase 负责计算新 Fiber 树、收集副作用，它可以被打断、重试、丢弃；commit phase 负责把结果真正应用到 DOM，并触发生命周期和布局相关副作用，它必须尽快且不可中断。  

所以面试官问你“为什么不要在 render 里做副作用”，根本原因就是 render phase 不是一次性的，可能执行多次。

---

### 题目 6：类组件生命周期和函数组件 Hooks 如何对应？

**考察要点**

- 是否能做现代化映射

**参考答案**

类组件常见生命周期包括 `componentDidMount`、`componentDidUpdate`、`componentWillUnmount`；在函数组件中，通常由 `useEffect` 统一表达副作用生命周期。  

```tsx
useEffect(() => {
  subscribe();
  return () => unsubscribe();
}, []);
```

但要注意，这种对应不是一一机械映射。函数组件没有“实例生命周期”这一概念，而是每次渲染形成新的闭包快照。

---

### 题目 7：`useState` 的更新为什么有时看起来是异步的？

**考察要点**

- 是否理解批处理和调度

**参考答案**

`setState` 或 `setCount` 调用后，React 不会立刻同步改变量再立刻重渲染，而是把更新放进队列，由调度器合并处理，因此开发者观察上常觉得“异步”。  

```tsx
setCount(count + 1);
setCount(count + 1);
```

上面两次可能都基于同一个旧值。React 这样设计是为了批处理更新，减少不必要渲染，并配合优先级调度。

---

### 题目 8：为什么函数式更新（functional updates）能解决部分状态竞争问题？

**考察要点**

- 是否知道读取旧快照的风险

**参考答案**

函数式更新会把“前一个状态”作为参数传入，因此多个更新可以基于最新队列值顺序计算。  

```tsx
setCount(c => c + 1);
setCount(c => c + 1);
```

这样结果是加 2，而不是加 1。面试里要强调：函数式更新不是语法偏好，而是在异步批处理和闭包场景下更安全。

---

### 题目 9：什么是 lazy initialization（惰性初始化）？

**考察要点**

- 是否会写 `useState(() => expensive())`

**参考答案**

如果初始值计算成本较高，可以把函数传给 `useState`，React 只会在初次渲染时执行。  

```tsx
const [state] = useState(() => createInitialState());
```

这能避免每次渲染都重复执行重计算。面试追问时可以补充：它只影响初次初始化，不会影响后续更新。

---

### 题目 10：`useEffect` 的执行时机是什么？cleanup（清理）何时发生？

**考察要点**

- 是否知道提交后执行
- 是否理解清理顺序

**参考答案**

`useEffect` 会在浏览器完成绘制后异步执行，不阻塞页面展示；如果 effect 依赖变化，React 会先执行上一次 effect 的 cleanup，再执行新的 effect；组件卸载时也会执行 cleanup。  

```tsx
useEffect(() => {
  const timer = setInterval(tick, 1000);
  return () => clearInterval(timer);
}, []);
```

面试里要强调：cleanup 不只是“卸载时”，依赖变化时同样会发生。

---

### 题目 11：`useEffect` 的依赖数组最常见的坑是什么？

**考察要点**

- 闭包陷阱
- 对象/函数引用变化

**参考答案**

最常见的坑有三类：第一，漏依赖导致读取到陈旧值（stale value）；第二，把每次渲染都会创建的新对象、新函数放进依赖，导致 effect 频繁重跑；第三，为了“消 warning”错误地关闭 lint 规则，埋下逻辑 bug。  

真正正确的思路不是“怎么骗过依赖数组”，而是重新组织代码：把可变值放进状态、把稳定函数抽出来、必要时使用 `useRef`。

---

### 题目 12：什么是 stale closure（陈旧闭包）？

**考察要点**

- 是否理解函数组件每次渲染生成新作用域

**参考答案**

函数组件每次渲染都会创建新的变量绑定，事件回调、定时器回调、effect 回调都可能捕获旧渲染中的值。  

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // 可能是旧值
  }, 1000);
  return () => clearInterval(id);
}, []);
```

解决方式包括：把依赖补全、使用函数式更新、或把最新值同步到 `useRef`。这道题本质上在考你是否理解 React 的“渲染即闭包快照”模型。

---

### 题目 13：`useMemo` 和 `useCallback` 什么时候该用，什么时候不该用？

**考察要点**

- 是否知道它们不是“默认优化”

**参考答案**

`useMemo` 用于缓存计算结果，`useCallback` 用于缓存函数引用。但它们本身也有维护和比较成本，所以不应机械滥用。  

适合使用的场景包括：重计算非常昂贵、引用稳定性会影响子组件 `memo`、作为依赖项必须稳定；不适合使用的场景包括：计算本身很便宜、组件本来很小、只是为了“显得专业”而胡乱包一层。  

React 19 时代还要补充一个点：React Compiler（React Compiler）会自动处理部分记忆化场景，因此工程优化策略会变化。

---

### 题目 14：React Compiler 出现后，`useMemo` / `useCallback` 会消失吗？

**考察要点**

- 是否理解自动优化不是万能

**参考答案**

不会消失，但很多样板式记忆化可能减少。React Compiler 目标是自动分析组件纯度和依赖，帮开发者省去部分手工优化。然而它不是对所有动态行为都能完美推断，也不意味着业务方可以完全不关心渲染模型。  

面试时比较成熟的说法是：未来优化重心会从“手搓 memo”转向“写更纯、更可分析的组件”。

---

### 题目 15：`useRef` 有哪些典型用途？

**考察要点**

- DOM 引用
- 可变容器

**参考答案**

`useRef` 有两大典型用途。第一，引用 DOM 节点或组件暴露的实例能力；第二，保存跨渲染持久存在、但变化不触发重渲染的可变值，比如定时器 id、上一次值、最新回调。  

```tsx
const latestValueRef = useRef(value);
latestValueRef.current = value;
```

要注意：`ref.current` 的变化不会触发 UI 更新，因此不应把它当成状态替代品。

---

### 题目 16：什么是 callback ref（回调 ref）？什么时候比对象 ref 更合适？

**考察要点**

- 是否知道 ref 附着/释放时机

**参考答案**

回调 ref 是一个函数，在节点挂载和卸载时会被 React 调用。它适合需要在 ref 变化瞬间执行逻辑的场景，例如测量 DOM、与第三方库集成。  

```tsx
const setNode = useCallback((node: HTMLDivElement | null) => {
  if (node) {
    console.log(node.getBoundingClientRect());
  }
}, []);
```

相比对象 ref，回调 ref 更灵活，但也更容易因为函数引用变化导致重复调用。

---

### 题目 17：`forwardRef` 和 `useImperativeHandle` 分别解决什么问题？

**考察要点**

- 是否理解组件封装与受控暴露

**参考答案**

`forwardRef` 让父组件可以把 ref 透传给子组件；`useImperativeHandle` 则让子组件决定只暴露哪些命令式能力，而不是把整个内部 DOM 结构泄漏出去。  

```tsx
type InputHandle = {
  focus: () => void;
};

const FancyInput = React.forwardRef<InputHandle>((props, ref) => {
  const innerRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus() {
      innerRef.current?.focus();
    }
  }));

  return <input ref={innerRef} />;
});
```

这是“封装内部实现，同时提供最小必要命令式接口”的典型模式。

---

### 题目 18：自定义 Hook（custom hooks）的设计原则有哪些？

**考察要点**

- 是否知道逻辑复用而非视图复用

**参考答案**

自定义 Hook 的本质是抽取状态逻辑和副作用逻辑，而不是抽取 JSX 结构。设计时要注意：命名以 `use` 开头；内部仍遵守 Hooks Rules；对外暴露稳定、最小的 API；避免把过多业务耦合塞进一个万能 Hook。  

```tsx
function useDebouncedValue<T>(value: T, delay: number) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}
```

---

### 题目 19：如何测试自定义 Hook？

**考察要点**

- `renderHook`
- 关注行为而非实现

**参考答案**

可以使用 React Testing Library 提供的 `renderHook` 来渲染 Hook，并通过 `act` 驱动状态变化。  

```tsx
import { renderHook, act } from "@testing-library/react";

const { result } = renderHook(() => useCounter());
act(() => result.current.inc());
expect(result.current.count).toBe(1);
```

重点是测试外部可观察行为：输出值、暴露方法、副作用结果，而不是测试内部使用了几个 `useState`。

---

### 题目 20：Redux Toolkit、Zustand、Jotai、Recoil、React Context 应该怎么选？

**考察要点**

- 状态管理选型
- 优缺点比较

**参考答案**

可以从“规模、心智负担、调试能力、更新粒度”四个维度回答：  

```text
方案            适用场景                     优点                         缺点
Redux Toolkit   大中型团队/复杂业务          规范强、生态成熟、DevTools好   样板仍比轻量库多
Zustand         中小型到中型业务             API 简洁、上手快               约束较弱
Jotai           原子化状态、局部组合         粒度细、组合自然               体系认知门槛稍高
Recoil          复杂依赖图（近年热度下降）   原子/selector 模型强           生态相对弱
Context         低频全局共享                 原生、无依赖                   高频更新易引发重渲染
```

面试里最忌讳“一句话站队”。正确回答是：没有绝对最优，关键看团队复杂度和状态规模。

---

### 题目 21：React Server Components（RSC）和 Client Components 的根本区别是什么？

**考察要点**

- 是否理解执行环境和能力边界

**参考答案**

Server Component 在服务端执行，可以直接访问服务器资源、数据库、私有密钥，不会把全部组件逻辑打进浏览器 bundle；Client Component 在浏览器执行，能使用交互能力和 Hooks。  

`"use client"` 用于声明该文件需要客户端能力；`"use server"` 常用于 Server Actions。根本区别不是“写法不同”，而是“执行位置、可用能力、打包边界”不同。

---

### 题目 22：Streaming SSR（流式服务端渲染）、Suspense、`React.lazy` 之间是什么关系？

**考察要点**

- 是否理解渐进式渲染

**参考答案**

Streaming SSR 允许服务端分块输出 HTML，而不是等整页都准备好；Suspense 提供了声明式“等待边界”；`React.lazy` 则是组件级代码分割的一个入口，常与 Suspense 配合。  

```tsx
const UserPanel = React.lazy(() => import("./UserPanel"));

<Suspense fallback={<Spinner />}>
  <UserPanel />
</Suspense>
```

在现代 Next.js 中，Suspense 的价值不只体现在前端 lazy loading，也体现在服务端数据流式返回。

---

### 题目 23：Next.js App Router 相比 Pages Router 的核心变化是什么？

**考察要点**

- 文件路由
- layout
- 服务端优先

**参考答案**

App Router 引入了基于 `app/` 目录的文件路由体系，天然支持嵌套 `layout`、`loading`、`error` 边界，并深度整合 React Server Components。  

```text
app/
├─ layout.tsx
├─ page.tsx
├─ dashboard/
│  ├─ layout.tsx
│  ├─ loading.tsx
│  ├─ error.tsx
│  └─ page.tsx
```

它推动的不是语法变化，而是渲染架构变化：默认更偏服务端获取数据，客户端只承担必要交互。

---

### 题目 24：Next.js 中 SSG、SSR、ISR 应该怎么理解？

**考察要点**

- 不同渲染模式权衡

**参考答案**

SSG（Static Site Generation）是构建时生成静态页面，性能最好，适合内容相对稳定场景；SSR（Server Side Rendering）是请求时生成页面，适合个性化和实时性要求高的场景；ISR（Incremental Static Regeneration）则是在静态基础上按时间窗口或触发条件重新生成。  

面试时建议从“性能、实时性、缓存、成本”四个维度做权衡，而不是只背缩写。

---

### 题目 25：Server Actions 的价值是什么？

**考察要点**

- 是否理解前后端边界压缩

**参考答案**

Server Actions 允许前端组件直接调用服务端定义的动作函数，省去部分手写 API route 的样板代码。它的价值在于简化数据修改路径、靠近组件组织逻辑，并利用框架帮你处理序列化和网络边界。  

但它并不意味着后端不重要，复杂权限、事务、一致性要求高的逻辑仍需要稳定的服务层设计。

---

### 题目 26：React 性能优化有哪些高频抓手？

**考察要点**

- 是否有体系化方法

**参考答案**

高频抓手包括：减少无效渲染（`React.memo`、稳定 props、拆分组件）；减少计算开销（`useMemo`）；减少函数引用变化（`useCallback`）；合理使用 `key`；代码分割（code splitting）和懒加载；列表虚拟化（virtualization）；借助 React DevTools Profiler 定位瓶颈。  

面试最加分的点是：先测量，再优化，而不是拍脑袋乱加 memo。

---

### 题目 27：什么场景适合列表虚拟化（virtualized lists）？

**考察要点**

- 是否理解 DOM 数量带来的压力

**参考答案**

当列表项数量很大，真实 DOM 数量、布局计算、内存占用都会成为瓶颈，此时适合用 `react-window`、`react-virtual` 之类的库只渲染可视区域。  

```text
Viewport
┌────────────────────┐
│ item 101           │
│ item 102           │
│ item 103           │
└────────────────────┘

真实数据 1..100000
实际渲染只覆盖当前窗口附近
```

但要注意虚拟列表会增加滚动定位、动态高度、无障碍支持的复杂度。

---

### 题目 28：React Testing Library 的核心哲学是什么？

**考察要点**

- test behavior, not implementation

**参考答案**

React Testing Library 强调“从用户视角测试行为，而不是实现细节”。这意味着你应该优先通过文本、角色（role）、标签、可交互行为来查询元素，而不是依赖组件实例或内部 state。  

```tsx
screen.getByRole("button", { name: "提交" });
```

这样的测试更稳健，因为只要用户能感知的行为不变，内部实现重构不会轻易把测试打碎。

---

### 题目 29：MSW（Mock Service Worker） 为什么比手写 fetch mock 更适合前端测试？

**考察要点**

- 是否理解网络层模拟

**参考答案**

MSW 在网络层拦截请求，模拟得更接近真实环境，因此你的测试代码不用绑定某个具体请求库实现。无论组件内部用 `fetch`、Axios 还是封装 client，只要最终发出网络请求，都能被统一模拟。  

这让测试更像“集成测试”，也更能暴露真实交互问题。

---

### 题目 30：Vitest 和 Jest 怎么选？

**考察要点**

- 工具链匹配

**参考答案**

如果项目使用 Vite 生态，Vitest 往往启动更快、配置更轻、与 ESM 支持更顺滑；Jest 生态成熟、插件丰富、历史包袱多但资料也多。  

选择时主要看现有工程体系、团队经验和兼容需求，而不是谁“更高级”。在新前端项目里，Vitest 已经非常有竞争力。

---

### 题目 31：CSS Modules、Tailwind CSS、styled-components、CSS-in-JS 各自适合什么场景？

**考察要点**

- 样式方案选型

**参考答案**

CSS Modules 适合中型项目，隔离性好，接近原生 CSS；Tailwind CSS 适合设计系统稳定、追求开发效率的团队；`styled-components` 代表的 CSS-in-JS 适合动态样式表达力强的场景，但运行时开销要关注。  

现代趋势是：如果动态样式需求不重，优先选择零或低运行时成本方案，比如 CSS Modules、Tailwind、vanilla-extract。

---

### 题目 32：前端无障碍（accessibility）面试最常考什么？

**考察要点**

- ARIA
- 键盘可达
- 语义化

**参考答案**

最常考的包括：使用语义化标签代替无意义 `div`；正确设置 ARIA roles 和属性；保证键盘导航顺序合理；焦点可见；表单控件有 label；模态框要做焦点陷阱和关闭逻辑；图片有 `alt` 文本。  

一个成熟回答应该体现：无障碍不是“补几个 aria 属性”，而是从结构、交互、阅读器体验共同考虑。

---

## 三、实战代码：一个更安全的异步数据 Hook

```tsx
import { useEffect, useState } from "react";

type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

export function useAsyncData<T>(loader: () => Promise<T>) {
  const [state, setState] = useState<AsyncState<T>>({ status: "idle" });

  useEffect(() => {
    let cancelled = false;
    setState({ status: "loading" });

    loader()
      .then(data => {
        if (!cancelled) setState({ status: "success", data });
      })
      .catch(error => {
        if (!cancelled) setState({ status: "error", error });
      });

    return () => {
      cancelled = true;
    };
  }, [loader]);

  return state;
}
```

这个例子能串联多个面试点：可辨识联合、`useEffect` cleanup、竞态取消、状态建模、泛型 Hook。

---

## 四、前端渲染模式总览图

```text
用户请求
   │
   ├─ CSR: 返回壳 + JS -> 浏览器拉数据 -> 渲染
   ├─ SSR: 服务端取数 -> 生成 HTML -> 浏览器 hydrate
   ├─ SSG: 构建期生成 HTML -> CDN 直出
   └─ ISR: 静态页面 + 定时/按需再生
```

面试时如果你能把 React 原理题和 Next.js 渲染模式题串起来，回答会显得更有体系。

---

## 五、补充高频追问

### 题目 33：Error Boundary（错误边界）能捕获什么，不能捕获什么？

**考察要点**

- 是否理解渲染错误边界

**参考答案**

错误边界主要用于捕获其子树在渲染、生命周期和构造阶段抛出的错误，并渲染降级 UI。但它不能捕获事件处理函数中的错误、异步回调错误、服务端渲染错误，也不能捕获它自身内部抛出的错误。  

面试时要明确：错误边界是“渲染树容错机制”，不是全局异常处理的万能方案。全局异常、Promise rejection、网络错误仍要配合监控系统统一上报。

---

### 题目 34：React Context 为什么既常用又容易被滥用？

**考察要点**

- 是否理解传播范围与重渲染成本

**参考答案**

Context 非常适合传递低频变化的全局信息，比如主题、语言、当前登录用户、表单上下文；但如果把高频变化的大状态对象直接放进 Context，任何消费该 Context 的组件都可能跟着重渲染。  

因此 Context 更像“依赖注入容器”，而不是通用状态管理银弹。成熟做法通常是：把不常变的配置放 Context，把高频业务状态交给更细粒度的 store。

---

### 题目 35：Hydration（激活）不匹配为什么会发生？怎么排查？

**考察要点**

- SSR/CSR 一致性

**参考答案**

Hydration mismatch 的本质是“服务端输出的 HTML 与客户端首次渲染结果不一致”。常见原因包括：在渲染阶段读取 `window`、`localStorage`、时间戳、随机数；服务端和客户端语言环境不同；条件渲染依赖浏览器特有状态。  

排查思路是先定位不一致节点，再检查首次渲染是否依赖非确定性输入。解决方式通常是把这些逻辑推迟到 `useEffect`，或通过服务端注入稳定初始数据。

---

### 题目 36：为什么说“把一切都做成受控组件”不一定最优？

**考察要点**

- 表单性能与心智负担

**参考答案**

受控组件（controlled components）让 React 完全接管输入值，优点是状态统一、校验清晰；但在大型表单、频繁输入、复杂富文本场景下，过度受控可能带来重渲染压力和大量样板代码。  

因此真实工程中常采用折中方案：关键字段和业务状态受控，普通原生输入可配合表单库做更轻量管理。面试官想听到的是“理解取舍”，而不是教条式地说某种方式永远最好。

---

### 题目 37：如何理解“组件设计中的状态下沉与状态提升”？

**考察要点**

- 是否具备组件建模能力

**参考答案**

状态提升（lifting state up）是把多个子组件共享的状态移到最近公共父级；状态下沉则是避免把只影响局部的状态放得过高，导致无关组件跟着重渲染。  

好的组件设计不是一味提升，而是让状态停留在“刚好足够共享”的层级。很多性能问题，本质上不是 memo 不够，而是状态放错了位置。

---

### 题目 38：如果面试官问“React 最容易被误解的地方是什么”，你会怎么答？

**考察要点**

- 是否有体系化总结

**参考答案**

一个比较有深度的回答是：很多人把 React 理解成“一个会自动优化 DOM 的库”，但更准确的理解应该是“一个声明式 UI 状态机框架”。误解往往来自三处：把渲染当成命令式过程、把 Hooks 当成生命周期替代语法糖、把性能优化等同于乱加 `useMemo`。  

如果你能把这三个误区说出来，再结合自己项目中的例子，面试官通常会认为你对 React 的理解已经超出 API 层。

---

## 六、React 面试速答模板

当面试官问到某个 React 机制时，你可以按以下顺序组织回答：

1. 先下定义：它是什么。  
2. 再讲原因：它为什么存在。  
3. 再讲边界：什么时候会出问题。  
4. 最后讲实践：项目里如何用、如何排查。  

例如回答 `useEffect` 时，不要只说“副作用 Hook”，而要补充“提交后执行、依赖变化先清理再重跑、闭包容易陈旧、尽量让 effect 只同步外部系统”。这种结构化表达，比零散背知识点更像真正做过复杂前端的人。

---

## 七、场景化补充题

### 题目 39：如果一个页面首屏很慢，你会如何用 React 视角排查？

**考察要点**

- 是否有实战排查方法

**参考答案**

我会先区分问题是网络慢、JavaScript 执行慢、渲染慢，还是 hydration 慢。然后使用浏览器 Performance 面板和 React DevTools Profiler 看首屏阶段哪些组件渲染耗时异常、是否存在重复渲染、是否 bundle 过大、是否把非首屏组件同步加载了。接着再决定是否做代码分割、服务端预取、状态下沉、列表虚拟化或去掉无效记忆化。  

这种回答的关键在于“先定位瓶颈，再做 React 级优化”，而不是一上来就说 `useMemo`。

---

### 题目 40：React 面试中，怎样回答才能体现“我不只是会写页面”？

**考察要点**

- 是否具备工程化视角

**参考答案**

你需要把回答从“组件 API”提升到“渲染模型、状态边界、工程权衡”。例如讲 Hooks 时要谈闭包和副作用边界，讲状态管理时要谈选型约束，讲性能时要谈测量方法，讲 Next.js 时要谈服务端和客户端职责拆分，讲测试时要谈行为驱动。  

换句话说，面试官不是在招一个只会把设计稿还原出来的人，而是在找一个能长期维护复杂前端系统的工程师。

补一句非常实用的话：如果你在回答中能把“原理、边界、排查、取舍”四个层次讲全，哪怕某个 API 细节没背到最细，整体评价通常也会比只会罗列名词更高。

这也是为什么真正准备 React 面试时，不能只刷八股题答案，而要结合自己做过的页面、性能问题、状态管理取舍和线上故障复盘去组织语言。

---

## 本章要点

- React 面试重点在于解释渲染模型，而不是背 API。
- Virtual DOM、diff、Fiber、批处理更新是同一条主线上的不同层次。
- Hooks 的核心难点是闭包快照、副作用边界、引用稳定性。
- 状态管理没有银弹，选型要看复杂度和团队约束。
- RSC、App Router、Server Actions 代表的是前端渲染架构演进。
- 性能优化必须以测量为前提，避免机械 memo。
- 测试、样式、无障碍往往是拉开高级前端候选人差距的部分。

## 延伸阅读

- React 官方文档
- React RFC / React Compiler 文档
- Next.js App Router 文档
- React Testing Library 文档
- Web.dev 关于性能与可访问性文章
