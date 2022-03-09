### 异步组件

​		异步组件，如其名，组件是通过异步进行加载的，所以我们在声明时，参数是一个异步函数。像下面的第二个例子，我们可以通过`import`或者`require`引入组件，再通过打包工具能够让该组件进行网络的异步加载，减少我们单次请求大小。

```javascript
Vue.component('async-example', function (resolve, reject) {
  setTimeout(function () {
    // 向 `resolve` 回调传递组件定义
    resolve({
      template: '<div>I am async!</div>'
    })
  }, 1000)
})

Vue.component('async-webpack-example', function (resolve) {
  require(['./my-async-component'], resolve)
})

// 可以处理加载状态的异步组件
const AsyncComponent = () => ({
  // 需要加载的组件 (应该是一个 `Promise` 对象)
  component: import('./MyComponent.vue'),
  // 异步组件加载时使用的组件
  loading: LoadingComponent,
  // 加载失败时使用的组件
  error: ErrorComponent,
  // 展示加载时组件的延时时间。默认值是 200 (毫秒)
  delay: 200,
  // 如果提供了超时时间且组件加载也超时了，
  // 则使用加载失败时使用的组件。默认值是：`Infinity`
  timeout: 3000
})

Vue.component('async-component', AsyncComponent)
```

​		在[组件创建](./Vue组件创建.md)中我们已经阅读到关于异步组件的源码处理，接下来主要介绍`resolveAsyncComponent`，他的目的是传入异步组件的异步函数，获得对应的构造函数。

```javascript
  // async component
  let asyncFactory
  // 如果函数没有cid，说明不是通过extend生成的构造函数，当作异步函数处理
  if (isUndef(Ctor.cid)) {
    // 这里的Ctor是注册时的异步函数
    asyncFactory = Ctor
    // 对resolveAsyncComponent单独进行
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
```

#### resolveAsyncComponent

```javascript
export function resolveAsyncComponent (
  factory: Function,
  baseCtor: Class<Component>
): Class<Component> | void {
  // 异步组件加载失败时，如果定义了error，返回error组件
  if (isTrue(factory.error) && isDef(factory.errorComp)) {
    return factory.errorComp
  }
  // 同上 对成功情况进行处理
  if (isDef(factory.resolved)) {
    return factory.resolved
  }
  // 当前在渲染的组件
  const owner = currentRenderingInstance
  // 能够到达这里说明工厂函数还处于pendding状态
  // owner是当前正在渲染组件 不在 factory.owners 中
  // 有新调用的，不同的父实例组件，push进owners数组
  if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
    // already pending
    factory.owners.push(owner)
  }
  // 如果是loading状态，且定义了loading组件，返回loading组件
  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
    return factory.loadingComp
  }
  // 这只会在第一次调用时执行，因为一旦生成了factory.owners，就无法进入该逻辑
  if (owner && !isDef(factory.owners)) {
    const owners = factory.owners = [owner]
    let sync = true
    let timerLoading = null
    let timerTimeout = null
	// 组件实例摧毁时，从owners中移除
    ;(owner: any).$on('hook:destroyed', () => remove(owners, owner))
	// 强制进行渲染刷新
    // 通知父组件进行刷新，也会导致异步组件刷新，从而进入resolveAsyncComponent函数
    const forceRender = (renderCompleted: boolean) => {
      // 强制更新父组件,父组件会更新子组件，又调用本函数。
      for (let i = 0, l = owners.length; i < l; i++) {
        (owners[i]: any).$forceUpdate()
      }
	  // 渲染结束，清空调用的父实例，清空定时器
      if (renderCompleted) {
        owners.length = 0
        if (timerLoading !== null) {
          clearTimeout(timerLoading)
          timerLoading = null
        }
        if (timerTimeout !== null) {
          clearTimeout(timerTimeout)
          timerTimeout = null
        }
      }
    }
	
    const resolve = once((res: Object | Class<Component>) => {
      // cache resolved
      // 缓存resolved结果，ensureCtor是根据异步函数结果进行构造器生成
      factory.resolved = ensureCtor(res, baseCtor)
      // invoke callbacks only if this is not a synchronous resolve
      // (async resolves are shimmed as synchronous during SSR)
      if (!sync) {
        forceRender(true)
      } else {
        owners.length = 0
      }
    })

    const reject = once(reason => {
      // 如果传入了error组件，标志为error，更新视图，更新为error组件
      if (isDef(factory.errorComp)) {
        factory.error = true
        forceRender(true)
      }
    })
	// 这里的factory是组件注册时的异步函数，我们在编写是就是那这里的resolve，reject进行处理
    const res = factory(resolve, reject)
	// typeof(res) === 'object' && res !== null
    if (isObject(res)) {
      // 是promise
      if (isPromise(res)) {
        // () => Promise
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject)
        }
      // 是可以加载状态的异步组件
      } else if (isPromise(res.component)) {
        // 通过resolve，reject函数去处理 异步
        res.component.then(resolve, reject)
        // 如果传入了error，把error当作加载失败时的组件，挂载到errorComp
        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor)
        }
	    // 如果传入loading，把loading组件挂载在loadingCopm
        // loading为延迟加载时间，timeout是超时时间
        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor)
          // delay为0，立刻为loading状态
          if (res.delay === 0) {
            factory.loading = true
          } else {
            // 定时器
            timerLoading = setTimeout(() => {
              timerLoading = null
              // 如果延迟结束仍然没有完成，更新视图为loading组件
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true
                forceRender(false)
              }
            }, res.delay || 200)
          }
        }
	    // 配置超时时间
        if (isDef(res.timeout)) {
          timerTimeout = setTimeout(() => {
            timerTimeout = null
            // 超时时间之后，还是没有完成，reject
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== 'production'
                  ? `timeout (${res.timeout}ms)`
                  : null
              )
            }
          }, res.timeout)
        }
      }
    }

    sync = false
    // return in case resolved synchronously
    // 返回要展示的组件
    return factory.loading
      ? factory.loadingComp
      : factory.resolved
  }
}
```

