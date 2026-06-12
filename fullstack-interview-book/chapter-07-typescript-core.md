# 第七章：TypeScript 类型系统深度

TypeScript 的核心竞争力，不是“给 JavaScript 加类型”这么简单，而是用一套足够强大的静态类型系统（static type system）帮助工程师把大量错误前移到编译期。前端面试里，TypeScript 早已不是加分项，而是默认要求。本章围绕“类型系统能表达什么、编译器如何推断、工程中如何落地”展开，用高频面试题把核心知识点串起来。

---

## 一、先建立整体认知

很多候选人学 TypeScript 时是“语法碎片化”的：知道 `interface`、知道泛型（generics）、知道 `keyof`，但一到面试官追问“为什么这样设计”“类型系统怎样组合”就容易卡住。你需要先把它看成一棵树：

```text
TypeScript 类型系统
├─ 基础类型
│  ├─ primitive / literal
│  ├─ union / intersection
│  └─ narrowing
├─ 泛型
│  ├─ 约束 extends
│  ├─ 默认类型
│  └─ 推断 inference
├─ 类型编程
│  ├─ conditional types
│  ├─ mapped types
│  ├─ template literal types
│  └─ infer
├─ 工程能力
│  ├─ .d.ts / declare
│  ├─ decorators
│  └─ strict flags
└─ 实用抽象
   ├─ utility types
   ├─ type guard
   └─ interface vs type
```

---

## 二、高频面试题

### 题目 1：TypeScript 为什么比纯 JavaScript 更适合大型项目？

**考察要点**

- 是否理解静态类型检查的价值
- 是否能从协作、重构、维护成本角度回答

**参考答案**

TypeScript 的核心价值有四点。第一，静态类型检查（static type checking）能把很多原本运行时才暴露的问题提前到编译期，比如属性不存在、参数类型错误、返回值不符合约束。第二，IDE 能基于类型信息提供自动补全、跳转定义、重构支持，显著提升研发效率。第三，在多人协作时，类型本身就是可执行文档，接口边界更清晰。第四，随着项目规模扩大，重构风险会急剧上升，而 TypeScript 能帮助你快速定位受影响的调用点。因此面试时要强调：TypeScript 不是为了“写得更麻烦”，而是为了在复杂系统中降低不确定性。

---

### 题目 2：TypeScript 中的基础类型（primitive types）有哪些？

**考察要点**

- 是否掌握常见原始类型
- 是否知道 `any`、`unknown`、`never` 的区别

**参考答案**

常见基础类型包括 `string`、`number`、`boolean`、`bigint`、`symbol`、`null`、`undefined`。此外还要特别注意三个面试高频类型：`any` 表示关闭类型检查，能赋值给任何类型，也能从任何类型接收值；`unknown` 表示“未知但安全”，赋值给它没问题，但使用前必须缩小类型；`never` 表示永远不可能出现的值，常见于抛错函数、死循环或穷尽性检查。  

```ts
let a: any = 1;
let b: unknown = JSON.parse("{}");

function fail(msg: string): never {
  throw new Error(msg);
}
```

工程上优先用 `unknown` 替代 `any`，因为 `unknown` 会强迫你做显式校验。

---

### 题目 3：字面量类型（literal types）是什么？有什么用？

**考察要点**

- 是否理解“值级别约束”
- 是否会结合联合类型表达有限状态

**参考答案**

字面量类型就是把某个具体值直接当成类型，例如 `"success"`、`404`、`true`。它的意义在于把原本宽泛的类型收窄为一个有限集合，从而表达业务状态。最典型的场景是状态机和配置项。  

```ts
type Status = "idle" | "loading" | "success" | "error";

function render(status: Status) {
  if (status === "loading") return "加载中";
  return status;
}
```

如果不用字面量类型，只写 `string`，那任何字符串都能传入，类型系统就失去约束力。

---

