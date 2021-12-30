​		首先回顾下`Vue`的构造函数

```javascript
function Vue (options) {
  this._init(options)
}
```

​		在`_init`方法中调用了不同的初始化方法，其中在`initState`方法中进行了`computed`的初始化，相关代码如下

```javascript
export function initState (vm: Component) {
  const opts = vm.$options
  if (opts.computed) initComputed(vm, opts.computed)
}
```

​		接下来对`initComputed`主干代码进行分析

```javascript
const computedWatcherOptions = { lazy: true }
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
}
```

​		在使用过程中，我们知道`computed`有两种方式进行定义`computed: { [key: string]: Function | { get: Function, set: Function } }`。都是对象类型，键为字符串，值分为两种， 函数 或者 具有`get` `set`函数的对象。

​		首先遍历定义的`computed`，每个值的处理流程如下：

- 拿到真正的`getter`函数，（如果是函数取函数，否则取其中的`getter`属性）
- 实例化`Watcher`
- 校验当前值是否在`data`或`props`中定义，都不存在的话调用`defineComputed`方法

#### 实例化watcher

​		`computed`的`Watcher`不同之处在于传入的`{lazy: true}`。再到`Watcher`构造函数中查看处理的不同之处。

**Watcher构造函数中**

```javascript
this.value = this.lazy ? undefined : this.get()
```

当是`computed Watcher`时，不执行`this.get`，值是惰性进行计算的

**Watcher.update**

```javascript
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

​		在`update`函数中，由于`lazy`的原因，该处动作仅仅是将`dirty`置为`true`。

​		再看看`dirty`相关处理，初始化时，`this.dirty = this.lazy`。还涉及到一处代码如下

```javascript
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }
```

​		这里通过`this.get()`更新`value`的值，然后将`dirty`置回`false`



#### defineComputed方法

​		我们不分析服务端渲染以及Vue1.x中cache的情况，所以代码流程如下：

```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = createComputedGetter(key)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? createComputedGetter(key)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

​		该方法主要目的是通过`defineProperty`对`key`进行代理。这里主要是对代理的`get`和`set`进行处理。对`get`的处理，使用`createComputedGetter(key)`进行构建。

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

**watcher.evaluate()**		

​		首先我们思考一个问题，这里的函数何时被触发，既然他是`get`函数，那就是对`computed`的该属性进行访问时触发。当渲染时，如果该`computed`在视图上有体现，会访问该属性值，执行`get`函数。

​		根据传入的`key`值，拿到对应的`Watcher`对象。

​		初始时，`dirty === true`，执行`evaluate`()，执行`Watcher.get()`，此时会将当前`Watcher`作为全局`Watcher`，然后执行我们写在`computed`的计算方法，拿到计算值。在这个过程中，函数内会触发其他值的`get`函数，从而达到将该`Watcher`加入到依赖值的`dep`中，达到建立依赖的目的。在`evaluate()`结尾，`dirty`置为`false`。后面如果依赖值更新时，会通知`computed`值进行更新，`computed Watcher`执行`update`方法，还原`dirty`为`true`，下次访问计算属性时才会重新求值。

**watcher.depend()**

​		这里判断`Dep.target`是因为在`targetStack`中刚把`computed Watcher`给`pop`出来，所以下一个`Watcher`是触发`computed Watcher`的渲染时`Watcher`，而且此时当前全局`Watcher`已经变成了渲染`Watcher`。执行`watcher.depend()`是为了让依赖的`Watcher`将当前渲染`Watcher`加入依赖。这样在`computed Watcher`依赖项发生改变时，进行更新派发过程中，因为添加的顺序，所以先会通知`computed Watcher`，置`dirty`为`true`，使其可重新求值；然后通知渲染`Watcher`，使`computed Watcher`重新计算求值。

**return watcher.value**

​		执行完上面的步骤后，`computed Watcher`的`value`已经更新，返回更新后的值。

​	

##### `dirty`的意义

`dirty`值初始化为`true`

- 它控制了哪里的逻辑
  - `if (watcher.dirty) watcher.evaluate()`，只有`dirty === true`时，才能够去值的重新计算。

- 值的转变有两个地方
  - `Watcher.update()`中置为`true`
    - 何时触发？依赖进行更新派发时，只有依赖项发生改变时才会触发，这也意味着`computed`值可以进行重新计算。
  - `Watcher.evaluate()`中置为`false`
    - 意味着新的`computed`值已经计算完。

所以`dirty`的作用在于优化、减少计算，如果访问该`computed`值时，该值并没有发生更新变化，`dirty === false`，则不会进行`computed Watcher`的重新计算。

