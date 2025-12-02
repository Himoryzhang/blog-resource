---
title: 《一天理解 Javascript Promise》读书笔记
description: 梳理 Promise 生命周期、链式调用、多 Promise 协作与 async/await 等核心知识。
pubDate: 2024-10-18 21:05:00
---

这本书的作者是Nicholas C Zakas。你可能对这个名字没有印象，但是他的作品你一定知道——《JavaScript 高级程序设计》，一本前端从业者必读的书，大家称其为「红宝书」。

作者发现JS语言过于庞大，很难有一本专门的书去记录整个JS语言。于是作者决定对于JS中的某个特定主题编写一本书，以帮助开发人员理解这门技术。

本书短小精悍，主要内容为技术人员如何使用Promise，一共有五个章节：

1. Promise基础
2. 链式Promise
3. 多个Promise协同工作
4. 异步函数和await表达式
5. 追踪未处理的拒绝情况

## 1. 第一章 Promise基础

本章对Promise的基础概念进行介绍。

### 1.1 Promise的生命周期

每个Promise都有一个生命周期，该生命周期存在三种状态
- **pending**：未完成。Promise的开始状态，后续根据Promise的执行结果，会进入到下述两种状态。
- **fulfilled**：成功完成。
- **rejected**：被拒绝

无法通过编程的方式直接获取Promise的状态，但是Promise状态变化后会触发处理器，可以在处理器中执行对应状态下的操作。

### 1.2 Promise 是 thenable 的

Promise对象具有包含 fulfilled handler 和 rejected handler的then方法。包含此类then方法的对象都可被称为thenable。

例如，用于发送请求的 `fetch()` 函数，也是thenable的。

### 1.3 创建Promise

#### 1.3.1 创建处于pending状态的Promise：new Promise()

Promise 通过构造函数创建。

```javascript
new Promise((resolve, reject)=>{
	// ...
})
```

Promise构造函数接收的回调被叫做执行器（executor），包含两个入参（resolve和reject方法）
- **resolve方法**：将Promise从pending状态转变到fulfilled状态。
- **reject方法**：将Promise从pending状态转变到rejected状态。

在创建Promise时，executor会立即执行。

当不确定需要处理的对象是否为Promise时，通过Promise.resolve传递该对象是获取该对象值的最佳方法。

#### 1.3.2 创建fulfilled状态的promise：Promise.resolve()
- 若入参是一个Promise，那么会将该Promise返回，二者严格相等。
- 若入参是非Promise的thenable对象，那么会执行该对象的then方法。

示例：
```javascript
var fulfilledThenable = {
	then(resolve, reject){
		resolve("fulfilled state")
	}
}
Promise.resolve(fulfilledThenable) // Promise {<fulfilled>: 'fulfilled state'}
Promise.resolve(fulfilledThenable).then(v=>{ console.log(v) }) // fulfilled state

var rejectedThenable = {
	then(resolve, reject){
		reject("rejected state")
	}
}
Promise.resolve(rejectedThenable) // Promise {<rejected>: 'rejected state'}
Promise.resolve(rejectedThenable).then((v)=>{}, (v)=>{ console.log(v) }) // rejected state
```


#### 1.3.3 创建rejected状态的promise：Promise.reject()

该方法会创建一个状态为rejected，reason为传入对象的Promise。不会对thenable对象等任何传入内容做特殊处理，从而避免了传入的参数影响错误处理行为。

示例：
```javascript
var fulfilledThenable = {
	then(resolve, reject){
		resolve("fulfilled state")
	}
}

Promise.reject(fulfilledThenable) // Promise {<rejected>: {…}}，即调用reject创建的新Promise
Promise.reject(fulfilledThenable).catch( v => { console.log(v) }) // {then: ƒ} 即传递的 fulfilledThenable 对象
Promise.reject(fulfilledThenable).catch( v => { v.then(res => { console.log(res) }) }) // fulfilled state，即fulfilledThenable对象的执行结果。

var rejectedThenable = {
	then(resolve, reject){
		reject("rejected state")
	}
}
Promise.reject(rejectedThenable) // Promise {<rejected>: {…}}
Promise.reject(rejectedThenable).catch( v => { console.log(v) }) // {then: ƒ}
Promise.reject(rejectedThenable).catch( v => { v.then(()=>{}, res => { console.log(res) }) }) // rejected state
```