### 题目 4：联合类型（union types）和交叉类型（intersection types）分别是什么？

**考察要点**

- 是否理解 `|` 与 `&` 的语义差异
- 是否能说出使用场景

**参考答案**

联合类型 `A | B` 表示“值可以是 A，也可以是 B”；交叉类型 `A & B` 表示“值必须同时满足 A 和 B”。联合类型常用来表达多种输入情况，交叉类型常用来做能力组合。  

```ts
type Id = string | number;
type WithTime = { createdAt: Date };
type User = { id: number; name: string };
type TimedUser = User & WithTime;
```

要注意：交叉类型不是对象浅合并的运行时概念，而是静态层面的“同时满足”。如果两个同名属性类型冲突，可能得到 `never`。

---

### 题目 5：什么是类型缩小（type narrowing）？

**考察要点**

- 是否知道 TypeScript 会根据控制流分析类型
- 是否能举出 `if`、`switch`、守卫函数等例子

**参考答案**

类型缩小是指 TypeScript 根据代码中的条件判断、分支逻辑和控制流分析，把原本较宽的类型收窄成更具体的类型。  

```ts
function printId(id: string | number) {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return id.toFixed(0);
}
```

这里 `typeof id === "string"` 成立时，编译器自动把 `id` 缩小为 `string`。面试时可以强调：TypeScript 不是单纯的“声明式类型检查”，而是带有控制流分析（control flow analysis）的类型系统。

---

### 题目 6：`any`、`unknown`、`never` 三者在面试中如何区分？

**考察要点**

- 是否能说出赋值关系和使用限制

**参考答案**

`any` 是“完全放弃检查”，它会污染类型系统，往往沿着调用链扩散。`unknown` 是“类型安全的顶层类型”，任何值都能赋给它，但它不能直接赋给更具体的类型，也不能直接访问属性。`never` 是“底层类型”，表示不可能存在的值。  

```ts
let x: unknown = "hi";
// let y: string = x; // 报错

if (typeof x === "string") {
  const y: string = x;
}
```

工程原则通常是：能不用 `any` 就不用，优先 `unknown + type guard`。

---

### 题目 7：泛型（generics）的本质是什么？

**考察要点**

- 是否理解“类型参数化”
- 是否能区分泛型和联合类型

**参考答案**

泛型本质上是“把类型当作参数传入”，从而让函数、接口、类在复用逻辑时保持类型信息。联合类型强调“可能是多种情况之一”，泛型强调“同一套逻辑适配多种类型，但不同位置之间存在关联”。  

```ts
function identity<T>(value: T): T {
  return value;
}
```

如果用联合类型 `string | number`，你只能说“输入可能是两者之一”；但用泛型 `T`，你表达的是“输入是什么，输出就是什么”，这种关联性是联合类型做不到的。

---

### 题目 8：泛型函数、泛型接口、泛型类分别怎么写？

**考察要点**

- 基本语法是否熟练

**参考答案**

```ts
function wrap<T>(value: T): { value: T } {
  return { value };
}

interface ApiResponse<T> {
  code: number;
  data: T;
  message: string;
}

class Box<T> {
  constructor(public value: T) {}
  getValue(): T {
    return this.value;
  }
}
```

面试时建议补充：泛型函数更适合局部行为抽象，泛型接口适合描述数据结构，泛型类适合描述带状态的对象模型。

---

### 题目 9：泛型约束（generic constraints）中的 `extends` 有什么作用？

**考察要点**

- 是否知道约束不是继承语义
- 是否会写“至少有某些属性”的约束

**参考答案**

在泛型里，`extends` 表示“类型参数必须满足某种约束”，不一定是类继承。  

```ts
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}
```

这样 `string`、数组、带 `length` 属性的对象都能传入。面试追问时可以补一句：泛型约束的意义是让实现体安全地访问某些结构，同时保留调用侧的具体类型。

---

