---
title: Javascript Asynchronous学习笔记
date: 2025-09-10T13:18:23+08:00
cover: /images/park.jpg
images:
  - /images/park.jpg
categories:
  - JavaScript
  - '学习笔记'
tags:
  - 'Node.js'
  - Asynchronous
# nolastmod: true
# math: true
draft: false
---

Javascript Asynchronous学习笔记

<!--more-->

该笔记是我在学习JavaScript异步编程的过程中，梳理出的一些个人理解应该必知必会的知识点。整篇笔记以这组[Asynchronous JavaScript](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Async_JS)MDN文档为主要参考依据，行文按照“同步->异步->Promise->async/await”的顺序，循序渐进、逐层剖析JavaScript Asynchronous的演进脉络。力争描绘出一幅完整的路线图，达到知其然亦知其所以然的目标。

## JavaScript Synchronous

同步代码逻辑直观、执行顺序明确、上手难度低，对于大多数编程语言而言，同步代码都是最常接触和使用的部分，当然JavaScript也不例外。

```javascript
function doStep1(init) {
  return init + 1;
}

function doStep2(init) {
  longRunningTask();
  return init + 2;
}

function doStep3(init) {
  return init + 3;
}

function longRunningTask() {
  // maybe several minutes
}

function doOperation() {
  let result = 0;
  result = doStep1(result);
  result = doStep2(result);
  result = doStep3(result);
  console.log(`result: ${result}`);
}

doOperation();
```

上述代码中，doStep1→doStep2→doStep3 的执行顺序严格同步，结果始终是 6（0+1+2+3），这体现了同步代码的可预测性。但当 doStep2 中包含longRunningTask这样的耗时操作时，同步代码的弊端就会显现：整个程序会卡在doStep2，直到该任务完成后才会执行doStep3，期间无法处理任何其他操作。在浏览器环境中，这种阻塞会导致页面无法响应点击、滚动等用户操作，甚至出现假死状态；在Node.js中，则会导致服务器无法处理新的请求，严重影响服务可用性。

面对同步代码带来的阻塞问题，不同的编程语言选择的异步思路也大相径庭：有的选择了多线程（Multi-threading）并因为其复杂性衍生出一整套工具集，Java就是其中的代表；有的选择了协程（Coroutine）并因为其轻量特点而体验颇佳，go就是其中的佼佼者；JavaScript早期通过回调函数（callback）实现异步，即便在单线程模式下也达成了不错的并发效果，后续又发展出Promise API、async/await语法糖，一步步降低了异步代码维护的心智负担，如今已经达到以同步代码的形式写异步代码的效果。

> JavaScript诞生之初是作为浏览器脚本语言，需要频繁操作DOM，若允许多线程同时修改DOM，会导致复杂的状态同步问题。因此JavaScript选择了单线程模式。

## JavaScript Asynchronous

曾经有很长一段时间，JavaScript都是通过回调函数（callback）实现异步的，随着时代的发展，因为callback hell等令人诟病的缺陷，callback实现异步的方案逐渐淡出人们的视野，Promise逐步成为JavaScript异步编程的主角。

```javascript
function doStep1(init, callback) {
  const result = init + 1;
  callback(result);
}

function doStep2(init, callback) {
  const result = init + 2;
  callback(result);
}

function doStep3(init, callback) {
  const result = init + 3;
  callback(result);
}

function doOperation() {
  doStep1(0, (result1) => {
    doStep2(result1, (result2) => {
      doStep3(result2, (result3) => {
        console.log(`result: ${result3}`);
      });
    });
  });
}

doOperation();
```

上述代码中，我们只能不停地在callback内部调用callback，callback hell已经初见雏形，这种代码难于阅读和调试，也很难便捷地进行错误处理。正是这一系列弊端催生了Promise的问世。

## JavaScript Promise

Promise是现代JavaScript异步编程的基石。它本身是对异步操作的当前状态的抽象，提供了一系列处理异步操作结果的链式调用方法，成功地把JavaScript开发者从callback hell中拯救了出来。

