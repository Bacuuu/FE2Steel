### 关于组件创建

​		在之前介绍`render`函数时，有对`_createElement`进行说明，其作用主要是创建`VNode`，大部分情况通过`VNode`的构造器进行生成。当标签名是注册过的组件名时，以及标签非`string`类型时，通过`createComponent`进行`VNode`的创建。

​		推荐先阅读[Vue.component]() [Vue.extend]()

#### createComponent的前置条件

 ##### 在`tag`为`string`条件下，符合条件下调用`createComponent`

```javascript
  if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
	// component
	vnode = createComponent(Ctor, data, context, children, tag)
  }
```

​		满足两个条件

- `data`不存在或者`data.pre`为`false`。查看`data`的类型声明中是没有`pre`的，猜测这里是对`v-pre`做处理。
  - `v-pre`作用：跳过这个元素和它的子元素的编译过程。可以用来显示原始 Mustache 标签。跳过大量没有指令的节点会加快编译。
- `isDef(Ctor = resolveAsset(context.$options, 'components', tag))`，这里判断是否存在且进行赋值操作

看下这个`resolveAsset`函数。

```javascript
export function resolveAsset (
  options: Object,
  type: string,
  id: string,
  warnMissing?: boolean
): any {
  /* istanbul ignore if */
  if (typeof id !== 'string') {
    return
  }
  // 获取options中的资源，这里是获取组件，结构是{componentName: component, }
  const assets = options[type]
  // check local registration variations first
  // 判断组件中是否有传入的Tag名的
  if (hasOwn(assets, id)) return assets[id]
  const camelizedId = camelize(id)
  // 将标签名转化为驼峰，再进行判断组件是否存在
  if (hasOwn(assets, camelizedId)) return assets[camelizedId]
  const PascalCaseId = capitalize(camelizedId)
  // 将驼峰再首字母大写，再次尝试判断
  if (hasOwn(assets, PascalCaseId)) return assets[PascalCaseId]
  // 由于hasOwn只能查找自身属性，不查找原型链属性。再从原型链查一次
  // fallback to prototype chain
  const res = assets[id] || assets[camelizedId] || assets[PascalCaseId]
  if (process.env.NODE_ENV !== 'production' && warnMissing && !res) {
    warn(
      'Failed to resolve ' + type.slice(0, -1) + ': ' + id,
      options
    )
  }
  return res
}
```

​		代码具体解读在注释中体现，该函数主要作用在`options`中去某个属性中查找符合条件的值，这里就是查找组件。所以找到相应组件才会进入`createComponent`函数。这里的`Ctor`被赋值为对应的`component`

##### 当`tag`非`string`时也会调用`createComponent`

```javascript
vnode = createComponent(tag, data, context, children)
```

#### createComponent

```javascript
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }
  // 在initGlobalAPI函数中挂载，_base就是Vue
  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  // 对象参数形式，使用extend方法生成构造函数
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  // 这里Ctor一定是传入的构造函数 | 通过extend生成的构造函数 | 异步组件
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  // 如果函数没有cid，说明不是通过extend生成的构造函数，当作异步函数处理
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
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

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  // 目的是更新Ctor参数，
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  // 如果有model参数，见下方
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  // 函数式组件，单独起md
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  // 事件监听器在 `on` 内，
  // 但不再支持如 `v-on:keyup.enter` 这样的修饰器。
  // 需要在处理函数中手动检查 keyCode。
  const listeners = data.on
  
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  // nativeOn
  // 仅用于组件，用于监听原生事件，而不是组件内部使用
  // `vm.$emit` 触发的事件。
  data.on = data.nativeOn

  // 没找到abstract相关参数资料...
  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  // 合并hook
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  // 主要是传入name, data, context, componentOptions, asyncFactory
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```

##### resolveConstructorOptions

```javascript
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  // 通过Vue.extend构造时挂载，Ctor.super 是 Ctor的父类
  if (Ctor.super) {
    // 递归调用，父级可能也是继承来的，也进行参数合并调用
    const superOptions = resolveConstructorOptions(Ctor.super)
    // 将父类参数缓存在当前构造函数
    const cachedSuperOptions = Ctor.superOptions
    // 对比父类参数是否改变
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      // 更新父类参数
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      // 见下方函数，查找修改的options
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        // extendOptions是我们传入的options
        // 赋值更新后的，覆盖
        extend(Ctor.extendOptions, modifiedOptions)
      }
      // 合并父类参数和拓展参数
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      // 如果有name，将自己挂载到components[name]下
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}

function resolveModifiedOptions (Ctor: Class<Component>): ?Object {
  let modified
  const latest = Ctor.options
  // Ctor.saledOptions生成在extend方法中，单层拷贝options，可理解为Ctor.options的副本
  const sealed = Ctor.sealedOptions
  // 遍历对比，modified为修改的键值
  for (const key in latest) {
    if (latest[key] !== sealed[key]) {
      if (!modified) modified = {}
      modified[key] = latest[key]
    }
  }
  return modified
}
```