### 题目 10：泛型默认类型（default type parameters）是什么？

**考察要点**

- 是否知道 `<T = unknown>` 语法
- 是否理解默认值的工程用途

**参考答案**

泛型参数可以提供默认类型，调用方不传时使用默认值。  

```ts
interface Result<T = unknown, E = Error> {
  data?: T;
  error?: E;
}
```

它的作用是降低使用门槛，让常见场景更简洁，同时保留高级定制能力。在库设计中非常常见，例如 React 的一些类型定义、请求库返回类型封装等。

---

### 题目 11：TypeScript 的泛型推断（generic inference）是怎么工作的？

**考察要点**

- 是否理解调用点推断
- 是否知道复杂场景下可能推断失败

**参考答案**

当调用泛型函数时，编译器会根据实参、上下文类型和返回值位置反推类型参数。  

```ts
function first<T>(list: T[]): T | undefined {
  return list[0];
}

const n = first([1, 2, 3]); // T 被推断为 number
```

推断失败的常见场景包括：信息不足、多个候选冲突、类型被过度宽化。工程中如果发现推断不稳定，可以显式指定类型参数，或者通过 `as const` 保留更精确的信息。

---

### 题目 12：条件类型（conditional types）是什么？

**考察要点**

- 是否会读 `T extends U ? X : Y`
- 是否理解“按类型分支”

**参考答案**

条件类型允许我们像写三元表达式一样，在类型层面做判断。  

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<"abc">; // true
type B = IsString<number>; // false
```

它是 TypeScript 类型编程的基础能力，很多内置工具类型都依赖它实现。注意：这里的 `extends` 是“可赋值关系判断”，不是运行时判断。

---

### 题目 13：为什么说条件类型对联合类型会发生分发（distributive behavior）？

**考察要点**

- 是否理解“裸类型参数”分发

**参考答案**

当条件类型左侧是“裸类型参数” `T`，且传入联合类型时，TypeScript 会把联合类型拆开逐个判断，再把结果联合起来。  

```ts
type ToArray<T> = T extends any ? T[] : never;
type R = ToArray<string | number>; // string[] | number[]
```

这叫分发条件类型。它非常有用，比如实现 `Exclude`、`Extract`。如果你不想分发，可以把 `T` 包一层元组：`[T] extends [U] ? ... : ...`。

---

### 题目 14：`infer` 关键字的作用是什么？

**考察要点**

- 是否理解“在条件类型中声明待推断变量”

**参考答案**

`infer` 只能出现在条件类型中，用来从某个复杂结构中提取类型片段。  

```ts
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type R = GetReturnType<() => Promise<number>>; // Promise<number>
```

面试里要强调：`infer` 让类型系统具有“模式匹配”能力。你不是手动索引某个结构，而是在匹配成功时把其中一部分绑定为变量。

---

### 题目 15：请手写 `Exclude<T, U>`。

**考察要点**

- 是否掌握分发条件类型

**参考答案**

```ts
type MyExclude<T, U> = T extends U ? never : T;

