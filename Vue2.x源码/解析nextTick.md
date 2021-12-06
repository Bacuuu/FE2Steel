​		在[Watcher执行逻辑](./Watcher的执行逻辑.md)里，我们提到了`nextTick`函数，用于控制`Watcher`的统一执行时刻。本章就来介绍下`nextTick`，他的位置在`util\next-tick.js`。从文件位置可以看出尤大把他放到了工具类中，这是因为它的实现和Vue非耦合的，也可以单独拿出来放到我们项目中使用的。

#### nextTick

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

​		`callbacks: Array`是全局变量，向该数组中传入函数，函数作用：当我们传入`cb`时，执行`cb`，否则执行`_resolve(ctx)`。读到这里，很容易知道`callbacks`是一个储存回调的队列，后续肯定会进行统一的执行。但是当没有传入`cb`时候，这里的`_resolve(ctx)`有什么用呢？

​		关于`_resolve`定义在后面，如果`Promise`存在，`_resolve`就是入参的`resolve`，使该`promise`完成，并且整个`nextTick`函数返回了这个`promise`。我再去看了下Vue文档，发现如下内容。

```javascript
// 修改数据
vm.msg = 'Hello'
// DOM 还没有更新
Vue.nextTick(function () {
  // DOM 更新了
})

// 作为一个 Promise 使用 (2.1.0 起新增，详见接下来的提示)
Vue.nextTick()
  .then(function () {
    // DOM 更新了
  })
```

> 2.1.0 起新增：如果没有提供回调且在支持 Promise 的环境中，则返回一个 Promise。请注意 Vue 不自带 Promise 的 polyfill，所以如果你的目标浏览器不原生支持 Promise (IE：你们都看我干嘛)，你得自己提供 polyfill。

​		原来`nextTick`不仅可以通过传入回调方式，还能够通过返回`Promise`执行。于是`_resolve`有关内容就清楚了，不传入首个回调函数参数时，`nextTick`执行后返回一个`fulfilled`状态的`Promise`，通过`then`去做后续操作。

​		还剩下下面这段代码，`timeFunc`本质是调用`flushCallbacks`。

```javascript
  if (!pending) {
    pending = true
    timerFunc()
  }
```



#### flushCallbacks

```javascript
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

​		该函数复制`callbacks`后，遍历执行每个函数，所以该函数目的就是执行并清空`callbacks`。在刚进入函数时将`pending`置为`false`。下面再分析下`pending`的作用。

​		`pendding`进行变动的时间点有两个部分：执行`callbacks`时，变为`false`；`nexttick`传入回调时，当`pending`为`false`，置为`true`，并执行`timeFunc`，最终是调用`flushCallbacks`。这样做的目的是`flushCallbacks`不会重复调用，只有执行之后才能够继续触发调用。下方是 ~~章鱼哥~~ 一个大致流程图。

![image-20211206224703855](https://cdn.jsdelivr.net/gh/Bacuuu/sleeping-image@master/uPic/image-20211206224703855.png)

#### timerFunc

​		上面反复提到的`timerFunc`到底是在做什么呢，其实他的目的是做微任务的降级兼容，将`flushCallbacks`放入微任务队列中，便于在宏任务执行后，渲染前执行。降级顺序如下：

- Promise.then，直接将`flushCallbacks`放入`.then`回调中作为微任务执行。
- [MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)，这个接口不常用，它提供了监视对DOM树所做更改的能力。这里将`flushCallbacks`作为DOM更改后的回调，然后创建一个`textNode`，指定其监听这个DOM，`timerFunc`的作用就是每次去修改`textNode`，以触发回调执行（作为微任务）。
- [setImmediate](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setImmediate)，这里是将回调作为宏任务去执行。该方法定义是，用来把一些需要长时间运行的操作放在一个回调函数里，在浏览器完成后面的其他语句后，就立刻执行这个回调函数。
- setTimeout，这里也是作为宏任务执行，将延迟时间设置为0（但实际上最低是4ms）。~~下下策了属于是~~


​		可以看出，`nextTick`作用是将回调放入维护的队列中，执行时再将这些任务作为微任务执行，使其在当前宏任务执行结束，渲染前执行；兼容性差的情况会使用`setImmediate`或`setTimeout`。