**Promise的创建**

```javascript
let promise1 = new Promise(function(resolve, reject) {
  setTimeout(()=>resolve("done"), 1000)
})

let promise2 = new Promise((resolve, reject) => {
  setTimeout(()=>reject(new Error("something wrong!")), 1000)
})
```

上述代码中，我们通过`new Promise`构造器创建Promise对象，`new Promise`构造器接收一个函数作为参数，这个函数叫做executor，executor函数的两个参数resolve和reject是JavaScript自身提供的回调函数。我们的业务代码就放在executor函数内部，当我们通过`new Promise`构造器创建Promise对象时，它的executor函数会自动执行，相应地我们的业务代码也会一起执行，当业务代码执行结束后，需要调用resolve或reject驱动Promise状态变更，比如上面的resolve("done")。

> 一旦忘记调用resolve或reject，Promise状态将永远停滞，这是一种常见的容易踩坑的地方，需要特别注意！！！

```javascript
let promise1 = Promise.resolve("done")

let promise2 = Promise.reject(new Error("something wrong! "))
```

上述代码中，我们通过Promise的静态方法创建Promise对象，这种方式下创建的Promise直接就是settled状态。这种方式主要用于兼容场景，比如我们包装一个缓存版本的fetch，无缓存时直接返回fetch的json解析结果（一个Promise对象），有缓存时返回Promise.resolve包装后的Promise对象。

```javascript
let response_cache = new Map();

export cachedFetch = (url) => {
  if (response_cache.has(url)) {
    return Promise.resolve(response_cache.get(url))
  }
  return fetch(url)
    .then(response => response.json())
    .then(json => {
      response_cache.set(url,json)
      return json
    })
}
```

> 其实自打async/await出现之后，需要用到这两个兼容方法的地方已经很少，它们几乎失去了用武之地。

**Promise的核心属性**

Promise对象有两个核心属性：state、result，result有三种类型：

- undefined：这是初始值，也是Promise尚未settled时的中间值
- value：当resolve(value)被调用时，result为value
- error：当reject(error)被调用时，result为error

state也有三种类型：

- pending：这是初始值，也是Promise尚未settled时的中间值
- fulfilled：当resolve(value)被调用时，state被置为fulfilled，该状态会触发then的第一个回调函数执行
- rejected：当reject(error)被调用时，state被置为rejected，该状态会触发catch的回调函数或then的第二个回调函数（如果有的话）执行
- fulfilled和rejected统称为settled，pending为unsettled（unsettled是一种约定俗成的叫法，非官方表述）

**Promise的链式调用**

Promise提供了三个链式调用方法，用来处理异步操作的结果：

- then：当Promise处于settled状态且state为fulfilled时触发（当然如果then有两个回调函数，那么rejected也可以触发）
- catch：当Promise处于settled状态且state为rejected时触发
- finally：当Promise处于settled状态时触发，主要用于资源清理

为什么可以进行链式调用呢？有两点关键原因：

- then、catch、finally这些方法本身返回Promise对象
- then、catch、finally返回的Promise对象内部，resolve或者reject阶段会把自身的回调函数队列通过`queueMicrotask`加入微任务队列

下面这段伪代码可以大致反映出Promise执行逻辑：