type A = MyExclude<"a" | "b" | "c", "a" | "c">; // "b"
```

原理是：对 `T` 中每个成员判断是否可赋值给 `U`，如果是就丢弃为 `never`，否则保留。最后 `never` 会从联合类型中消失。

---

### 题目 16：请手写 `Extract<T, U>` 和 `NonNullable<T>`。

**考察要点**

- 是否能类比 `Exclude`

**参考答案**

```ts
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T extends null | undefined ? never : T;
```

`Extract` 是“保留交集”，`Exclude` 是“删除交集”；`NonNullable` 本质上就是去掉 `null | undefined`。这类题看似简单，实质在考你是否真的理解分发条件类型。

---

### 题目 17：映射类型（mapped types）是什么？

**考察要点**

- 是否知道 `[K in keyof T]`

**参考答案**

映射类型用于“遍历一个已有类型的键，再生成一个新类型”。  

```ts
type Flags<T> = {
  [K in keyof T]: boolean;
};
```

如果 `T` 是 `{ name: string; age: number }`，那么 `Flags<T>` 就是 `{ name: boolean; age: boolean }`。映射类型非常适合批量修改属性特征，比如全部变可选、全部变只读。

---

### 题目 18：`keyof`、`in` 在映射类型里分别表示什么？

**考察要点**

- 是否能解释语法而不是死记

**参考答案**

`keyof T` 会得到 `T` 的键名联合类型；`in` 则表示“遍历这个联合类型中的每一个成员”。  

```ts
type Clone<T> = {
  [K in keyof T]: T[K];
};
```

因此可以把映射类型理解为“类型层面的 `for...of`”。`K` 依次取 `T` 的每个键，然后 `T[K]` 读取该键对应的值类型。

---

### 题目 19：映射类型里的 `readonly` 和 `?` 修饰符怎么用？

**考察要点**

- 是否会添加和移除修饰符

**参考答案**

在映射类型中，`readonly` 和 `?` 都可以批量增删。  

```ts
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};
```

其中 `+?`、`-?`、`+readonly`、`-readonly` 都是合法写法。面试常见追问是：为什么 `Required<T>` 要用 `-?`？因为它要移除可选标记。

---

### 题目 20：`as` 子句在映射类型中有什么作用？

**考察要点**

- 是否了解 key remapping

**参考答案**

`as` 子句用于重映射键名（key remapping），让你在遍历键时顺便改名、过滤。  

```ts
type PrefixKeys<T> = {
  [K in keyof T as `api_${string & K}`]: T[K];
};
```

如果原类型是 `{ user: string }`，结果就是 `{ api_user: string }`。更进一步，若某个键映射成 `never`，它就会被过滤掉。

---

### 题目 21：模板字面量类型（template literal types）能解决什么问题？

**考察要点**

- 是否理解字符串类型组合

**参考答案**

模板字面量类型让我们可以在类型层面拼接字符串、约束命名格式、做简单模式匹配。  

```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
type A = EventName<"click">; // "onClick"
```

工程上常用于事件名、路由名、CSS 变量名、接口字段映射等。它把“约定”从文档变成编译期约束。

---

### 题目 22：`Uppercase`、`Lowercase`、`Capitalize`、`Uncapitalize` 这些工具类型有什么用？

**考察要点**

- 是否知道它们属于字符串转换工具

**参考答案**

这几个都是基于模板字面量类型提供的字符串大小写工具。  

```ts
type A = Uppercase<"hello">; // "HELLO"
type B = Lowercase<"Hello">; // "hello"
type C = Capitalize<"name">; // "Name"
type D = Uncapitalize<"Title">; // "title"
```

面试里可以结合实际场景回答，比如把后端字段转成前端事件名、把路由段拼成常量类型、自动生成 getter/setter 风格的 API。

---

### 题目 23：如何用模板字面量类型做字符串模式匹配？

**考察要点**

- 是否会结合 `infer`

**参考答案**

可以通过条件类型加模板字面量类型来提取字符串的一部分。  

```ts
type GetId<T extends string> = T extends `user:${infer Id}` ? Id : never;
type Id = GetId<"user:42">; // "42"
```

这类技巧常用于路由参数、事件命名规范、缓存 key 解析等。虽然它不是运行时解析，但可以让调用者在编译期遵守字符串协议。

---

### 题目 24：Type guard（类型守卫）有哪些形式？

**考察要点**

- `typeof`、`instanceof`、`in`、自定义守卫

**参考答案**

常见类型守卫有四类：`typeof` 用于原始类型；`instanceof` 用于类实例；`in` 用于判断对象是否具有某属性；自定义守卫函数通过返回值 `arg is Type` 告诉编译器。  

```ts
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every(item => typeof item === "string");
}
```

面试时不要只背语法，要说明其价值：把不安全的外部输入转换为安全的内部类型。

---

### 题目 25：`typeof`、`instanceof`、`in` 三种守卫分别适合什么场景？

**考察要点**

- 是否能区分使用边界

**参考答案**

`typeof` 主要用于 `string`、`number`、`boolean`、`bigint`、`symbol`、`function`、`undefined` 这些基础类型判断；`instanceof` 依赖原型链，适合类实例；`in` 适合区分对象结构。  

```ts
function format(input: Date | string) {
  if (input instanceof Date) return input.toISOString();
  return input.trim();
}
```

如果数据来自接口响应，通常更推荐结构化判断和自定义守卫，而不是 `instanceof`，因为远程 JSON 并没有类实例信息。

---

### 题目 26：什么是可辨识联合（discriminated unions）？

**考察要点**

- 是否知道公共字面量字段的重要性

**参考答案**

可辨识联合是指多个对象类型共享一个字面量标签字段，比如 `type`、`kind`、`status`，编译器可据此精准缩小类型。  

```ts
type Loading = { state: "loading" };
type Success = { state: "success"; data: string[] };
type ErrorState = { state: "error"; message: string };
type Result = Loading | Success | ErrorState;
```

它是前端状态建模的最佳实践之一，比“很多可选字段堆在一起”更安全，因为每种状态的数据边界是明确的。

---

### 题目 27：`interface` 和 `type` 的区别是什么？面试该怎么回答？

**考察要点**

- 是否能回答“语义区别”和“工程建议”

**参考答案**

`interface` 更偏向描述对象结构，支持声明合并（declaration merging）；`type` 能表示更广泛的类型别名，包括联合类型、交叉类型、元组、条件类型等。  

```ts
interface User {
  id: number;
}

