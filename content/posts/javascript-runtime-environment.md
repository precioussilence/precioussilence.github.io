---
title: Javascript运行时环境学习笔记
date: 2025-09-08T16:32:18+08:00
cover: /images/bottle.jpg
images:
  - /images/bottle.jpg
categories:
  - '学习笔记'
tags:
  - JavaScript
  - 'Event Loop'
  - 'Runtime'
# nolastmod: true
# math: true
draft: false
---

JavaScript Runtime Environment学习笔记

<!--more-->

> ⚠️ **【重要免责声明】** 
> 1. 该笔记是我在学习JavaScript Runtime Environment过程中，按照自己的理解记录的知识点，主要目的是供自己复习回顾
> 1. 该笔记使用豆包和DeepSeek做了大量信息收集工作
> 1. 受限于个人眼界，这里所列的知识点未必实时和精确，若您有缘看到该笔记，我只能说此内容颇具抛砖引玉之功，并无指点迷津之效。

## JavaScript Runtime Environment

该笔记旨在深入剖析JavaScript的执行模型，解释其单线程、异步和非阻塞行为背后的底层机制。该笔记将从宏观（引擎 vs 宿主）到微观（执行上下文、调用栈和事件循环），层层递进，力争构建出一个非常完整的知识体系。

### The engine and the host

JavaScript代码的执行依赖JavaScript引擎（JavaScript engine）和宿主环境（host environment）的共同作用。简单来说：

- 引擎是“执行核心”：负责代码的解析、编译和执行，是JavaScript语言本身的运行核心。
- 宿主环境是“平台支撑”：提供运行容器、扩展API，并管理事件循环，让JavaScript能在具体场景（如浏览器、客户端、服务器等）中发挥作用。

