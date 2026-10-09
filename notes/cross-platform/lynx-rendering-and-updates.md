# Lynx 渲染、线程模型与资源更新

[返回笔记目录](../README.md)

## 本文目录

- [页面描述到平台 UI 的渲染流程](#页面描述到平台-ui-的渲染流程)
- [双线程、首帧直出与后台接管](#双线程首帧直出与后台接管)
- [动态资源与客户端发版边界](#动态资源与客户端发版边界)

公开 ReactLynx 的机制与项目内部版本分别说明。线上数值保留原始统计时间和采样条件；新增计时方法仍为待实施方案。

## 页面描述到平台 UI 的渲染流程

开发时用 TSX、JSX 和样式描述页面，构建工具把它们转换、打包成 Lynx bundle。客户端加载 bundle 后，引擎解析脚本、样式和配置，执行渲染逻辑生成元素树，再计算样式和布局，创建或更新平台 UI，最后由系统绘制。

参考：[官方渲染流程](https://lynxjs.org/guide/performance/analysis-performance/reactlynx-render-process)。本地内部工程使用 CompilerNG／LepusNG；公开框架的具体 API 和内部版本不能直接混用。

不是把同一段 TypeScript 直接交给 Swift 或 Kotlin 编译器，也不能把 Lynx 统一理解为 WebView。需要相机、相册、账号等宿主能力时，通常通过 Native 组件或 JSBridge 接入；宿主需要提供相应实现，平台能力和版本仍可能不同。参考：[Lynx 项目](https://github.com/lynx-family/lynx)、[宿主 App 集成](https://lynxjs.org/3.8/guide/start/integrate-with-existing-apps.html?platform=android)。

跨平台依赖两端集成的 Lynx 引擎、统一页面描述与平台适配层，分别创建和更新本平台 UI。

## 双线程、首帧直出与后台接管

Lynx 的双线程架构主要分为主线程和后台线程，两边都有独立的 JS 执行环境。主线程负责初始界面构建、应用 UI 更新和主线程脚本；后台运行完整的组件运行时，处理业务逻辑、组件状态、普通事件回调和更新计算。

以 ReactLynx 为例，构建阶段会生成面向两个线程的代码。页面启动时，主线程执行精简的首帧逻辑，利用已经准备好的数据先显示界面；后台同时初始化组件运行环境，准备好后与主线程已有节点建立对应关系，接管后续更新。

状态变化后，后台计算差异，生成 Patch，也就是界面修改指令，异步发送给主线程；主线程根据指令更新对应 Element，由渲染链路完成必要的布局和绘制。

这样既能减少首屏等待，也能降低业务计算对主线程的占用。代价是跨线程传递和调度延迟；拖拽、跟手动画等场景可以用主线程脚本 MTS 就近处理交互，减少往返。

线程分工参考：[ReactLynx 编程思想](https://lynxjs.org/zh/react/thinking-in-reactlynx)。

**框架与执行引擎的分工。**

| 层次 | 负责什么 |
|-|-|
| ReactLynx | Lynx 官方的 React 风格 UI 框架，底层基于 Preact；管理组件、Hooks、状态和更新计算，将页面描述转为 Lynx 引擎调用。 |
| Lynx 引擎 | 提供 Element 操作、样式、布局及平台 UI 渲染能力。 |
| PrimJS／JavaScriptCore | 执行 JavaScript 的引擎，为框架和业务代码提供运行基础。 |

React 风格指“组件化、声明式 UI、状态驱动更新”：开发者描述当前状态应该显示什么，框架处理更新。思路类似 SwiftUI 的 `@State` 驱动 `body`。例如下面的计数器示例：

```jsx
import { useState } from '@lynx-js/react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <view bindtap={() => setCount(count + 1)}>
      <text>{count}</text>
    </view>
  );
}
```

这里使用 Lynx 的 `view`、`text` 元件。参考：[ReactLynx 官方介绍](https://lynxjs.org/react/introduction)。

按原始资料在 2026-09-30 记录的默认配置，主线程使用基于 QuickJS 的 PrimJS；后台在 Android 默认使用 PrimJS，在 iOS 默认使用 JavaScriptCore，调试时可切换到 PrimJS。两侧是独立 JS 环境，对象、闭包和状态不会自动共享，需要通过框架提供的机制传递；不能把 ReactLynx 当作 JS 引擎。参考：[JS 运行时](https://lynxjs.org/zh/guide/scripting-runtime/index.html)。

**编译期的双线程产物。**

是按线程职责生成不同产物，并非把源码机械地切成互不重叠的两半。首帧所需的组件求值可能在两侧执行；普通事件回调、`useEffect` 等后台专属逻辑按编译规则裁剪，MTS 则在主线程执行。具体依赖分析和标注规则见[ReactLynx 编程思想](https://lynxjs.org/zh/react/thinking-in-reactlynx)。

编译器还会提取稳定的宿主元件结构，为可变属性、文本和子树预留位置。比如计数器外层 `view` 和内部 `text` 的结构稳定，`count` 是动态值；后续更新可以定位动态位置，减少重复创建、比较静态结构。参考：[编译协同渲染](https://lynxjs.org/zh/react/compiler-informed-rendering)。

**Element 树与后台组件结构。**

- **Element 树**是 Lynx 引擎中的元件节点层级，保存界面结构及相关属性，由主线程侧操作；平台视图和布局由引擎进一步处理。
- **“逻辑树”**是本文中的概括，指后台组件／虚拟节点、模板实例及动态数据等用于描述和计算 UI 的结构，不是一个统一的官方类型名。后台组件运行时另外管理状态和回调；它不持有主线程的真实 Element 引用。

例如计数器在后台记录“这里应显示 count”，主线程侧则持有实际承载该内容的 Element。两者通过对应关系协作。模板与实例的双线程分工见[编译协同渲染](https://lynxjs.org/zh/react/compiler-informed-rendering)。

**后台接管与实例对应关系。**

以计数器为例：

1. 主线程沿首帧路径创建 Element，先显示 `0`。
2. 后台初始化组件状态 `count = 0`、回调和节点结构，为后续更新建立基础。
3. 框架匹配两侧结构并建立实例 ID 对应关系，后台由此知道后续修改应落到主线程的哪个实例。
4. 相同部分复用；若动态值不同就更新对应值，若动态子树不同就局部调整。例如首帧价格为 `199`、后台结果为 `179` 时，修正价格文本；这是差异处理示例，不表示必然存在差异。

后台这次计算是为了建立可持续更新的组件环境，不是在后台创建另一套 UIView。接管后主线程仍持有 Element，后台负责驱动后续状态更新。后续计数器流程：

```text
主线程收到点击
  → 后台执行普通点击回调，count 从 0 变成 1
  → 后台计算变化并发送 Patch
  → 主线程按实例对应关系找到文本位置，应用修改
  → 必要的布局与绘制后，显示 1
```

Patch 可以表达属性修改及节点增删等变化，无须每次传整棵树。参考：[ReactLynx 首帧与接管原理](https://lynxjs.org/next/zh/blog/reactlynx-instant-first-frame-rendering)。

**首帧直出与双线程的适用条件。**

- 首屏所需 bundle 和初始数据应可同步使用。例如 Native 已提供商品名和价格，主线程可以先显示；如果价格必须等网络请求，就只能先显示已知内容或占位，不能消除数据等待。参考：[首帧直出](https://lynxjs.org/zh/guide/interaction/ifr.html)。
- 后台繁忙仍会延迟普通事件处理与页面更新；MTS 用于减少高频交互的跨线程往返，其自身计算也占用主线程。MTS 捕获的后台数据是经同步传入的值，不是可随意修改的共享变量。参考：[主线程脚本](https://lynxjs.org/zh/react/main-thread-script)。
- “双线程”描述主要脚本职责，不表示整个 App 只有两个 OS 线程，也不表示所有 JS 都在后台；布局等任务的具体线程安排仍需结合引擎版本与配置说明。

## 动态资源与客户端发版边界

客户端安装包中已有 Lynx 引擎、容器和桥接能力；页面 bundle 可以由资源系统独立下发。客户端取得新的兼容 bundle 后，用已有引擎加载它，就能更新页面布局、文案和部分业务逻辑。项目中相关资源获取涉及 Gecko 等机制。

动态更新的边界是现有客户端能力：若需要新的 Native 接口、平台组件、权限声明、引擎能力或不兼容的协议，仍可能需要客户端发版。下载新资源也不等于所有用户立即生效，还取决于版本匹配、缓存、下发时机、灰度和加载入口；不能说任意原生功能都可动态更新。

Native 页面代码随客户端发布；选型需要权衡动态迭代能力、原生体验与长期维护成本。

应该先确认资源包是否已经本地可用，再分别看容器初始化、bundle 解析执行、业务数据请求、布局和图片加载。只有明确本次打开确实在等远程资源，才能把这部分耗时归到资源获取上。不能用“TypeScript 编译产物不适配系统”作为未经验证的原因。