type Status = "idle" | "done";
```

如果只是对象契约，团队常优先 `interface`，因为可读性强、可扩展；如果需要组合、映射、条件判断，通常用 `type`。面试里不要说“哪个绝对更好”，而要说“看表达目标”。

---

### 题目 28：什么是声明合并（declaration merging）？

**考察要点**

- 是否知道这是 `interface` 的特性之一

**参考答案**

同名 `interface` 会自动合并成员。  

```ts
interface Window {
  appVersion: string;
}

interface Window {
  track: (event: string) => void;
}
```

最终 `Window` 同时拥有两个属性。这在扩展全局对象、第三方库类型增强时很常见。但也要注意，滥用声明合并会让类型来源分散，增加维护成本。

---

### 题目 29：`extends` 和 `&` 在对象组合上有什么区别？

**考察要点**

- 是否理解继承接口与交叉类型的差异

**参考答案**

`interface A extends B` 更像显式层次化建模，语义清晰，适合面向对象风格接口扩展；`A & B` 则是类型层面的组合表达。多数简单对象场景下二者结果接近，但在复杂冲突、条件类型组合、联合分发时行为不同。  

```ts
interface Base {
  id: number;
}

interface User extends Base {
  name: string;
}

type Admin = Base & { role: "admin" };
```

面试时可以总结：继承强调“is-a”，交叉强调“has-all-of”。

---

### 题目 30：请手写 `Partial`、`Required`、`Pick`、`Omit`、`Record`。

**考察要点**

- 映射类型是否熟练

**参考答案**

```ts
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};

type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type MyOmit<T, K extends keyof any> = MyPick<T, Exclude<keyof T, K>>;

type MyRecord<K extends keyof any, V> = {
  [P in K]: V;
};
```

这些题本质在考你是否理解：键从哪里来、值如何取、修饰符如何改、如何组合已有工具类型。

---

### 题目 31：请手写 `ReturnType`、`Parameters`、`Awaited`。

**考察要点**

- 是否掌握 `infer`

**参考答案**

```ts
type MyReturnType<T extends (...args: any[]) => any> =
  T extends (...args: any[]) => infer R ? R : never;

