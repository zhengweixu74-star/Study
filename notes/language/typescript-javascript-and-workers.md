# TypeScript、JavaScript 与 Worker 线程通信

[返回笔记目录](../README.md)

## 本文目录

- [TypeScript 与 JavaScript 的关系](#typescript-与-javascript-的关系)
- [Node.js 与浏览器运行环境](#nodejs-与浏览器运行环境)
- [类型标注、类型推断、接口与泛型](#类型标注类型推断接口与泛型)
  - [类型标注与类型推断](#类型标注与类型推断)
  - [JavaScript number 与 Swift Int、Double](#javascript-number-与-swift-intdouble)
  - [接口描述对象的结构](#接口描述对象的结构)
  - [泛型保留输入与输出的类型关系](#泛型保留输入与输出的类型关系)
- [类型擦除与运行时检查](#类型擦除与运行时检查)
- [线程、事件循环与异步](#线程事件循环与异步)
- [Worker 消息通信与回调执行位置](#worker-消息通信与回调执行位置)
  - [两个脚本与一次消息往返](#两个脚本与一次消息往返)
  - [onmessage 赋值与箭头函数语法](#onmessage-赋值与箭头函数语法)
  - [同名属性不会跨对象覆盖](#同名属性不会跨对象覆盖)
  - [数据与界面的边界](#数据与界面的边界)
- [与 Swift 的对应关系及边界](#与-swift-的对应关系及边界)

## TypeScript 与 JavaScript 的关系

TypeScript 在 JavaScript 的基础上增加静态类型检查和相关语法。静态类型检查是工具在执行代码之前，根据类型声明和代码结构检查值的使用是否符合要求。JavaScript 的值也有类型，但普通 JavaScript 变量没有固定的静态类型声明，可以先保存数字，再保存字符串。

JavaScript 运行环境是执行 JavaScript 代码并提供定时器、网络等能力的软件，例如浏览器或 Node.js。TypeScript 代码在移除类型语法后，作为普通 JavaScript 由运行环境执行，所以 TypeScript 没有自己的线程模型。

JavaScript 代码一般可以直接写进 TypeScript 文件，但 TypeScript 可能对其中的代码报告类型错误。

## Node.js 与浏览器运行环境

Node.js 是让 JavaScript 在浏览器之外运行的运行环境，常用于编写服务器程序、命令行工具和自动化脚本。JavaScript 是编程语言；Node.js 是执行 JavaScript 代码并提供系统功能的软件。

V8 是负责解析和执行 JavaScript 代码的软件组件，属于 JavaScript 引擎。Node.js 使用 V8 执行 JavaScript 代码。Node.js 还提供文件读写、网络通信和进程管理等接口，让 JavaScript 程序可以使用这些系统能力。

浏览器也能运行 JavaScript，但浏览器和 Node.js 提供的功能不同：

| 运行环境 | 典型能力 | 常见用途 |
| --- | --- | --- |
| 浏览器 | 操作网页内容、响应按钮点击 | 实现网页交互 |
| Node.js | 读写本地文件、接收和响应网络请求 | 编写服务器程序和本地工具 |

例如，将下面的代码保存为 `hello.js`：

```javascript
const name = "小明";
console.log(`你好，${name}`);
```

安装 Node.js 后，在终端中进入 `hello.js` 所在目录，再执行：

```bash
node hello.js
```

Node.js 读取并执行 `hello.js`。代码先将字符串 `"小明"` 保存到 `name`，再通过 `console.log` 在终端输出：

```text
你好，小明
```

整个过程不需要打开浏览器。

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

这些报错发生在执行之前的类型检查阶段；运行时并没有代码拦截这次赋值。

### JavaScript number 与 Swift Int、Double

JavaScript 的 `number` 可以表示整数值和小数值，Swift 的 `Int` 只能表示整数。按数值表示方式比较，`number` 更接近 Swift 的 `Double`。

“值是整数”和“类型是整数类型”是两回事。`20` 是整数值，但程序既可以用整数类型保存 `20`，也可以用浮点类型保存。浮点类型用有限位数表示数值，可以表示小数，但不能精确表示所有小数。

| 类型 | 数值表示方式 | 示例 |
| --- | --- | --- |
| JavaScript `number` | 64 位二进制浮点数 | `20`、`20.5` 都属于 `number` |
| Swift `Int` | 有固定表示范围的整数，可以表示正数、负数和零；位数由平台决定 | 可以保存 `20`，不能保存 `20.5` |
| Swift `Double` | 64 位二进制浮点数 | 可以保存 `20` 和 `20.5` |

JavaScript 中：

```javascript
let age = 20;        // 当前值的类型是 number
age = 20.5;         // 仍然是 number

console.log(5 / 2);  // 2.5
```

`age` 从 `20` 变为 `20.5`，类型始终是 `number`。TypeScript 的 `number` 就是这个类型，所以声明 `age: number` 不会限制变量只能保存整数。

Swift 中：

```swift
var age = 20         // 编译器将 age 推断为 Int
// age = 20.5       // 编译错误：不能把小数赋给 Int

var value: Double = 20
value = 20.5        // 合法

print(5 / 2)        // 2：两个 Int 做整数除法
print(5.0 / 2.0)    // 2.5：两个 Double 做浮点除法
```

整数除法舍去商的小数部分（向零取整），所以 `5 / 2` 在 JavaScript 中得到 2.5，在 Swift 中得到 2。`number` 与 `Double` 的存储格式相同，但两种语言的类型检查、类型推断和转换规则不同。

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

`const` 只禁止给变量 `user` 重新赋值，对象的属性仍然可以修改，例如 `user.age = 21` 是合法的。

TypeScript 按结构判断对象是否符合接口：只要对象有接口要求的成员、类型也匹配，就符合接口，不需要声明。接口只用于类型检查，不能用来创建对象，运行时也不存在 `User` 这个值。接口在“描述成员要求”上类似 Swift 的 `protocol`，区别是 Swift 类型必须显式声明遵循协议。

### 泛型保留输入与输出的类型关系

泛型是用类型参数描述可用于多种类型的代码。下面的 `T` 是类型参数；参数 `value` 与返回值都使用 `T`，表示函数返回与输入相同类型的值。

```typescript
function identity<T>(value: T): T {
    return value;
}

const numberValue = identity<number>(10);
const stringValue = identity("hello"); // 也可以推断类型参数
```

这与 Swift 的 `func identity<T>(_ value: T) -> T` 类似。TypeScript 的类型参数只在类型检查时使用，运行时不存在。

## 类型擦除与运行时检查

类型擦除是执行前移除只用于静态检查的类型语法。类型标注、`interface` 和泛型类型参数不会直接保留在输出的 JavaScript 中。

```typescript
let age: number = 20;
```

移除类型标注后，对应代码是：

```javascript
let age = 20;
```

通常先用工具把 TypeScript 转成 JavaScript 再执行。部分运行环境可以直接加载 TypeScript 文件，加载时只移除类型语法，不做类型检查，也不支持所有 TypeScript 语法。

类型声明不会在运行时检查网络响应等外部数据。`as User` 是类型断言，告诉类型检查器“把这个值当作 `User`”；断言不会转换数据，也不会检查字段是否存在。外部数据必须符合约定时，程序要自己检查实际字段。

并非所有 TypeScript 语法都会被擦除。普通 `enum` 会生成一个运行时对象，属于需要生成 JavaScript 代码的语法。

## 线程、事件循环与异步

线程是操作系统调度执行的一条指令序列。异步调用让调用方不必等结果就继续执行，结果准备好后再接收；异步不代表创建了新线程。

在浏览器页面的主线程上，一段同步 JavaScript 代码执行完之前，其他回调都不能执行。页面的 JavaScript 只在主线程执行，但浏览器自己还会用其他线程处理网络、渲染等工作。

事件循环是运行环境不断从队列中取出待执行的回调并执行的机制。回调是先交给其他代码保存、等到指定事件发生时才被调用的函数。主线程长时间执行计算时，其他回调和界面交互都要等待。

| 机制 | 作用 | 是否自动把函数体移到新线程 |
| --- | --- | --- |
| `setTimeout` | 达到定时条件后安排回调，实际执行还需等待调度 | 否 |
| `Promise` | 表示操作的完成结果或失败原因，并登记后续处理函数 | 否 |
| `async` / `await` | 用异步函数和等待表达式组织依赖异步结果的代码 | 否 |
| 浏览器 `Worker` | 创建独立的工作执行环境，在工作线程运行脚本 | 是，指定脚本由 Worker 执行 |
| Node.js `worker_threads` | 为 Node.js 提供 Worker 线程能力 | 是，具体接口由 Node.js 提供 |

`new Promise(...)` 传入的执行函数会立即同步执行；`async` 函数在第一次遇到 `await` 之前也是同步执行的。把耗时循环放在这两个位置，主线程照样被阻塞。

`await` 暂停当前异步函数，结果就绪后再继续执行后面的代码。暂停期间线程可以执行其他回调，代码也不会被切到后台线程。`setTimeout(..., 0)` 也不是立即执行：要等当前同步代码结束，回调再排队执行。

## Worker 消息通信与回调执行位置

### 两个脚本与一次消息往返

下面以浏览器页面创建专用 Worker 为例。专用 Worker 是只供创建方使用的独立脚本执行环境，运行在单独的线程上。主线程持有一个 `Worker` 对象，通过它发送消息和接收结果；Worker 脚本中的 `self` 指向 Worker 的全局对象，即 Worker 执行环境中提供全局属性和方法的对象。

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

`postMessage` 只是把消息放进对方的队列，不会直接调用对方的函数。`postMessage` 返回时，对方的接收回调可能还没执行。

### onmessage 赋值与箭头函数语法

```javascript
worker.onmessage = (event) => {
    console.log("计算结果:", event.data);
};
```

`worker.onmessage` 是 `worker` 对象上的消息处理属性。`=` 把右侧函数保存到该属性。`(event) => { ... }` 创建一个箭头函数：`event` 是参数，花括号内是稍后被调用时执行的函数体。赋值时只登记函数，不会立刻执行其中的 `console.log`。

消息到达后，浏览器在接收方的执行环境中调用已登记的函数，并传入消息事件对象。`event.data` 是发送方传给 `postMessage` 的数据。

### 同名属性不会跨对象覆盖

| 表达式 | 属性所属对象 | 接收哪一侧的消息 | 本例回调执行位置 |
| --- | --- | --- | --- |
| `worker.onmessage` | 页面主线程持有的 `Worker` 对象 | Worker 发回的消息 | 页面主线程 |
| `self.onmessage` | Worker 内的全局对象 | 页面主线程发来的消息 | Worker 线程 |

两个属性的名称相同，但所属对象不同，因此互不覆盖。只有再次给同一个对象的同一个 `onmessage` 属性赋值，才会替换该属性之前保存的处理函数。需要为同一对象登记多个监听函数时，可以使用 `addEventListener("message", handler)`。

本例中 `worker.onmessage` 的回调在主线程执行，是因为 `worker` 对象是在页面主线程创建的。回调在哪个线程执行，取决于 Worker 对象由谁创建：Worker 内部也可以再创建 Worker，这时回调在外层 Worker 的线程执行。

### 数据与界面的边界

通过 `postMessage` 发送普通对象时，浏览器使用结构化克隆复制一份数据，接收方拿到的是副本，不是同一个对象。可转移对象（如 ArrayBuffer）和 SharedArrayBuffer 另有规则：前者把所有权转给接收方，后者让双方共享同一块内存。

Worker 不能直接访问页面的 DOM。DOM 是浏览器把页面节点表示成对象、供脚本读取和修改的接口。因此，Worker 适合执行计算并发送结果，页面主线程收到结果后再修改界面。

## 与 Swift 的对应关系及边界

| 概念 | TypeScript / JavaScript | Swift 对照与边界 |
| --- | --- | --- |
| 类型检查与推断 | TypeScript 在执行前检查类型，也能推断类型 | Swift 也支持静态类型检查和类型推断 |
| 成员要求 | `interface` 描述对象所需成员，通常按结构匹配 | `protocol` 也描述要求，但遵循关系通常需要显式声明 |
| 泛型 | `<T>` 表达静态类型关系，类型参数会被擦除 | Swift 也有泛型，不能把 TypeScript 的擦除方式套用于 Swift 的实现 |
| 回调函数 | 箭头函数可以保存到属性，等待事件发生后执行 | Swift 闭包也可以保存并稍后调用 |
| 后台计算与结果更新 | Worker 计算，通过消息把结果交给页面主线程 | 类似把计算放到后台、再回到主线程更新界面；但 Worker 是独立的执行环境，不共享内存，与 GCD 队列不同 |
| `async` / `await` | 暂停与恢复异步函数，不自动创建后台线程 | Swift 也有同名语法，但代码在哪里执行由 Swift 的隔离规则决定 |

跨平台框架可能自己规定 JavaScript 在哪个线程执行，与浏览器的 Worker 模型不同。相关内容见 [WebView 与 Lynx 双向通信](../cross-platform/webview-and-lynx-bridge.md) 和 [Lynx 渲染、线程模型与资源更新](../cross-platform/lynx-rendering-and-updates.md)。
