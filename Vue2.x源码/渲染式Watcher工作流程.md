## 整体流程

这节介绍下渲染`Watcher`，流程是从组件挂载开始的。

#### 组件挂载

在项目中我们通常是这样进行根组件的挂载

```javascript
  new Vue({
    ...options,
    render: h => h(App)
  }).$mount('#app')
```

首先是`Vue`实例化，这里会进行`initState`，执行`observe(data)`，这里涉及到[`Observer`的实例化](#Observer实例化过程)。

然后进行挂载，这里使用的`$mount`方法定义如下

```javascript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

最终是调用`mountComponent`方法，在这里我们关注其中的两处代码

```javascript
let updateComponent = () => {
vm._update(vm._render(), hydrating)
}

new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```

这里进行对`Watcher`实例化。

#### Watcher实例化过程

进入`constructor`函数后，主要做以下几个事情

- `this.before = options.before `
- `this.getter = updateComponent`
- 执行`this.get()`

分析`get()`执行过程

- 将该`Watcher`作为全局计算`Watcher`
- 执行`this.getter`也就是`updateComponent`，调用`_update`，此时会触发子元素的`getter`
  - 其中`_update(_vm.render())`
  - `render`将实例渲染为`VNode，会对`vm`数据进行访问，从而触发`getter`
  - ~~`updateComponent`调用链：`_update -> __patch__ -> createElm -> createComponent -> init(create-component.js) -> $mount `~~
  - 这部分涉及到组件的知识，目前我的理解是这样的，后续复习时再和这部分结合一下
  - 这里假设有一个数据的`getter`被触发，会执行`dep.depend`，当前`Watcher`中就会添加该`Dep`进`newDeps`，同时`Dep`会添加当前`Watcher`进`subs`。两者建立了关联
- 根据`this.deep`进行处理，这里不进行讨论。
- `popTarget`,将当前`Watcher`移除，使用栈底的代替。
- `cleanupDeps`，整理`deps`，这一步是在更新`Watcher`的`deps`，因为刚才的过程中是存入`newDeps`，需要进行新旧更替。这一步目的是优化，将新旧`deps`进行对比，如果有移除的`dep`，将该`Watcher`从`dep`中移除。
- 以上就完成了**依赖收集**这一步。
- 这里我是有疑问的，`targetStack`的作用是什么，为什么要保持单一的全局`Watcher`。
  - 我目前的理解是：在`Watcher`收集依赖的过程中，刚才提到会进行`updatecomponent`，所以这个过程中也会触发子组件`new Watcher`，将子组件的`Watcher`放入栈底，作为全局`Watcher`，这样像一棵树一样完成了依赖的收集。但是更新派发的过程中会怎么呢。

下面分析下**更新派发**流程

- 更新派发是由数据的更新触发的，所以在`getter`中开始分析。
- 先会进行新旧值的比较，如果相同是不会进行后续操作的。
- 最后执行`dep.notify`，这里实际上就是将该数据`dep`的`subs`中的`Watcher`执行`update`。
- `update`，根据`sync`，执行不同逻辑，这里[后续进行介绍](#TODO)。最后都是执行`Watcher.run`。
- `run`，渲染式`Watcher`中，执行`this.get`，这里我们之前进行了分析，就是调用`updateComponent`，即进行组件的更新，[更新的流程](#TODO)后续进行介绍。
- 所以在这里个人感觉上面的问题被解决掉，数据变化后通知的是最靠近的父节点组件进行组件更新。



#### Observer实例化过程

- `initState`中，会执行`observe(data)`，这一步是根据值创建`Observe`对象，并添加为数据的`__ob__`属性。以下为构造函数主要过程。
- `def(value, '__ob__', this)`首先是将实例挂载到数据的`__ob__`。
- 根据数据的类型做处理
  - 如果是数组类型，兼容`__proto__`的去更改数据的原型链，这一步的目的是[重写数组的一些方法](todo)，使其中元素能够变为响应式对象。再对数组中的元素调用`observe`方法，使得每个元素都有独立的`Observer`。
  - 如果是其他类型，执行`walk`方法
- `walk`，对对象属性的所有值执行`defineReactive`方法。
- `defineReactive`，这里就是至关重要的进行数据劫持的地方。这部分主要的对`getter` `setter`的操作已经在上面的依赖收集、更新派发中介绍了。



##### TODO

1. `Watcher.update`中根据`sync`，`Watcher`队列这部分的逻辑介绍
2. 派发更新后，组件更新流程
3. `Oberser`中为何要重写数组的方法