type MyParameters<T extends (...args: any[]) => any> =
  T extends (...args: infer P) => any ? P : never;

type MyAwaited<T> =
  T extends Promise<infer U> ? MyAwaited<U> : T;
```

其中 `Awaited` 需要递归展开 Promise，直到拿到最终值类型。面试官很喜欢借此考你对递归条件类型和 `infer` 的理解。

---

### 题目 32：如何实现 `DeepPartial` 和 `DeepReadonly`？

**考察要点**

- 是否会递归类型
- 是否考虑对象与基础类型的边界

**参考答案**

```ts
type DeepPartial<T> =
  T extends Function ? T :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;

type DeepReadonly<T> =
  T extends Function ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;
```

关键点有两个：第一，递归只对对象继续展开；第二，函数通常应原样保留，否则会破坏可调用签名。实际工程里还可能进一步区分数组、Map、Set。

---

### 题目 33：Stage 3 装饰器（decorators，TS 5.0+）和旧版实验装饰器有什么区别？

**考察要点**

- 是否了解新装饰器语义变化

**参考答案**

TypeScript 5.0 开始支持接近标准的 Stage 3 装饰器。它和早期 `experimentalDecorators` 最大区别在于：调用约定不同、上下文对象不同、与标准对齐更好。新装饰器更强调返回新的类或方法定义，而不是直接操作 descriptor 的旧模式。  

```ts
function sealed(value: Function, context: ClassDecoratorContext) {
  console.log("decorating", context.name);
}

@sealed
class UserService {}
```

面试回答时要点是：新标准更利于生态统一，但很多老项目仍基于旧语法，因此迁移时要关注 Babel、TS 配置和框架支持。

---

### 题目 34：类装饰器、方法装饰器、参数装饰器分别适合做什么？

**考察要点**

- 是否能说出实际用途

**参考答案**

类装饰器常用于注册依赖注入（dependency injection）元数据、添加静态能力；方法装饰器适合日志、鉴权、缓存、埋点；参数装饰器适合标记参数来源或注入规则。  

```ts
function logMethod(target: Function, context: ClassMethodDecoratorContext) {
  return function (this: unknown, ...args: unknown[]) {
    console.log("call", String(context.name), args);
    return target.apply(this, args);
  };
}

class Api {
  @logMethod
  fetch(id: number) {
    return id;
  }
}
```

不过要强调：装饰器是增强工具，不应滥用，否则会让控制流变得隐式，增加理解成本。

---

### 题目 35：`declare`、环境声明（ambient types）、`.d.ts` 文件分别是什么？

**考察要点**

- 是否理解“只声明类型，不产生运行时代码”

**参考答案**

`declare` 用于告诉编译器“这个值/类型在别处存在，请只做类型检查，不要生成实现代码”。`.d.ts` 是专门放声明的文件，通常用来给 JS 库补类型、描述全局变量或模块接口。  

```ts
declare const __APP_VERSION__: string;

declare module "legacy-sdk" {
  export function init(token: string): void;
}
```

面试时可以总结：`.d.ts` 的本质是“类型世界的头文件”。

---

### 题目 36：三斜线指令（triple-slash directives）是什么？现在还常用吗？

**考察要点**

- 是否知道 `/// <reference ... />`

**参考答案**

三斜线指令是早期 TypeScript 用于声明依赖关系和类型引用的机制，例如：

```ts
/// <reference path="./types.d.ts" />
/// <reference types="node" />
```

现代项目更常依赖 `tsconfig.json`、ESM 模块系统和包管理器自动解析，因此三斜线指令使用频率下降，但在某些声明文件、旧项目兼容场景里仍然会出现。

---

### 题目 37：`strictNullChecks` 为什么重要？

**考察要点**

- 是否理解空值安全

**参考答案**

