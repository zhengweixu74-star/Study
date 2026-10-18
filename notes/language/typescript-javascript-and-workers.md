# TypeScript、JavaScript 与 Worker 线程通信

[返回笔记目录](../README.md)

## 本文目录

- [TypeScript 与 JavaScript 的关系](#typescript-与-javascript-的关系)
- [类型标注、类型推断、接口与泛型](#类型标注类型推断接口与泛型)
- [类型擦除与运行时检查](#类型擦除与运行时检查)
- [线程、事件循环与异步](#线程事件循环与异步)
- [Worker 消息通信与回调执行位置](#worker-消息通信与回调执行位置)
- [与 Swift 的对应关系及边界](#与-swift-的对应关系及边界)

## TypeScript 与 JavaScript 的关系

TypeScript 在 JavaScript 的基础上增加静态类型检查和相关语法。静态类型检查是工具在执行代码之前，根据类型声明和代码结构检查值的使用是否符合要求。JavaScript 的值也有类型，但普通 JavaScript 变量没有固定的静态类型声明，可以先保存数字，再保存字符串。

JavaScript 运行环境是执行 JavaScript 代码并提供定时器、网络等能力的系统，例如浏览器或 Node.js。TypeScript 不提供独立于 JavaScript 的线程模型；代码移除类型语法后，由相应 JavaScript 运行环境执行。

JavaScript 语法通常也可以写在 TypeScript 中，但 TypeScript 可能对这些代码报告类型错误。语法兼容不等于所有 JavaScript 代码都能通过 TypeScript 检查。

## 类型标注、类型推断、接口与泛型

### 类型标注与类型推断

类型标注是在声明中显式写出允许使用的类型。类型推断是类型检查器根据初始值和使用位置推导类型，不要求每处都手写标注。

JavaScript 允许下面的赋值：

```javascript
let age = 20;
age = "hello";
```

TypeScript 中，显式标注与类型推断都可以约束后续赋值：

```typescript
let age: number = 20;
age = "hello"; // 类型检查报错：string 不能赋给 number

let count = 20; // 推断为 number
count = "hello"; // 类型检查报错
```

这里的报错来自静态类型检查，不是 TypeScript 为变量插入了运行时拦截器。Swift 的 `var age: Int = 20` 和 `var age = 20` 也分别体现显式标注与类型推断；JavaScript 的 `number` 与 Swift 的 `Int` 并不是同一种数值类型。

### 接口描述对象的结构

`interface` 是 TypeScript 中描述对象所需成员及其类型的声明。下面的 `User` 要求对象包含字符串 `name` 和数字 `age`。

```typescript
interface User {
    name: string;
    age: number;
}

const user: User = {
    name: "Tom",
    age: 20
};
```

`const` 表示变量绑定不能被重新赋值，不表示对象内部的属性自动不可变。这个例子仍允许 `user.age = 21`。

TypeScript 通常按结构判断对象是否符合接口，也就是检查所需成员是否匹配。接口不是用来创建对象的构造函数，运行时也不存在名为 `User` 的接口对象。接口在“描述成员要求”这一点上类似 Swift 的 `protocol`，但 Swift 的具体类型通常需要显式声明遵循协议；两者的类型匹配规则不同。

### 泛型保留输入与输出的类型关系

泛型是用类型参数描述可用于多种类型的代码。下面的 `T` 是类型参数；参数 `value` 与返回值都使用 `T`，表示函数返回与输入相同类型的值。

```typescript
function identity<T>(value: T): T {
    return value;
}

const numberValue = identity<number>(10);
const stringValue = identity("hello"); // 也可以推断类型参数
```

这个关系类似 Swift 的 `func identity<T>(_ value: T) -> T`。TypeScript 用类型参数保留静态检查所需的信息，不会因此为每个 `T` 自动创建新的线程或运行时类型对象。

## 类型擦除与运行时检查

类型擦除是执行前移除只用于静态检查的类型语法。类型标注、`interface` 和泛型类型参数不会直接保留在输出的 JavaScript 中。

```typescript
let age: number = 20;
```

移除类型标注后，对应代码是：

```javascript
let age = 20;
```

常见工具先将 TypeScript 转成 JavaScript。部分运行环境也能在加载 TypeScript 文件时移除受支持的类型语法。能够加载文件不等于执行了完整的静态类型检查，也不表示所有 TypeScript 语法都受支持。

类型声明不会自动校验网络响应或其他外部数据。`as User` 是类型断言，即要求类型检查器按 `User` 看待一个值；断言不会转换数据，也不会验证字段。如果外部数据必须符合约定，程序仍需在读取数据时检查实际字段。

类型擦除也不表示所有 TypeScript 特有语法都不产生 JavaScript。类型声明与需要生成运行代码的语法要区分，例如普通 `enum` 通常会生成相应的 JavaScript。

## 线程、事件循环与异步

线程是操作系统调度程序指令执行的一条路径。异步调用允许调用方在结果尚未准备好时继续执行，随后再接收结果；异步本身不保证创建新线程。

在浏览器页面的主线程上，JavaScript 的一段同步代码会先执行完，同一执行环境中的其他回调才能获得执行机会。浏览器本身还可以使用其他线程处理网络、渲染等工作，因此“页面 JavaScript 在主线程执行”不等于“浏览器只有一条线程”。

事件循环是运行环境持续选择待执行工作并安排回调执行的机制。回调是预先交给其他代码、等待指定事件发生后再被调用的函数。主线程长时间执行计算时，主线程上的其他回调和界面交互可能延迟。

| 机制 | 作用 | 是否自动把函数体移到新线程 |
| --- | --- | --- |
| `setTimeout` | 达到定时条件后安排回调，实际执行还需等待调度 | 否 |
| `Promise` | 表示操作的完成结果或失败原因，并登记后续处理函数 | 否 |
| `async` / `await` | 用异步函数和等待表达式组织依赖异步结果的代码 | 否 |
| 浏览器 `Worker` | 创建独立的工作执行环境，在工作线程运行脚本 | 是，指定脚本由 Worker 执行 |
| Node.js `worker_threads` | 为 Node.js 提供 Worker 线程能力 | 是，具体接口由 Node.js 提供 |

`new Promise(...)` 传入的执行函数会立即同步执行。`async` 函数在首次执行到 `await` 之前也会同步运行。把耗时循环放进这两个位置，不会自动避免主线程阻塞。

`await` 暂停当前异步函数后续代码，等待结果再继续；`await` 不会阻塞整个线程等待结果，也不是切换后台线程的指令。`setTimeout(..., 0)` 同样不是立即执行：当前同步代码要先结束，回调还要等待调度。

## Worker 消息通信与回调执行位置

### 两个脚本与一次消息往返

下面以浏览器页面创建专用 Worker 为例。专用 Worker 是由创建方使用的独立脚本执行环境。主线程保留一个 `Worker` 对象，用该对象发送消息和接收结果；Worker 内的 `self` 指向 Worker 自己的全局对象，即该执行环境提供全局属性和方法的对象。

页面主线程中的 `main.js`：

```javascript
const worker = new Worker("./worker.js");

worker.onmessage = (event) => {
    console.log("计算结果:", event.data);
};

worker.postMessage(100);
```

Worker 线程中的 `worker.js`：

```javascript
self.onmessage = (event) => {
    const result = event.data * 2;
    self.postMessage(result);
};
```

这两个示例使用 JavaScript；TypeScript 转换后也使用相同的 Worker 接口。示例假设浏览器可以成功加载 `worker.js`，并约定发送的消息是数字。

执行顺序如下：

1. 页面主线程创建 `Worker` 对象，浏览器加载 Worker 脚本。页面主线程设置接收结果的函数。
2. 页面主线程调用 `worker.postMessage(100)`，把数字 `100` 发送给 Worker，不同步等待结果。
3. Worker 的事件循环派发消息事件，并调用 `self.onmessage` 中保存的函数。浏览器传入的 `event.data` 是 `100`。
4. Worker 线程计算得到 `200`，再调用 `self.postMessage(200)` 发回结果。
5. 页面主线程的事件循环派发返回的消息事件，并调用 `worker.onmessage` 中保存的函数。主线程上的 `console.log` 输出结果 `200`。

`postMessage` 发送消息，不会直接跨线程调用对方的函数体。发送与接收之间可能存在调度延迟，不能把“发送完成”理解为“接收回调已经执行”。

### onmessage 赋值与箭头函数语法

```javascript
worker.onmessage = (event) => {
    console.log("计算结果:", event.data);
};
```

`worker.onmessage` 是 `worker` 对象上的消息处理属性。`=` 把右侧函数保存到该属性。`(event) => { ... }` 创建一个箭头函数：`event` 是参数，花括号内是稍后被调用时执行的函数体。赋值时只登记函数，不会立刻执行其中的 `console.log`。

消息到达后，浏览器在接收方的执行环境中调用已登记的函数，并传入消息事件对象。`event.data` 是发送方传给 `postMessage` 的数据。这里的箭头函数可类比 Swift 的 `{ event in ... }` 闭包；相同的是“把函数作为值保存，稍后再调用”，不表示两种语言的调度规则相同。

### 同名属性不会跨对象覆盖

| 表达式 | 属性所属对象 | 接收哪一侧的消息 | 本例回调执行位置 |
| --- | --- | --- | --- |
| `worker.onmessage` | 页面主线程持有的 `Worker` 对象 | Worker 发回的消息 | 页面主线程 |
| `self.onmessage` | Worker 内的全局对象 | 页面主线程发来的消息 | Worker 线程 |

两个属性的名称相同，但所属对象不同，因此互不覆盖。只有再次给同一个对象的同一个 `onmessage` 属性赋值，才会替换该属性之前保存的处理函数。需要为同一对象登记多个监听函数时，可以使用 `addEventListener("message", handler)`。

本例中回调在主线程执行，是因为 `worker` 对象位于页面主线程。不能仅凭 `onmessage` 这个名称断定线程；Worker 内也可以创建其他 Worker，此时创建方不一定是页面主线程。

### 数据与界面的边界

普通对象通过 `postMessage` 发送时，通常使用结构化克隆，即由平台按支持的规则复制数据。接收方不会因此拿到发送方同一个普通对象的共享引用。可转移对象与显式共享内存有其他规则，不能把所有消息都理解为共享对象。

Worker 不能直接访问页面的 DOM。DOM 是浏览器把页面节点表示成对象、供脚本读取和修改的接口。因此，Worker 适合执行计算并发送结果，页面主线程收到结果后再修改界面。

## 与 Swift 的对应关系及边界

| 概念 | TypeScript / JavaScript | Swift 对照与边界 |
| --- | --- | --- |
| 类型检查与推断 | TypeScript 在执行前检查类型，也能推断类型 | Swift 也支持静态类型检查和类型推断 |
| 成员要求 | `interface` 描述对象所需成员，通常按结构匹配 | `protocol` 也描述要求，但遵循关系通常需要显式声明 |
| 泛型 | `<T>` 表达静态类型关系，类型参数会被擦除 | Swift 也有泛型，不能把 TypeScript 的擦除方式套用于 Swift 的实现 |
| 回调函数 | 箭头函数可以保存到属性，等待事件发生后执行 | Swift 闭包也可以保存并稍后调用 |
| 后台计算与结果更新 | Worker 计算，通过消息把结果交给页面主线程 | 可类比把计算安排到后台，再把界面更新安排到主线程；Worker 并不等于一个 GCD 队列 |
| `async` / `await` | 暂停与恢复异步函数，不自动创建后台线程 | Swift 也有同名语法，但执行位置遵循 Swift 的并发调度与隔离规则 |

跨平台框架还可能自行规定 JavaScript 的执行线程，不能把浏览器 Worker 模型直接套用到所有框架。相关内容见 [WebView 与 Lynx 双向通信](../cross-platform/webview-and-lynx-bridge.md) 和 [Lynx 渲染、线程模型与资源更新](../cross-platform/lynx-rendering-and-updates.md)。