​		该段代码是对参数进行合并，如果父类参数修改后也进行修改。

##### transformModel

​	新开[文档]()单独说明

##### extractPropsFromVNodeData

```javascript
export function extractPropsFromVNodeData (
  data: VNodeData,
  Ctor: Class<Component>,
  tag?: string
): ?Object {
  // we are only extracting raw values here.
  // validation and default values are handled in the child
  // component itself.
  // 组件预定义的props，预期的传入props
  const propOptions = Ctor.options.props
  if (isUndef(propOptions)) {
    return
  }
  const res = {}
  // 实际传入
  // attrs是 普通的 HTML attribute {id = '123'}这种
  // props是组件参数传入
  const { attrs, props } = data
  if (isDef(attrs) || isDef(props)) {
    for (const key in propOptions) {
      // hyphenate作用 将形如AbCdEfg转换为ab-cd-efg
      const altKey = hyphenate(key)
      // 查询预定义的props中是否在真实传入参数中存在
      // 存在就复制到res，同时attrs如果存在，删除attrs中原本
      checkProp(res, props, key, altKey, true) ||
      checkProp(res, attrs, key, altKey, false)
    }
  }
  return res
}

// 查询key,altKey是否在hash对象中作为键存在
// 有的话，复制到res
// preserve用于控制是否保留原本，false不保留
function checkProp (
  res: Object,
  hash: ?Object,
  key: string,
  altKey: string,
  preserve: boolean
): boolean {
  if (isDef(hash)) {
    if (hasOwn(hash, key)) {
      res[key] = hash[key]
      if (!preserve) {
        delete hash[key]
      }
      return true
    } else if (hasOwn(hash, altKey)) {
      res[key] = hash[altKey]
      if (!preserve) {
        delete hash[altKey]
      }
      return true
    }
  }
  return false
}

```

​		`extractPropsFromVNodeData`目的是将`attrs`和`props`根据预定义的`props`进行合并，`attrs`不会保留原本，这里后续处理应该会用到❓

##### installComponentHooks

```javascript
function installComponentHooks (data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  // hooksToMerge是定义好的hook名字
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    // 根据Hook键拿到预定义的hook函数
    const toMerge = componentVNodeHooks[key]
    // 传入的和预定义的不同 且 未合并
    if (existing !== toMerge && !(existing && existing._merged)) {
      // 更新hooks为合并后的
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

// 合并hook
// 就是对传入的两个函数依次执行，同时标记为 已合并
function mergeHook (f1: any, f2: any): Function {
  const merged = (a, b) => {
    // flow complains about extra args which is why we use any
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}
```

​		这里是对我们传入的`VnodeData`进行生命周期合并操作，如果需要合并就是将预定义的`hook`和新增的`hook`分别执行。❓用户自定义`render`函数时文档中没有提到有关hook传入，这里的hook什么情况下会传入。

#### 总结

​		`createComponent`大致流程如下

- 由于在render函数中，`createComponent`函数首个参数传入为`tag`，而一种可能下`tag`是非字符串类型的，所以我们先对`tag`（这里的`Ctor`）进行类型判断。
  - `Object`，当作参数传入`extend`，生成子类构造函数
  - `function`，通过`cid`继续判断
    - 异步组件，详见[Vue组件-异步组件](./Vue组件-异步函数组件.md)
    - 组件构造函数，后续对该情况处理
- 通过`resolveConstructorOptions`对参数进行合并更新
- `transformModel`，对`model`参数进行处理，[单独说明]()
- 对组件上的参数进行处理，将`attrs`和`props`根据预定义的`props`进行合并
- 函数式组件通过`createFunctionalComponent`处理，[单独说明]()
- 事件监听处理
- 生命周期合并
- 通过`new VNode`生成VNode并返回，应为该函数的目的是`render -> VNode`的转化。这里的`VNode`传入了`componentOptions`、`asyncFactory`参数，后面在`patch`将`VNode`转换为真实DOM时再来分析。