开启 `strictNullChecks` 后，`null` 和 `undefined` 不再能随意赋给其他类型，迫使你显式处理空值。  

```ts
function findName(list: string[]): string | undefined {
  return list[0];
}
```

如果不开启这个选项，很多潜在的空引用错误会被忽略。它是严格模式里最值得优先开启的选项之一。

---

### 题目 38：`noImplicitAny`、`strictFunctionTypes`、`strictBindCallApply` 分别解决什么问题？

**考察要点**

- 是否理解严格编译选项的实际意义

**参考答案**

`noImplicitAny` 防止编译器在信息不足时默默推断成 `any`；`strictFunctionTypes` 让函数参数位置按照更严格的变型规则检查，避免不安全赋值；`strictBindCallApply` 则让 `bind`、`call`、`apply` 的参数检查更精确。  

这些选项的共同目标是：减少“看起来能编译，运行时却出错”的灰色地带。大型项目里建议直接启用 `strict: true`，再按遗留包袱逐步收敛。

---

### 题目 39：面试中如果被问“TypeScript 类型系统最难的部分是什么”，你怎么答？

**考察要点**

- 是否具备体系化总结能力

**参考答案**

比较好的回答不是说某个语法最难，而是指出三个真正的难点：第一，类型推断和控制流分析是隐式发生的，很多错误来自“以为编译器懂你”；第二，条件类型、映射类型、模板字面量类型叠加后，类型层本身就是一门小语言，需要抽象能力；第三，工程中难点不是“会不会写一个工具类型”，而是“什么时候该抽象、什么时候保持简单”。如果你能把“语法层难点”和“工程层边界”都说出来，面试官通常会觉得你理解很成熟。

---

## 三、典型工程图：类型系统在前端项目中的位置

```text
接口响应(JSON)
     │
     ▼
运行时校验(zod/io-ts/自定义 guard)
     │
     ▼
领域类型(domain types)
     │
     ├─ UI Props
     ├─ API Client
     ├─ State Store
     └─ Utility Types
     │
     ▼
编译期约束 + IDE 提示 + 重构安全
```

这张图是面试加分点：真正成熟的 TypeScript 工程，不是“只有静态类型”，而是“运行时校验 + 编译期建模”双轮驱动。因为接口响应来自外部世界，TypeScript 再强也无法替代运行时验证。

---

## 四、面试回答策略总结

1. 回答定义题时，不要只背语法，要补一句“解决什么问题”。
2. 回答实现题时，先说原理，再写类型。
3. 回答工程题时，要强调“可维护性、边界、复杂度控制”。
4. 遇到 `infer`、条件类型、映射类型连环追问时，先从最小例子讲起，再扩展到工具类型。

---

## 五、补充高频追问

### 题目 40：`as const` 在 TypeScript 里到底做了什么？

**考察要点**

- 是否理解字面量收窄
- 是否知道只读推断

**参考答案**

`as const` 会让表达式尽可能保留最窄的字面量类型，并把对象属性和数组成员推断为只读。它经常用于配置表、事件常量、路由映射、Redux action 常量等场景。  

```ts
const routes = {
  home: "/",
  user: "/user/:id"
} as const;
```

此时 `routes.home` 的类型不是普通 `string`，而是字面量 `"/"`。很多候选人只记住“防止宽化”，却忽略它带来的只读语义，这在面试里很容易被追问。

---

### 题目 41：为什么说“对象字面量赋值检查”是 TypeScript 的一个重要细节？

**考察要点**

- excess property checks

**参考答案**

当你把对象字面量直接赋给一个目标类型时，TypeScript 会做额外属性检查（excess property checks），防止拼写错误或多传无意义字段。  

```ts
type User = { name: string };
const user: User = { name: "Ada", age: 18 }; // 报错
```

但如果这个对象先赋给一个中间变量，再传给目标类型，检查会宽松一些。面试官借此想看你是否理解：TypeScript 的类型检查不是单一规则，而是和“值出现的位置”有关。