> JavaScript execution requires the cooperation of two pieces of software: the JavaScript engine and the host environment.
>
> The JavaScript engine implements the ECMAScript (JavaScript) language, providing the core functionality. It takes source code, parses it, and executes it. However, in order to interact with the outside world, such as to produce any meaningful output, to interface with external resources, or to implement security- or performance-related mechanisms, we need additional environment-specific mechanisms provided by the host environment. For example, the HTML DOM is the host environment when JavaScript is executed in a web browser. Node.js is another host environment that allows JavaScript to be run on the server side.
>
> 以上文本引自 [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

**JavaScript引擎**

引擎是执行JavaScript代码的核心程序，其主要职责是将代码转换为可执行的机器指令，专注于解析、编译与执行。它的核心作用包括：

- 实现ECMAScript规范：提供规范定义的所有内置对象（如Array, Promise）、方法（如map, filter）及语言特性（如闭包、异步逻辑）。
- 解析与检查：将原始JavaScript代码转换为抽象语法树（AST），并检查语法错误。
- 编译与优化：通过解释执行和即时编译（JIT）相结合的方式，将代码转换为高效的低级指令（字节码或机器码）并进行动态优化。
- 代码执行：管理调用栈、执行上下文、变量作用域等，执行代码逻辑。
- 内存管理：负责内存分配（例如为对象和变量分配空间）与垃圾回收（自动回收不再使用的内存）。

常见引擎：V8（Chrome/Node.js）、SpiderMonkey（Firefox）、JavaScriptCore（Safari）等。

**JavaScript宿主环境**

宿主环境是JavaScript代码运行的外部容器，它为引擎提供运行上下文，并扩展JavaScript与外部系统交互的能力。其核心作用包括：

- 提供全局执行环境：定义全局对象（如浏览器中的window、Node.js中的global），作为代码执行的顶层上下文。
- 提供特定领域API：提供引擎不包含的、特定领域的API，使JavaScript能与外部系统交互：
  - 浏览器宿主环境：提供DOM（document）、BOM（localStorage、fetch）、事件系统（addEventListener）等，支持网页交互。
  - Node.js宿主环境：提供文件系统（fs）、网络（http）、进程管理（process）等模块，支持服务器端开发。
- 管理事件循环：负责实现事件循环机制，并驱动其运行，协调同步任务、异步任务（如定时器、网络请求）的执行顺序（不同宿主环境的事件循环细节可能不同）。
- 桥接底层系统：作为JavaScript与操作系统、硬件的中间层（如浏览器桥接渲染引擎，Node.js桥接操作系统接口）。

常见宿主环境：浏览器（Chrome、Firefox）、Node.js、Electron（桌面应用）、React Native（移动应用）等。

### Agent execution model

众所周知JavaScript是“单线程”的。这一特性的核心正是体现在Agent的“单线程”执行模式上。每个Agent类似于一个线程，之所以说类似，是因为其底层实现可能是、也可能不是一个真正的操作系统线程，这取决于具体的宿主环境。若是为Agent的“单线程”换个更为形象的说法，那就是：在一个Agent的生命周期内，代码是顺序执行的，不存在并行执行多段代码的情况。

然而，JavaScript宿主环境并不局限于单个Agent，相反，它采用的是Multi-Agent架构，这正是现代JavaScript能够处理并发工作的关键。因此我们就看到浏览器或Node.js可以同时做很多事情，这与JavaScript的“单线程”执行理念并不冲突。

同一个Agent内部又可以划分为不同的“子环境”，即Realm。每个Realm对应一个全局对象，负责隔离不同的全局上下文。同一个Agent内的Realm可彼此同步访问，不同Agent内的Realm只能通过消息传递通信。

**Agent**

> In the JavaScript specification, each autonomous executor of JavaScript is called an agent, which maintains its facilities for code execution:
>
> - Heap (of objects): this is just a name to denote a large (mostly unstructured) region of memory. It gets populated as objects get created in the program. Note that in the case of shared memory, each agent has its own heap with its own version of a SharedArrayBuffer object, but the underlying memory represented by the buffer is shared.
> - Queue (of jobs): this is known in HTML (and also commonly) as the event loop which enables asynchronous programming in JavaScript while being single-threaded. It's called a queue because it's generally first-in-first-out: earlier jobs are executed before later ones.
> - Stack (of execution contexts): this is what's known as a call stack and allows transferring control flow by entering and exiting execution contexts like functions. It's called a stack because it's last-in-first-out. Every job enters by pushing a new frame onto the (empty) stack, and exits by emptying the stack.
>
> 以上文本引自 [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

Agent是一个抽象概念，是ECMAScript规范中最小的独立执行单元，它代表一个拥有专属线程、任务队列、事件循环和执行上下文的“封闭系统”。Agent的核心功能包括：

- 执行控制：拥有独立的执行线程（单线程），所有代码在该线程内按“调用栈 + 事件循环 + 任务队列”的机制执行，不存在并发冲突。
- 资源隔离：拥有独立的内存空间、变量环境和执行状态，与其他Agent完全隔离（不同Agent间不能直接访问彼此资源，必须通过诸如postMessage的消息传递方式通信，数据通过结构化克隆传递）。
- 任务管理：维护自己的任务队列（宏任务、微任务）和事件循环，负责异步任务的执行调度。

Agent的典型实例（宿主环境实现）有：

- 浏览器主线程（处理DOM、渲染、主线程JS执行）
- Web Worker线程（后台计算，不阻塞主线程）
- Service Worker线程（拦截网络请求、离线缓存）
- Node.js中的Worker线程（服务器端并发计算）

**Realm**

> Each agent owns one or more realms. Each piece of JavaScript code is associated with a realm when it's loaded, which remains the same even when called from another realm. A realm consists of the follow information:
>
> - A list of intrinsic objects like Array, Array.prototype, etc.
> - Globally declared variables, the value of globalThis, and the global object
> - A cache of template literal arrays, because evaluation of the same tagged template literal expression always causes the tag to receive the same array object
>
> On the web, the realm and the global object are 1-to-1 corresponded. The global object is either a Window, a WorkerGlobalScope, or a WorkletGlobalScope. So for example, every iframe executes in a different realm, though it may be in the same agent as the parent window.
>
>Realms are usually mentioned when talking about the identities of global objects. For example, we need methods such as Array.isArray() or Error.isError(), because an array constructed in another realm will have a different prototype object than the Array.prototype object in the current realm, so instanceof Array will wrongly return false.
>
> 以上文本引自 [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

Realm是ECMAScript规范中另一个抽象概念，它是Agent内部的全局环境隔离容器，Realm代表一组相互关联的全局对象、内置对象和执行规则，用于隔离不同的“全局上下文”。Realm的核心功能包括：

- 全局对象管理：每个Realm对应唯一的全局对象（如浏览器中的window、iframe中的contentWindow），所有全局变量、函数都挂载在该对象上。
- 内置对象隔离：拥有独立的内置对象副本（如Object、Array、Promise等），不同Realm的内置对象即使功能相同，也属于不同实例。
- 同步交互能力：同一Agent内的多个Realm可直接同步访问彼此的资源（如主页面通过iframe.contentWindow调用iframe的函数），无需通过消息传递的方式实现通信。
- 提供安全边界：可通过宿主环境的安全策略（如浏览器同源策略）限制跨Realm访问，避免恶意代码篡改。

Realm的典型实例（宿主环境实现）有：

- 浏览器主页面的全局环境（window对应的Realm）
- 页面中`<iframe>`标签的独立全局环境（iframe.contentWindow对应的Realm）
- Node.js中通过vm模块创建的沙箱环境（独立全局对象的Realm）

### Stack and execution contexts

在JavaScript中，调用栈（Call Stack）和执行上下文（Execution Contexts）是Agent执行代码的核心机制，它们共同掌管着代码的执行顺序、变量的访问规则和函数的调用关系等JavaScript代码核心规则。

> We first consider synchronous code execution. Each job enters by calling its associated callback. Code inside this callback may create variables, call functions, or exit. Each function needs to keep track of its own variable environments and where to return to. To handle this, the agent needs a stack to keep track of the execution contexts. A execution context, also known generally as a stack frame, is the smallest unit of execution. It tracks the following information:
>
> - Code evaluation state
> - The module or script, the function (if applicable), and the currently executing generator that contains this code
> - The current realm
> - Bindings, including:
>   - Variables defined with var, let, const, function, class, etc.
>   - Private identifiers like #foo which are only valid in the current context
>   - this reference
>
> 以上文本引自 [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

**执行上下文**

执行上下文是JavaScript引擎为每段代码创建的执行环境，用于存储代码执行所需的全部信息，它是代码得以执行的“基石”。每个执行上下文包含三个核心部分：

- 变量环境（Variable Environment）：存储var声明的变量、函数声明（function）和函数参数，包含变量提升（hoisting）的结果。
- 词法环境（Lexical Environment）：存储let、const声明的变量，以及块级作用域相关信息，特性是“暂时性死区”（Temporal Dead Zone）。
- this 绑定（This Binding）：确定当前执行上下文中this关键字的指向（如全局对象、函数调用者、实例对象等）。

JavaScript中存在三种执行上下文：

- 全局执行上下文
  - 脚本首次加载时创建（比如页面初始化）
  - 它是所有代码的执行根环境
  - 它负责创建全局对象（比如浏览器window对象）
  - 绑定this到全局对象
  - 管理全局变量和函数
- 函数执行上下文
  - 函数被调用时创建
  - 它负责存储函数的参数、局部变量和内部函数
  - 它负责建立与外层上下文的链接（作用域链）
  - 它负责确定函数内this的指向
- eval执行上下文
  - 使用eval函数执行字符串代码时创建
  - 它负责为动态执行的代码创建独立环境（不推荐使用，存在性能和安全问题）

执行上下文的生命周期分为三个阶段：

- 创建阶段：引擎初始化上下文，完成变量环境 / 词法环境的创建、this绑定等（此时变量已“声明”但未“赋值”，值为undefined）。
- 执行阶段：引擎执行代码，为变量赋值、执行函数调用、处理流程控制（如if、for）等。
- 销毁阶段：上下文执行完毕后，从调用栈中弹出，其占用的内存被垃圾回收（闭包会阻止此过程）。

**调用栈**

调用栈是JavaScript引擎用于管理执行上下文顺序的栈结构（遵循“后进先出”LIFO原则），它决定了代码的执行优先级和顺序。其核心功能包括：

- 跟踪执行位置：记录当前正在执行的上下文，以及后续需要执行的上下文。
- 控制执行顺序：确保代码按“函数调用顺序”执行（先调用的后完成）。
- 切换上下文：函数调用时推入新上下文，函数执行完毕后弹出上下文，恢复之前的执行状态。

下面是一段简单的代码，可借其一窥调用栈的本来面目：

```javascript
function b() {
  console.log('b 执行'); // 步骤 3
}

function a() {
  console.log('a 执行'); // 步骤 2
  b(); // 调用 b，推入 b 的上下文
}

console.log('全局代码'); // 步骤 1
a(); // 调用 a，推入 a 的上下文
```

以上述代码为例，调用栈的变化过程如下：

1. 脚本加载，全局执行上下文入栈（栈：[全局上下文]）。
1. 执行 console.log('全局代码')，完成后继续执行。
1. 调用 a()，a 的函数执行上下文入栈（栈：[全局上下文, a 上下文]）。
1. 执行 console.log('a 执行')，完成后调用 b()。
1. 调用 b()，b 的函数执行上下文入栈（栈：[全局上下文, a 上下文, b 上下文]）。
1. 执行 console.log('b 执行')，b 执行完毕，b 的上下文出栈（栈：[全局上下文, a 上下文]）。
1. a 执行完毕，a 的上下文出栈（栈：[全局上下文]）。
1. 全局代码执行完毕，全局上下文出栈，栈空。

调用栈的关键特性：

- 单线程限制：由于JavaScript单线程特性，调用栈同一时间只能处理一个上下文（规避了并发冲突问题）。
- 栈溢出（Stack Overflow）：若函数递归调用次数过多（如无限递归），会导致调用栈中上下文堆积，以至于超出引擎栈容量限制，抛出RangeError: Maximum call stack size exceeded。
- 与异步任务的配合：调用栈仅处理同步任务，异步任务（如setTimeout、fetch）完成后，其回调函数会被放入任务队列，等待调用栈清空后由事件循环推入栈中执行。

至此，我们可以清晰地看到执行上下文与调用栈的关系：

- 两者是“内容”与“容器”的关系，协同保障代码有序执行
- 调用栈是执行上下文的“管理器”：执行上下文的创建、激活、销毁全由调用栈控制，栈帧的顺序决定了上下文的执行优先级。
- 执行上下文是调用栈的“管理对象”：调用栈中存储的是执行上下文实体，栈的操作（推入 / 弹出）本质是对上下文生命周期的管理。
- 共同支撑作用域链：每个执行上下文在创建时会生成作用域链（当前上下文变量环境 + 外层上下文作用域链），而调用栈的层级结构（如函数嵌套）直接决定了作用域链的层级，确保变量能按规则查找（从内到外）。

### Job queue and event loop

在 JavaScript 中，任务队列（Job Queue，也称为 Task Queue） 和 事件循环（Event Loop） 是处理异步操作的核心机制，它们与调用栈和执行上下文配合，共同实现了单线程环境下的非阻塞并发功能。

> An agent is a thread, which means the interpreter can only process one statement at a time. When the code is all synchronous, this is fine because we can always make progress. But if the code needs to perform asynchronous action, then we cannot progress unless that action is completed. However, it would be detrimental to user experience if that halts the whole program—the nature of JavaScript as a web scripting language requires it to be never blocking. Therefore, the code that handles the completion of that asynchronous action is defined as a callback. This callback defines a job, which gets placed into a job queue—or, in HTML terminology, an event loop—once the action is completed.
>
> Every time, the agent pulls a job from the queue and executes it. When the job is executed, it may create more jobs, which are added to the end of the queue. Jobs can also be added via the completion of asynchronous platform mechanisms, such as timers, I/O, and events. A job is considered completed when the stack is empty; then, the next job is pulled from the queue. Jobs might not be pulled with uniform priority—for example, HTML event loops split jobs into two categories: tasks and microtasks. Microtasks have higher priority and the microtask queue is drained first before the task queue is pulled. For more information, check the HTML microtask guide. If the job queue is empty, the agent waits for more jobs to be added.
>
> 以上文本引自 [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

**Job queue**

任务队列是JavaScript引擎用于存储异步操作完成后待执行的回调函数的队列结构。当异步操作（如定时器、网络请求、DOM事件）完成后，其对应的回调函数会被放入任务队列，等待调用栈空闲时执行。

根据优先级和来源，任务队列中的任务分为两类，执行顺序有严格区分：

- 微任务（Microtasks）
  - Promise.then/catch/finally
  - async/await（本质是Promise语法糖）
  - queueMicrotask()
  - MutationObserver（DOM变化观察器）
  - 同一轮事件循环中，所有微任务会在宏任务执行后立即执行，且会清空微任务队列后再执行下一个宏任务。
- 宏任务（Macrotasks）
  - setTimeout/setInterval
  - setImmediate（Node.js环境）
  - I/O操作（如文件读取、网络请求）
  - DOM事件（如click、load）
  - script标签中的顶层代码
  - 每次事件循环仅执行一个宏任务，执行完毕后会优先处理全部微任务，然后再执行下一个宏任务。

任务队列的核心功能包括：

- 暂存异步回调：当异步操作（如setTimeout倒计时结束、fetch请求返回）完成后，其回调函数不会立即执行，而是被放入对应的任务队列等待。
- 按类型排序：微任务和宏任务分别存储在不同的队列中，保证优先级高的微任务先于宏任务执行。
- 与事件循环配合：为事件循环提供“待执行任务”的来源，确保异步操作不会阻塞主线程。

**Event Loop**

事件循环是JavaScript引擎的核心协调机制，它不断监控调用栈和任务队列，决定何时执行哪个任务，确保单线程环境下异步操作能有序执行。

事件循环遵循严格的执行顺序，可简化为以下步骤（以浏览器环境为例）：

1. 执行同步代码：先清空调用栈中的所有同步任务（从全局执行上下文开始，按顺序执行函数调用）。
1. 处理微任务队列：调用栈清空后，检查微任务队列，依次执行所有微任务（执行过程中新增的微任务会被加入队列末尾并一同执行），直到微任务队列为空。
1. 执行UI渲染（浏览器特有）：微任务执行完毕后，浏览器可能会进行一次 UI 渲染（如更新DOM、重绘页面）。
1. 处理宏任务队列：从宏任务队列中取出第一个任务，推入调用栈执行其同步代码（该宏任务的回调函数）。
1. 重复循环：回到步骤 2，继续处理新产生的微任务，如此往复，形成“事件循环”。

```javascript
console.log('同步任务 1'); // 同步代码

setTimeout(() => {
  console.log('宏任务 1'); // 宏任务
}, 0);

Promise.resolve().then(() => {
  console.log('微任务 1'); // 微任务
}).then(() => {
  console.log('微任务 2'); // 微任务（由前一个微任务触发）
});

console.log('同步任务 2'); // 同步代码
```

事件循环的执行顺序，以上述这段简单代码为例：

1. 执行同步代码：打印“同步任务 1”
1. 执行同步代码：执行setTimeout函数，由于时间设置为0，因此web api直接把回调函数放入宏任务队列。
1. 执行同步代码：执行第一个then函数，微任务 1被放入微任务队列
1. 执行同步代码：打印“同步任务 2”（此时调用栈清空）。
1. 处理微任务队列：执行 微任务 1 → 打印“微任务 1” -> 将微任务 2加入微任务队列。
1. 处理微任务队列：执行 微任务 2 → 打印“微任务 2”（此时微任务队列为空）。
1. （微任务队列清空后，浏览器可能执行 UI 渲染）。
1. 处理宏任务队列：执行 宏任务 1 → 打印“宏任务 1”。
1. 重复循环（此时无新任务，循环等待）。

最终输出顺序：同步任务 1 → 同步任务 2 → 微任务 1 → 微任务 2 → 宏任务 1。

事件循环的核心功能包括：

- 协调同步与异步任务：确保同步任务优先执行，异步任务在完成后按规则排队等待，避免阻塞主线程。
- 保障异步任务优先级：通过“先微任务后宏任务”的规则，明确异步回调的执行优先级（如Promise回调比定时器回调先执行）。
- 实现非阻塞I/O：在单线程模型下，通过事件循环将耗时操作（如网络请求）交给宿主环境（如浏览器内核）处理，完成后再通过任务队列通知主线程，实现 “看似并发” 的效果。

至此，我们可以清晰地看到任务队列与事件循环的关系：

- 两者是“数据存储”与“调度机制”的关系，共同支撑异步操作
- 任务队列是事件循环的“数据源”：事件循环需要从任务队列中获取待执行的异步任务，没有任务队列，事件循环就没有调度对象。
- 事件循环是任务队列的“处理器”：任务队列仅负责存储任务，而事件循环则决定了何时取出任务、按什么顺序执行任务。
- 协同解决单线程瓶颈：单线程若直接执行所有任务（包括耗时的异步操作）会导致阻塞，而通过任务队列暂存异步任务，事件循环在合适时机调度执行异步任务，则使得主线程能同时“等待”多个异步操作，实现非阻塞效果。

事件循环和任务队列的核心逻辑由ECMAScript规范定义，但具体实现细节因宿主环境而异：

- 浏览器环境：微任务优先级高于宏任务，且在微任务执行后可能触发 UI 渲染。
- Node.js环境：宏任务队列有更细的分类（如timers、poll、check等），执行顺序与浏览器不同；微任务队列也分为nextTick（优先级最高）和普通微任务。
- 例如，Node.js中process.nextTick()的回调会优先于所有微任务和宏任务执行，这与浏览器环境不同。

任务队列解决了“异步操作完成后如何通知主线程”的问题，通过分类存储确保不同类型的异步任务按优先级等待。事件循环解决了“单线程如何协调同步与异步任务”的问题，通过固定的执行规则，使JavaScript在单线程限制下仍能高效处理并发操作（如同时等待多个网络请求、响应用户交互）。理解这两个机制，能帮助你预测异步代码的执行顺序（如为什么Promise比setTimeout先执行）、避免因异步逻辑错误导致的bug（如DOM操作在异步回调中执行的时机问题）。

### 总结

JavaScript的并发模型核心在于：用单线程的调用栈处理同步代码，用多线程的宿主环境处理异步 I/O，用事件循环机制来调度异步任务回到主线程执行的顺序。其中，微任务的高优先级保证了Promise等操作的及时性，而宏任务与渲染的穿插则保证了页面响应的及时性。