​		

##### once

```javascript
/**
 * Ensure a function is called only once.
 */
export function once (fn: Function): Function {
  // 标志位记录该函数是否已经调用
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```

​		该函数目的是函数只调用一次，通过闭包记录函数是否已经执行。这里确保`resolve`和`reject`只运行一次。

##### ensureCtor

```javascript
function ensureCtor (comp: any, base) {
  // 判断es模块，拿到正确的参数值
  if (
    comp.__esModule ||
    (hasSymbol && comp[Symbol.toStringTag] === 'Module')
  ) {
    comp = comp.default
  }
  // 参数进行构造器生成
  return isObject(comp)
    ? base.extend(comp)
    : comp
}
```

​		该函数目的是根据传入参数生成构造函数，对`es`模块进行了处理。

#### owner、owners是什么作用

​		`owner`是从`currentRenderingInstance`赋值，该值是从`render`过程中导出的，代表当前正在渲染的`Vue`实例。

​		`owners`指的是当前工厂函数所生成组件的调用父实例合集，而且在每个实例`destroy`时，将该实例移出`owners`。

​		对于这部分我的理解是，工厂函数被调用是因为父组件调用生成子组件，所以``currentRenderingInstance``就是调用的父组件。

#### resolveAsyncComponent函数运行流程

##### 第一次运行

- 对`error`和`resolved`的判断，因为第一次调用，`factory`并不会有该属性，直接跳过；
- `owner && isDef(factory.owners)`也因为`factory`无`owners`，略过；
- `factory.loadingComp`不存在，再次略过；
- 通过`owner && !isDef(factory.owners)`进入第一次运行的主要逻辑
  - 赋值`owner`给`factory.owners`数组
  - 增加事件监听，`owner`摧毁时，从`owners`数组中移除
  - 调用工厂函数，赋值给res
    - 这里有个疑问，我们传入的工厂函数可能没有返回值，`res`可能为`undefined`，不会进入后续流程了。
    - 看到这里我就停止思考了，感觉无法理解不进入后续流程，进而去寻找工厂函数是不是在哪里经过了封装处理能够返回值。
    - 经过一段时间调试后才明白，对于`(resolve, reject) => {}`这种方式的调用，确实是不进入`isObject(res)`的流程的，因为我们在工厂函数中调用的就是在这里定义的`resolve`、`reject`，使用该回调函数也会去走下面`Promise`的逻辑，这里是通过回调函数进行的处理。
  - `res`为`Promise`时，通过`.then`进行调用                                                                                                                                                                
    - `resolve`，`factory.resolved`赋值为根据工厂函数生成的构造函数
    - `reject`，`factory.error = true`，通知`owners`进行更新。二次调用时就能直接在开始的`error`判断中进行拦截，返回`error`组件。
  - `res`为处理加载状态的异步组件，对`res.component`使用`.then`进行相应的处理。因为有其他参数状态，对其他参数进行处理
    - `error`，加载失败时使用的组件。将`error`使用`ensureCtor`创建加载失败组件的构造函数，赋值给`errorComp`
    - `loading`，异步组件加载时使用的组件。将`loading`使用`ensureCtor`创建加载失败组件的构造函数，赋值给`loadingComp`。
      - 当`delay === 0`时，直接置为`loading`状态。这里单独把逻辑提出来，不通过`setTimeout`统一执行。
      - 延迟`delay`毫秒，执行回调函数，作用是延迟一定时间后置`loading`为`true`，**这里的`delay`并不是将整个异步组件加载流程进行推迟，仅仅是将loading状态进行延迟展示，如果这个期间异步加载已经完成，则直接展示完成状态**
      - 当使用`loading`进行延迟时，`resolveAsyncComponent`返回为`factory.resolved`，为`undefined`，所以此时返回`undefined`，对于`createComponent`来说会使用`createAsyncPlaceholder`创建一个注释节点。
    - `timeout`，设置`timeout`后会设置定时器，如果定时器结束仍然没有`factory.resolved`，说明已经异步组件加载仍然是未完成或者已经`reject`。
      - 对于未完成情况，就是已经超时了，此时进行`reject`操作
      - 对于`reject`情况，在`reject`时候会执行`forceRender(true)`，这会清空`loading`和`timeout`两个定时器，不会执行到该代码
      - 按道理来说完成情况下定时器已经被清除了，无需进行`factory.resolved`判断。个人猜测是因为当组件刚转换为`resolved`状态时，首先是挂载`factory.resolved`，但是在`forceRender`过程中，有可能`timeout`定时器到了，开始执行`timeout`的回调，如果没有`resolved`的判断，就会更新状态为`error`。

##### 第二次运行

- 当`resolved`、`reject`、`loading`时会调用`forceRender`，进行父组件的重新渲染，也就会导致异步组件的重新渲染，重新进入`resolveAsyncComponent`函数。
  - `reject`，直接判断上次是否（使用传入的`error`）赋值了`errorCopm`，返回`error`组件。
  - `resolved`，返回我们预期的异步组件。
  - `loading`，判断是否（通过传入的`loading`）赋值了`loadingComp`，返回`loading`组件

#### loading === 0时为什么要单独抽出来，不直接使用settimeout

​		如果`loading`为`0`时候，仍然进入setTimeout流程，首先会创建注释节点，定时器结束后，又会重新进入函数并使父组件重新渲染，从而进入该函数，再重新更新为`loading`组件，这里应该是为了性能的优化，避免重复渲染。