## Observer

#### 实例属性

- `value`
- `dep`

#### 构造函数

- 对`value`执行`walk`方法
- 如果是数组，遍历执行`observe`方法，因为该对象不一定有`__ob__`

#### 实例方法

- `walk`
  - 遍历对象，通过`defineReactive`将对象的每个属性变为响应式对象
- `observeArray`
  - 遍历数组，将数组中元素都执行`observe`方法

#### 相关其他方法

- `observe(value, asRootdata)`
  - 作用：为`value`的添加属性`__ob__`，为`Observer`对象
  - 返回：`Observer`对象
- `defineReactive(obj, key, val, ?rest)`
  - 作用：为`obj`设置`getter`和`setter`，用于依赖收集和更新派发
    - `getter`：
    - `setter`：



## Dep

#### 静态属性

- `Watcher`，当前在计算的Watcher

#### 实例属性

- `subs:[Watcher]`

#### 实例方法

- `addSub`
  - 向`this.subs`中添加`Watcher`
- `removeSub`
  - 从`this.subs`删除`Watcher`
- `depend`
  - 向当前计算的`Watcher`中添加该`Dep`
- `notify`
  - 复制当前`subs`，对`subs`中的`watcher`遍历，执行`update`

#### 文件中内部变量

- `targetStack: [Watcher]`

#### 文件中暴露出的方法

- `pushTarget(Watcher)`
  - 将`Watcher`放入`targetStack`中，并作为全局当前计算`Watcher`
- `popTatget`
  - 弹出`targetStack`末端`Watcher`，将当前全局计算的`Watcher`前移一位



## Watcher

#### 构造属性

- `expOrFn`，用于复制`this.getter`，在实例`get`方法中使用。

- `cb`，回调函数，在实例`run`方法中执行。

- `options`，各种属性参数

  ```javascript
  this.deep = !!options.deep
  this.user = !!options.user
  this.lazy = !!options.lazy
  this.sync = !!options.sync
  this.before = options.before
  ```

#### 构造函数

- 如果是渲染`Watcher`，则`vm._watcher = this`
- 属性赋值
- 对`expOrFn`类型判断并赋值给`getter`
- 如果`lazy === false`，调用`get`



#### 实例方法

- `get`
  - 调用`pushTarget`将该`watcher`实例作为全局的计算watcher
  - 触发`getter`，也就是传入的参数`expOrFn`
  - 判断`deep`，进行`traverse`处理
  - `PopTarget`，移除全局Watcher，并使用`cleanupDeps`清除依赖
- `addDep(dep)`
  - 根据传入`dep`的id，将`dep`放入`this.newDeps`中
  - 同时将该`watcher`放入`Dep`的`subs`中
- `cleanupDeps`
  - 将新`dep`和旧`dep`进行处理更新
  - 移除更新后移除的旧`dep`对`Watcher`的订阅（`dep.removeSub`）
  - 交换新旧`dep`，同时清除新`dep`
- `update`
  - 如果`lazy`，更新`dirty`为`true`
  - 如果`sync`，执行`run`
  - 否则`queueWatcher(this)`
- `run`
  - 如果`active`，才执行整个逻辑
  - 触发`get()`，得到最新的值，只有新旧值不一致时才会进行后续逻辑
  - 调用`cb()`，`this.cb.call(this.vm, value, oldValue)`
- `evaluate`
  - 调用`get`并更新`this.value`
  - 置`dirty`为`false`
- `depend`
  - 获取当前`deps`的长度，`deps`内的元素执行`depend`方法
- `teardown`
  - 判断`active=== true`，否则不进行后续逻辑
  - 判断`vm._isBeingDestroyed`（当前组件是否被摧毁），将该`watcher`从`vm._watchers`中移除
  - 调用`dep.removeSub`解除`deps`对当前`watcher`的订阅
  - 置`active`为`false`

通过本文的大致梳理，后续对数据响应式过程更好理解