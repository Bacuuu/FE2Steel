本文整理自 [掘金《「硬核JS」一次搞懂JS运行机制》](https://juejin.cn/post/6844904050543034376)

渲染进程的主要线程

- GUI渲染线程，主要负责渲染浏览器界面
- JS引擎，负责解析`javascript`脚本
- 事件触发线程，控制事件循环
- 定时触发线程，`setInterval` `setTimeout`，`setTimeout`最小间隔为4秒
- 异步http请求线程，`XMLHttpRequest`。

GUI和JS引擎是互斥的，JS线程执行时，GUI线程被挂起。



#### 事件循环

同步任务，位于主线程（js引擎）执行，形成执行栈。

异步任务，当异步任务运行完成后会将结果添加到 **任务队列**，作为事件回调

当同步任务执行结束后（JS引擎线程空闲），任务队列中的回调会被添加到执行栈。

所以JS引擎工作流程可以总结为：执行执行栈中的事件，如果有异步任务则丢进任务队列，执行栈为空之后读取事件队列（异步任务）并添加到执行栈中。



#### 宏任务、微任务

常见的宏任务

- 主代码块
- setTImeout
- setInterval
- setImmediate -- Node
- requestAnimationFrame -- Browser

宏任务 和 DOM任务 运行顺序

​	宏任务 -> GUI渲染 -> 宏任务 -> GUI渲染 -> ...



宏任务执行后，GUI渲染前，会先执行微任务。

常见微任务

- process.nextTick -- Node
- Promise.then()
- catch
- finally
- Object.observe
- MutationObserver



宏任务、微任务执行如下

- 整体代码作为第一个宏任务执行，在这个过程中会产出新的宏、微任务；

- 第一个宏任务执行结束后判断是否存在微任务；

- 有微任务先执行所有的微任务，再渲染，没有微任务则直接渲染；

- 从产生的宏任务队列中，取出接着执行下一个宏任务。

```javascript
console.log('macro 1 start') // 1
setTimeout(() => {
	console.log('macro 2 start') // 4
    new Promise(rs => {
    	rs()       
    }).then(() => {console.log('micro 2 start')}) // 5
})
new Promise((rs) => {
	console.log('in Promise') // 2
	rs()
})
.then(() => {
	console.log('micro 1 start') // 3
	setTimeout(() => { console.log('macro 3 start') }) // 6
})
```

分析下上述的代码执行过程

1. 整段代码作为第一个宏任务开始执行，输出**macro 1 start**

2. 定时器，将定时器放入宏任务队列中

3. Promise，先执行内部逻辑，输出**in promise**，同时Promise置为`fulfilled`完成状态，将回调放入微任务队列中

   

   至此第一段宏任务执行结束，接下来应该取出微任务队列中任务，执行。

1. 任务队列中只有一个由`Promise`产生的回调，先输出**micro 1 start**

2. 然后又是一个定时器，将该定时器放入宏任务队列中。

   

   至此第一段微任务队列清空，该执行下个宏任务。宏任务队列中现在有两个任务，因为没有设置定时时间，所以两个任务都将回调放入队列中待执行。

1. 取出宏任务队列中的第一个任务执行，输出**macro 2 sttart**
2. Promise，内部直接完成(resolve)，放入微任务队列



​	第二阶段宏任务完成，执行微任务

1. 输出**micro 2 start**



​	第三阶段宏任务开始

1. 输出**macro 3 start**



由此可以看出，宏任务是一个一个执行的，每个执行结束后去将微任务队列中任务全部执行，再执行下一个宏任务。

