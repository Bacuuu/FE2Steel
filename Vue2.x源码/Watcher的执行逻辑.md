在派发更新的过程中，会执行`Watcher.update`。对于更新的派发，并不是马上生效的，我们来解析下这个地方的逻辑。

#### update函数

` update`函数内容如下，`this.lazy`是计算属性的逻辑，在这里我们不做讨论。后面的逻辑是根据`this.sync`进行处理，但是我在v2.x的官方文档中并没有`watcher`配置项为`sync`的介绍。这里先不对`sync`的来源做讨论，进行后续逻辑的介绍。

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

#### run函数

执行`this.get()`，对比新旧值并调用回调函数。在渲染式`Watcher`中，这里会调用`updateComponent`进行组件的更新。



#### queueWatcher执行流程

我们主要对`sync === false`时的`queueWatcher(this)`进行分析。`queueWatcher`定义在`scheduler.js`，从命名可以看出是用于调度相关。

- 根据`watcher.id`判断传入的`Watcher`是否已经存在于队列`queue`中。
- 如果队列没有开始执行（根据`flushing`，下文会进行介绍），将`Watcher`放入队列中。如果已经开始执行，则将该`Watcher`插入队列中，具体逻辑如下
  - 从队列的最后一位开始遍历。因为当前队列中的`Watcher`是在执行的，所以插入队列有两种情况：
    - 插入到当前执行的`Watcher`后一位；
    - 待插入`Watcher`的`id`大于当前执行的`watcher`的`id`时，插入到该执行`Watcher`后面。
  - 上面两种情况，谁先满足就将`Watcher`插入队列。
  - 对于第二种情况，我们要先知道，队列在执行前已经根据`Watcher.id`进行由小到大的排序。
  - 所以这里插入的位置用一句话描述就是，在未执行的`Watcher`队列中，将待插入的`Watcher`插入到保持队列`id`递增顺序的位置。
- 如果没有执行`flushSchedulerQueue`，则执行下面的逻辑（根据`waiting`判断）
  - `config.async === false`，这里根据`vue.config.async`判定，但是生产环境是无效的。直接执行`flushSchedulerQueue`，
  - 否则是执行`nextTick(flushSchedulerQueue)`

这里说明下`flushing`和`waiting`

- `flushing`是在`flushSchedulerQueue`真正开始执行后才置为`true`，又它创建的分支代码也适用于控制根据不同情况如何插入队列`queue`
- `waiting`，确保不会重复执行`flushSchedulerQueue`，只有上一个执行结束才会开始下一次的调用。



#### flushSchedulerQueue执行过程

queueWatcher最终是执行了`flushSchedulerQueue`，这里哦我们就来分析下它的逻辑

- 如同我们上面所介绍的，首先将队列`queue`中的`Watcher`按照`id`从小到大排序。
- 然后就是遍历队列，依次执行`Watcher`的`run`。
- 执行`resetSchedulerState`，清空队列、重置相关的标志位。

这里进行`id`的排序的目的是

1. 父组件先于子组件更新，因为在`Watcher`实例化时，`id`是自增的，创建时是由父到子的。
2. 用户自定义`Watcher`优先于渲染`watcher`
3. 当组件在父组件`watcher`执行过程中被`destroy`，不会执行`Watch.update`，则该`watcher`能够被跳过，这也是由父到子执行的原因。



由上面的分析过程可以看到，`Watcher`并非立即执行的，而是进入队列后在某一时刻进行统一的运行。至于何时才运行，这取决于`nextTick`函数，后面我们再来介绍它。