```javascript
// 伪代码以then函数为例
Promise.prototype.then = function(onFulfilled, onRejected) {
  const newPromise = new Promise((resolve, reject) => {
    const handleFulfilled = (value) => {
      queueMicrotask(() => {
        result = onFulfilled(value)
        if (isPromise(result)) {
          result.then(resolve, reject);
        } else {
          resolve(result);
        }
      });
    };
    const handleRejected = (reason) => {
      queueMicrotask(() => {
        result = onRejected(reason)
        if (isPromise(result)) {
          result.then(resolve, reject);
        } else {
          reject(result);
        }
      });
    };
    if (this.status === 'pending') {
      this.onFulfilledCallbacks.push(handleFulfilled);
      this.onRejectedCallbacks.push(handleRejected);
    } else if (this.status === 'fulfilled') {
      handleFulfilled(this.value);
    } else if (this.status === 'rejected') {
      handleRejected(this.reason);
    }
  });
  return newPromise;
};
Promise.prototype.resolve = function() {
  if (this.state === 'pending') {
    this.state = 'fulfilled';
    this.value = value;
    this.onFulfilledCallbacks.forEach(cb => cb(this.value));
  }
}
Promise.prototype.reject = function() {
  if (this.state === 'pending') {
    this.state = 'rejected';
    this.reason = reason;
    this.onRejectedCallbacks.forEach(cb => cb(this.reason));
  }
}
```

需要格外注意的是：

- Promise一旦从pending状态流转到settled状态，就无法再改变！
- Promise的then、catch、finally方法本身在同步执行阶段会立即返回一个新的Promise对象，假如为promise1
- Promise的then、catch、finally的回调方法在微任务执行阶段也会返回一个新的Promise对象，这又分为两种情况
  - 若回调方法返回值result不是Promise，那么JavaScript引擎会把返回值直接作为promise1的结果
  - 若回调方法返回值result本身就是Promise，假如为promise2，那么JavaScript引擎会把promise1的状态置为promise2的状态，这个就是所谓的“状态跟随机制”

  ```javascript
  // JavaScript引擎解析Promise回调函数返回值的伪代码
  function handleCallbackResult(result, resolve, reject) {
    if (isPromise(result)) {
      result.then(resolve, reject);
    } else {
      resolve(result);
    }
  }
  ```

- Promise的then、catch、finally的回调方法是在同步阶段立即注册到Promise对象的
- 每个Promise对象内部都有两个队列，其中PromiseFulfillReactions队列负责维护then回调函数，PromiseRejectReactions队列负责维护catch回调函数，当Promise被settled时，JavaScript引擎自动把then或catch回调函数添加到微任务队列。

**Promise与任务队列**

当Promise被settled之后，其then、catch和finally的回调函数会先被加入微任务队列，等到事件循环开始消费微任务队列时才会真正执行。对于fetch这种Web API返回的Promise对象，其settled过程则稍显复杂。由于fetch的实际工作是在独立线程内完成的，其处理结果是无法直接给到JavaScript引擎消费的，因而fetch返回的Promise并不会在网络请求完成后立即被settled，相反，它需要把结果处理回调函数包装成宏任务投入宏任务队列，等到事件循环消费宏任务队列，取出它的宏任务执行，才会把它的Promise置为settled状态。

```javascript
const fetchPromise = fetch(
  "https://mdn.github.io/learning-area/javascript/apis/fetching-data/can-store/products.json",
);

fetchPromise
  .then((response) => response.json())
  .then((data) => {
    console.log(data[0].name);
  });
```

上述代码中，同时涉及Promise、宏任务队列、微任务队列，其执行过程如下：

- 同步代码执行阶段
  - 调用fetch API，浏览器立即发起异步网络请求（由浏览器的网络线程处理），同时直接返回一个处于pending状态的Promise对象fetchPromise
  - 调用fetchPromise.then注册回调函数，同时直接返回一个处于pending状态的Promise对象，假定为promise1
  - 调用promise1.then注册回调函数，同时直接返回一个处于pending状态的Promise对象，假定为promise2
  - 至此同步代码执行结束，调用栈清空
- 宏任务执行阶段
  - 网络线程处理完毕后，会把请求结果包装成宏任务并放入宏任务队列，该宏任务负责驱动fetchPromise状态变更
  - 执行网络请求结果处理宏任务，resolve或者reject原fetchPromise
  - JavaScript引擎发现fetchPromise状态变更后，会把它的回调函数`(response) => response.json()`加入微任务队列
  - 至此宏任务执行阶段结束，调用栈清空
