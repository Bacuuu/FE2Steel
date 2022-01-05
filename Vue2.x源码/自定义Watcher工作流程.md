		我们直接看初始化`watch`的地方。还是定义于`initState`函数中

```javascript
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
```

​		简短的初始化代码，只是这里的`nativeWatcher`是什么呢，我们查看定义找到

```javascript
// Firefox has a "watch" function on Object.prototype...
export const nativeWatch = ({}).watch
```

​		从尤大的注释中可以知道，在火狐浏览器中`Object`原型链中有名为`watch`的函数，这里就是为了避免识别错误的情况。

#### initWatch

```javascript
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

​		由于我们可以对多个对象进行监听，所以遍历整个`watch`对每个对象单独处理。

​		这里可以先询问下自己，对于`watch`而言，可以有哪些类型。

```javascript
watch: {
	// 字符串 
	a: 'someMethod',
	// 函数
	b: function () {},
	// 对象
	c: {
		handler: function () {},
		// otherOptions
	},
	// 数组
	d: [
		'someMethod',
		function () {}
	]
}
```

​		在日常的开发任务中，我们常用前三种类型，第四种我也是在读完相关源码后，再去查看官方文档才发现有这样一种用法，官方的描述如下。

> 你可以传入回调数组，它们会被逐一调用

​		回到源码中，如果是数组的情况，将数组中每个元素都使用`createWatcher`进行处理；其他类型则直接使用`createWatcher`进行处理。

#### createWatcher

```javascript
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

​		这里实际上是对上面提到的前三种类型做处理。

- 对象，取其中的`handler`作为回调函数，整个作为`options`
- 字符串，默认为是挂载在`vm`上，我理解为对应`methods`的方法
- 函数，直接作为回调进行使用

​		最终都会经过`$watch`方法处理，这个方法也是Vue对外暴露的API之一。

#### $watch

```javascript
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      const info = `callback for immediate watcher "${watcher.expression}"`
      pushTarget()
      invokeWithErrorHandling(cb, vm, [watcher.value], vm, info)
      popTarget()
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```

- 这里也是通过`Watcher`实例化时参数的差异化进行自定义`Watcher`的声明，这里的`options.user = true`。
- 当我们声明的`watch`具有`immediate === true`时，执行了对`Watcher`出栈、入栈的逻辑处理。
- 返回取消监听数据的方法，内部调用`Watcher.teardown`。

##### new Watcher()

​		这里我们对差异化的地方，`{user: true}`进行相关逻辑的介绍。真正对功能的影响在于`Watcher.run()`中

```javascript
if (this.user) {
  const info = `callback for watcher "${this.expression}"`
  invokeWithErrorHandling(this.cb, this.vm, [value, oldValue], this.vm, info)
} else {
  this.cb.call(this.vm, value, oldValue)
}
```

​		这里执行的是`invokeWithErrorHandling`函数，该函数在`immediate`时也进行了调用。

```javascript
function invokeWithErrorHandling (
  handler: Function,
  context: any,
  args: null | any[],
  vm: any,
  info: string
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```

​		可以看到，该函数实际上就是对回调函数的执行，中途增加了更多的异常捕获操作。[这里的#9511有机会再去学习下](#TODO)

​		回顾下调用过程，`dep.notify`时会调用`Watcher.update`，进而执行`run`方法，从而调用我们写入的回调函数。那么依赖收集是什么时候进行的呢。

​		在`Watcher`的构造函数中，调用了`this.getter`，会对需要监听的数据进行访问。访问时，会触发被监听数据的`get`，因为是`reactive`的，所以会进行依赖的添加。当该值发生变化时，进行依赖分发。

##### immediate

> 在选项参数中指定 `immediate: true` 将立即以表达式的当前值触发回调

​		具体代码逻辑也很简单，通过`invokeWithErrorHandling`进行回调`cb`的执行。至于`pushTarget`和`popTarget`，我的认知是：由于在这个过程中会涉及到对相关`data`的访问，所以将全局`Watcher`置空，避免进行非预期的依赖收集。

##### Watcher.teardown

​		该方法将该`Watcher`从Vue实例中移除，并解除该`Watcher`的依赖`dep`相关关系。最后将`this.active`置为`false`，使得不能执行`run`方法。



##### 关于deep参数

​		在`Watcher.get`方法中，如果`deep`，则会执行`traverse(value)`，目的是进行值的深层递归遍历，触发每个值的`get`，进行依赖的收集。[后续对traverse进行学习](#TODO)



#### TODO

- 关于`invokeWithErrorHandling`方法
- 关于`traverse`方法