### 1.4 处理 Promise

Promise具有许多方法，使开发人员可以方便地处理不同状态下的业务逻辑。

这些方法的参数被称为处理器（Handler）。所有Handler都会被添加到微任务队列中，作为微任务执行。

这些方法具有语义，选取恰当的方法处理状态最好。

#### 1.4.1 then

表示Promise从pending状态转移到下一个状态。

同时具有fulfilled handler 和rejected handler。

#### 1.4.2 catch

表示Promise被拒绝，处于rejected状态。

只处理rejected状态，只有rejected处理器。

#### 1.4.3 finally

表示当Promise处于fulfilled或是rejected时，执行相同的业务逻辑。

- 成功和失败都只有一个回调
- 回调函数没有入参，无法确定promise的状态
- 无法捕获异常，因此仍然需要一个rejected处理器

应用场景：隐藏loading

## 2. 第二章 链式Promise
链式Promise主要指Promise对象的then、catch、finally等方法返回的值仍然是一个Promise，因此可以形成链式调用。

利用链式调用，可以实现延迟处理器的执行时间。只需在处理器中创建一个新的promise。
```javascript
const p1 = Promise.resolve(42);
p1.then((value) => {
	console.log(value); // 42
	const p2 = new Promise((resolve) => {
		setTimeout(() => {
			resolve(43)
		}, 500)
	})
	return p2;
}).then((value) => {
	console.log(value);
})
```

始终在链式Promise的末端添加一个 `catch` ，这样可以确保对Promise链可能发生的任何错误进行处理。

## 3. 第三章 多个Promise协同工作

Promise提供了一些API，用于处理多个Promise协同工作的场景。

### 3.1 Promise.all

参数中的全部Promise处于fulfilled状态，则该方法的结果视为fulfilled。若某个Promise返回rejected，则该方法立刻返回rejected。

参数中任何非Promise值都会被传递给Promise.resolve()。

#### 3.1.1 使用场景
- 同时处理多个文件
- 调用多个相互依赖的web服务api
- 人为引入延迟，以避免loading动画过短

#### 3.1.2 人为引入延迟示例
```javascript
function delay(ms){
	return new Promise((resolve, reject)=>{
		setTimeout(()=>{
			resolve();
		}, ms)
	})
}
Promise.all([....requests, delay(500)]).then((results)=>{
	return results.slice(0, results.length - 1);
})
```

### 3.2 Promise.allSettled

不论参数中的Promise处于fulfilled还是rejected，一旦状态全部确定，则返回结果。返回的对象数组中的对象有两个属性，status和value。

#### 3.2.1 使用场景
- 分别处理多个任务。
- 调用多个独立的web服务api。
- 播放动画。无需关注每个动画的执行结果，仅需在全部动画结束后关闭即可

#### 3.2.2 分别处理多个任务示例
```javascript
function runTask(task) {
	return task();
}

function executeTasks(tasks){
	return Promise.allSettled(tasks.map(runTask))
}

const tasks = Array.from({length: 10}).map((_, idx)=>{
	return () => new Promise((resolve, reject) => {
		const result = Math.random() >= 0.5;
		if(Boolean(result)){
			return resolve(idx);
		}
		return reject(idx);
	});
})

executeTasks(tasks).then(results => {
	const successTasks = results.filter(r => r.status === 'fulfilled').map(r=>r.value);
	const failedTasks = results.filter(r => r.status === 'rejected').map(r=>r.reason);
	console.log('Sucess tasks', ...successTasks);
	console.log('Failed tasks', ...failedTasks);
})
```

### 3.3 Promise.any

若有一个promise处于fulfilled状态，则返回该promise的值；否则若所有promise都处于rejected状态，则返回处于rejected状态的promise。

#### 3.3.1 使用场景
- 执行对冲请求（客户端向多台服务器发送请求，并接受第一个回复的响应），有助于在有服务器资源专门用于管理额外负载和重复响应的情况下，最小化客户端延迟。
- 在service worker中使用最快速的响应。使用service worker的网页可以选择从网络或是缓存中加载数据

#### 3.3.2 对冲请求示例
```javascript
const Hosts = ["api1.example-url", "api2.example-url"];

function hedgedFetch(endpoint) {
	return Promise.any(Hosts.map(hostname => fetch(`https://${hostname}${endpoint}`)))
}