---

### 题目 42：什么时候应该使用类型断言（type assertion），什么时候不应该？

**考察要点**

- 是否知道断言不是转换

**参考答案**

类型断言的作用是告诉编译器“我比你更清楚这个值是什么类型”，它不会产生运行时转换，因此应该谨慎使用。合适场景包括：DOM 查询结果收窄、框架 API 的已知上下文、第三方库类型不完善的临时兜底。不合适场景包括：为了绕过错误随手写 `as any`、对接口响应做未经校验的断言、用断言掩盖真实设计问题。  

成熟回答要强调：断言是最后手段，优先考虑缩小类型、写守卫函数或补充声明。

---

### 题目 43：`enum`、字符串字面量联合、`const enum` 各有什么取舍？

**考察要点**

- 是否了解运行时代码与类型表达差异

**参考答案**

`enum` 会生成一定运行时代码，适合需要值空间统一管理、与老式 OOP 风格靠近的项目；字符串字面量联合更轻量、与现代前端生态更贴合，也更适合和模板字面量类型组合；`const enum` 会在编译期内联，运行时更轻，但构建链兼容性要留意。  

在现代 React/Node 工程中，很多团队更偏向 `as const` 对象加联合类型，因为 tree shaking 更友好，类型表达也更灵活。

---

### 题目 44：如何用穷尽性检查（exhaustiveness check）提高状态机安全性？

**考察要点**

- `never` 的工程用途

**参考答案**

对于可辨识联合，推荐在 `switch` 的默认分支里利用 `never` 做穷尽性检查。  

```ts
function assertNever(x: never): never {
  throw new Error(`unexpected value: ${x}`);
}
```

如果未来新增了一种状态但忘记处理，编译器就会报错。面试时一定要说明：`never` 不只是理论类型，它在真实项目里能帮助我们把遗漏分支提前暴露出来。

---

### 题目 45：TypeScript 严格模式迁移老项目时，一般应该怎么推进？

**考察要点**

- 是否有工程迁移思路

**参考答案**

老项目启用严格模式不能一步到位，通常建议分阶段推进：先打开 `noImplicitAny` 和 `strictNullChecks`，因为这两个收益最大；然后补齐核心领域模型和公共工具函数的类型；再逐步收敛边缘模块、第三方声明和遗留 `as any`。同时要建立 lint 规则和 CI 门禁，防止新代码继续制造类型债务。  

面试里如果你能说出“严格模式是工程治理而不是单次开关”，会非常加分。

---

## 六、常见误区总结

1. 误把 `type` 和 `interface` 当成“二选一立场问题”，而不是表达能力差异。  
2. 误把泛型当成“支持多个类型”的写法，没有理解“类型之间的关联”。  
3. 误把条件类型里的 `extends` 当成传统继承。  
4. 误把类型断言当成类型转换。  
5. 误以为 TypeScript 足够强就不需要运行时校验。  

这些误区在初中级候选人里非常常见。面试时如果你能主动指出它们，并说明自己在项目里如何规避，往往比单纯背更多语法更能体现深度。

---

## 本章要点

- TypeScript 的核心价值是把错误前移到编译期，并为大型协作提供稳定边界。
- 联合类型适合表达“多种可能”，泛型适合表达“位置间关联”。
- 条件类型、映射类型、模板字面量类型共同构成类型编程基础。
- `infer` 是提取复杂结构中子类型的关键工具。
- `interface` 与 `type` 没有绝对优劣，关键看表达目标。
- 严格模式配置决定了类型系统能否真正发挥作用。
- `.d.ts`、`declare`、装饰器等能力体现的是 TypeScript 的工程化深度。

## 延伸阅读

- TypeScript Handbook
- TypeScript 5.x Release Notes
- Effective TypeScript
- tsconfig 参考文档
- React TypeScript Cheatsheets