- 微任务执行阶段
  - 执行`(response) => response.json()`，json()是异步函数，JavaScript引擎立即发起json解析请求（由运行时的其他线程处理，不阻塞主线程），同时直接返回一个pending状态的Promise对象jsonPromise
  - json解析完成后，解析线程会把解析结果包装成微任务并放入微任务队列，该微任务负责驱动jsonPromise状态变更
  - 执行解析结果处理微任务，resolve或reject原jsonPromise
  - JavaScript引擎发现jsonPromise状态变更后，会把它的回调函数`(data) => console.log(data[0].name)`加入微任务队列
  - 执行`(data) => console.log(data[0].name)`，打印数据
  - 至此调用栈为空，全部执行结束

如果是使用Promise.resolve()创建Promise对象，其创建之初就是settled状态，那么它的执行过程会是什么样的呢？

```javascript
let promise1 = Promise.resolve();
let promise2 = Promise.resolve();

promise1
  .then(() => console.log(1))
  .then(() => console.log(2));

promise2
  .then(() => console.log(3))
  .then(() => console.log(4))
```

上述代码的执行过程如下：

- 同步代码执行阶段
  - 调用let promise1 = Promise.resolve()，直接返回一个处于fulfilled状态的Promise对象
  - 调用let promise2 = Promise.resolve()，直接返回一个处于fulfilled状态的Promise对象
  - 调用promise1.then注册回调函数`() => console.log(1)`，并直接返回一个pending状态的Promise对象，假定为p1，同时把回调函数加入微任务队列
  - 调用p1.then注册回调函数`() => console.log(2)`，并直接返回一个pending状态的Promise对象，假定为p2
  - 调用promise2.then注册回调函数`() => console.log(3)`，并直接返回一个pending状态的Promise对象，假定为p3，同时把回调函数加入微任务队列
  - 调用p3.then注册回调函数`() => console.log(4)`，并直接返回一个pending状态的Promise对象，假定为p4
  - 至此同步代码执行阶段结束，调用栈清空
- 微任务执行阶段
  - 执行回调函数`() => console.log(1)`，触发p1状态变更为fulfilled，进而触发p1的回调函数`() => console.log(2)`被加入微任务队列
  - 执行回调函数`() => console.log(3)`，触发p3状态变更为fulfilled，进而触发p3的回调函数`() => console.log(4)`被加入微任务队列
  - 执行回调函数`() => console.log(2)`，触发p2状态变更为fulfilled
  - 执行回调函数`() => console.log(4)`，触发p4状态变更为fulfilled
  - 至此微任务执行阶段结束，调用栈清空

## async/await

async/await是Promise的语法糖，通过它们你可以像写同步代码那样去使用异步函数，有了它们你可以更加舒适地同Promise协作。

**async**

当一个函数被标记为aync，那么就表明这个函数总是返回一个Promise对象，如果函数返回值不是Promise，那么会被包装成Promise，这也是前面提到的async/await出现之后Promise.resolve和Promise.reject失去用武之地的原因。

**await**

await只能在async函数内部使用，当一个Promise（异步函数返回值）前面加上await，那么就表明当前函数会一直暂停直到这个Promise处于settled状态。这其实就是then做的事情，只是await可以更优雅的做到这一点罢了。

比如前面这段Promise风格代码：

```javascript
const fetchPromise = fetch(
  "https://mdn.github.io/learning-area/javascript/apis/fetching-data/can-store/products.json",
);

fetchPromise
  .then((response) => response.json())
  .then((data) => {
    console.log(data[0].name);
  })
  .catch((error) => {
    alert(error)
  })
```

改写成async/await形式如下：

```javascript
async fecthProducts() {
  try {
    let response = await fetch("https://mdn.github.io/learning-area/javascript/apis/fetching-data/can-store/products.json")
    let json = await response.json()
    console.log(json[0].name);
  } catch (error) {
    alert(error)
  }
}
fetchProducts()
```

通俗的讲，Promise风格改写成async/await风格其实就是把then改为await，把catch改为try...catch块。