hedgedFetch('/endpoint').then((result)=>{
	console.log(result);
}).catch(reason => console.error(reason.message))
```

### 3.4 Promise.race

返回第一个确定状态的Promise。

#### 3.4.1 使用场景

为操作设置超时

```javascript
function timeout(ms) {
	return new Promise((resolve, reject)=>{
		setTimeout(()=>{
			reject(new Error("Request timed out"));	
		}, ms)
	})
}

function fetchWithTimeout(args) {
	return Promise.race([
		fetch(...args),
		timeout(5000)
	])
}

fetchWithTimeout(url).then(....).catch(....);
```

## 4. 第四章 异步函数（async function）和await表达式

本章主要介绍async和await的用法。

### 4.1 异步函数的特点

- 返回值总是一个Promise
- 抛出的错误总是处于拒绝状态的Promise
- 异步函数内部的try catch表达式可以捕获await表达式抛出的错误
- await表达式可以运用于多Promise表达式，从而达到并行处理多个promise的目的。
- 可以使用for-await-of循环来达到串行处理多个promise的目的。

### 4.2 for await of 循环

该循环总是对从iterable对象中获取的值调用promise。

```javascript
const promise1 = Promise.resolve(1);
const promise2 = Promise.resolve(2);
const promise3 = Promise.resolve(3);
const promiseArr = [promise1, promise2, promise3];

for await (const value of promiseArr) {
	console.log(value);
}
```

值得注意的是，如果将 `promiseArr` 替换为普通数组，该循环仍然能正常执行，因为数组中的每一项会被 `Promise.resolve()` 处理。

### 4.3 顶层await表达式

- 可以在js模块的顶层，即异步函数之外使用await表达式。
- 当js引擎遇到顶层await表达式时，js模块会暂停执行，直到该Promise变为fulfilled状态。
- 为了使用顶层await表达式，必须使用import或是`<script type="module">`来加载js代码。

## 5. 第五章 追踪未处理的拒绝情况

针对 rejected 状态的 Promise 的最佳实践。

### 5.1 Rejected Promise 相关事件
- **unhandledrejection**：Promise被拒绝，且在事件循环的一个回合内没有 Rejected Handler 被调用，globalThis对象会发出该事件。
- **rejectionhandled**：Promise被拒绝，且在事件循环的一个回合后被 Rejected Handler 调用，globalThis对象会发出该事件。

二者回调函数的入参都为包含以下属性的事件对象：
- **type**：事件名称
- **promise**：被拒绝的Promise对象
- **reason**：拒绝值

### 5.2 寻找相关Rejected promise对象
```javascript
// 捕获未处理的rejected promise
globalThis.onunhandledrejection = (event) => {
	...
}
// 捕获被处理的rejected promise
globalThis.onrejectionhandled = (event) => {
	...
}
```

### 5.3 使用场景

#### 5.3.1 在浏览器中报告未处理的拒绝情况

```javascript
const possiblyUnhandledRejections = new Map();

globalThis.onunhandledrejection = event => {
	possiblyUnhandledRejections.set(event.promise, event.reason);
}
globalThis.onrejectionhandled = event => {
	possiblyUnhandledRejections.delete(event.promise);
}
setInterval(()=>{
	possiblyUnhandledRejections.forEach((reason, promise) => {
		console.error("Unhandled rejection");
		console.error(promise);
		console.error(reason.message ? reason.message : reason);
	});
	possiblyUnhandledRejections.clear();
}, 60000)
```

#### 5.3.2 在浏览器中避免出现控制台警告

```javascript
globalThis.onunhandledrejection = event => {
	// 避免出现控制台警告
	event.preventDefault();
}
```

#### 5.3.3 捕获未处理的拒绝情况并进行处理

```javascript
globalThis.onunhandledrejection = ({promise, reason}) => {
	// 处理拒绝情况
	promise.catch(()=>{});
}
globalThis.onrejectionhandled = ({promise}) => {
	// 这部分永远不会被调用
	console.log(promise);
}
```

#### 5.3.4 在Node.js中追踪未处理的rejected promise

```javascript
process.on("unhandledRejection", (reason, promise) => {
	// 使脚本出现未处理的拒绝时崩溃。
	throw reason;
})
process.on("rejectionHandled", (promise) => {
	...
})
```

