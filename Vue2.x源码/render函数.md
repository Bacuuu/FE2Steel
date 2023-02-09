### _render

​		在`Vue`构造函数声明后，使用了`Mixin`对构造函数进行了增强，其中有一个`renderMixin`，`_render`函数的定义就在其中。

```javascript
  // 简化部分逻辑后的代码
  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    vnode = render.call(vm._renderProxy, vm.$createElement)
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
```

​		上面的代码主要作用是`vnode`的产生，正常情况`vnode`来自于`render.call(vm._renderProxy, vm.$createElement)`，其中`render`函数就是我们传入构造函数的参数或者是通过`template`生成，这点我们在 [\$mount函数在做什么](./$mount函数在做什么.md) 中有介绍。那接下来的重点就是这个`vm.$createElement`函数。

​		在`_init`函数中执行了`initRender(vm)`，其中涉及`vm.$createElement`的实现。

```javascript
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

​		这里我们可以看到`$createElement`实际上是调用了`createElement`，不过上面有一个类似方法`_c`。通过注释可以知道

- `_c`是用于模板编译的`render`函数调用

- `$createElement`用于用户自定义`render`函数。

  到了这里自然想去了解两者具体的使用场景，下面我就大概说一下我的理解。

#### 关于 `_c` 和 `$createElement`

​		我们首先来看下如果我们想要自定义`render`是什么写法，根据[官方函数说明](https://cn.vuejs.org/v2/api/#render)以及[官方例子](https://cn.vuejs.org/v2/guide/render-function.html)，可以看出提供给用户的`render`函数，第一个参数是`createElement`函数。我们可以直接使用`createElement`函数进行创建我们想要呈现的视图。再根据`render.call(vm._renderProxy, vm.$createElement)`，我们可以知道暴露给用户的`render`函数第一个参数，实际上就是`vm.$createElement`。

```javascript
Vue.component('anchored-heading', {
  render: function (createElement) {
    return createElement(
      'h' + this.level,   // 标签名称
      this.$slots.default // 子节点数组
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

​		如果我们自定义`render`，但是仍然使用的`compiler`版本，并不会进入模板编译，而是使用公共`render`方法，这和使用`runtime`版本是一致的。目前我们知道了，如果自定义`render`，是使用的`$createElement`方法，那`_c`方法在哪里体现呢。

​		`render`职责是生成`VNode`，当我们传入模板后，将模板编译为`render`函数，这一步实际上就已经决定了`render`函数本身是什么样了。在介绍`$mount`时，我们可以知道`render`函数是通过`compileToFunctions`函数生成的。这里我们不去深究这个函数，我们只需要知道在这其中经过了两个过程：

1. 将模板解析为AST
2. 将AST生成渲染函数

AST叫抽象语法树，是源代码的抽象语法结构的树状表示，就是通过另一种更容易解析的格式来表示我们传入的模板。写到这里我突然产生两个疑问

- 为什么需要`render`函数，它本身就是模板引擎的体现，是不会变化的，为什么不直接是一个`Object`或者其他不变量
- AST其实和VNode是差不多的东西，都是一种对结构的“描述”，为什么需要AST。

产生上面的问题是因为我忽略了更新这个过程，结合到会有更新这个过程，有了如下的思考

- 在更新时，调用的是`vm._update(vm._render(), hydrating)`，每次都需要生成新的`VNode`，所以对于第一个问题，`VNode`不可能是不变的，需要根据其他的`options`，例如`data`等进行重新生成，这就是`render`是函数的意义，会重新生成`VNode`。AST在编译阶段出现，它的目的是生成`render`函数，对于同一个`template`来说AST是不变的，AST用来对模板的属性，指令等进行描述，最终生成的`render`函数本身其实也是不变的，但是函数运行的输出是会改变的。
- 关于第二个问题的思考仍然存在，`templpte(不变) -> AST(不变) -> render函数本身(不变) -> render函数运行结果(变)`，前面的两个转换都是不变的，理论来说省略AST是可行的，但是中间添加AST肯定是出于一定考虑的。❓

​		回到刚才的`compileToFunctions`函数本身，通过查看函数定义，可以回溯看到其中调用`compoler/codegen/index`的`generate`函数，其中就能看到`_c`的调用，不过`_c`是位于字符串中，最后会通过`new Function()`的方式进行调用。
```javascript
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  // fix #11483, Root level <script> tags should not be rendered.
  const code = ast ? (ast.tag === 'script' ? 'null' : genElement(ast, state)) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```

#### $createElement

​		以下代码结合源码根据官方[`createElement`参数](https://cn.vuejs.org/v2/guide/render-function.html#createElement-%E5%8F%82%E6%95%B0)进行注释

```javascript
export function createElement (
  context: Component,
  // {String | Object | Function}
  // 一个 HTML 标签名、组件选项对象，或者
  // resolve 了上述任何一种的一个 async 函数。必填项。
  tag: any,
  // {Object}
  // 一个与模板中 attribute 对应的数据对象。可选。
  data: any,
  // {String | Array}
  // 子级虚拟节点 (VNodes)，由 `createElement()` 构建而成，
  // 也可以使用字符串来生成“文本虚拟节点”。可选。
  children: any,
  // 子节点规范的类型
  normalizationType: any,
  // 子节点规范化方式，自定义render为true
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  // 由于data是可选参数，这里是做参数适配。
  // 是数组或基本类型，说明是第二个参数是children，参数集体移动为对应的值
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  // 是用户自定义render
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```

​		`createElement`函数是对`_createElement`函数调用前的参数预处理，针对`data`参数可选的方式进行参数适配

```javascript
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // 被监听的数据，响应式数据，返回注释节点
  if (isDef(data) && isDef((data: any).__ob__)) {
    return createEmptyVNode()
  }
  // 针对<tr is="my-row"></tr>情况，更改标签名
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  // 没有tag，返回注释节点
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // 不太明白用意，插槽相关逻辑
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  // 分情况 子节点规范化，专门提出来说明1
  // 用户自定义render
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  // 创建VNode，专门提出来说明2
  let vnode, ns
  // 根据标签判断
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    // 是html保留标签
    if (config.isReservedTag(tag)) {
      vnode = new VNode(
        // parsePlatformTagName根据平台对tag做处理，正常来说是返回传入值，但是weex下会有处理
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
        // pre应该是跳过编译的v-pre
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component，校验有没有相关组件名和这个tag相同的
      // 是注册组件的情况，直接使用createComponent
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      // 未知tag，直接生成
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // 直接当作组件进行创建处理
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  // 对生成的VNode进行校验
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    // 命名空间相关，暂不清楚 ❓
    if (isDef(ns)) applyNS(vnode, ns)
    // 对style和class做处理❓
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    // 未定义VNode，创建注释节点返回
    return createEmptyVNode()
  }
}
```

##### 子节点规范化

```javascript
// 适用于非自定义render，因为函数组件可能包含数组
// children中元素还有数组，直接将children内部数组释放出来平级，将children打平
export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}

// 适用于用户自定义render
// 如果是基本类型，创建为文本节点，用户可能只给children传递一个基本类型
// 如果是数组，使用normalizeArrayChildren进行规范化
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```

​		下面再看下`normalizeArrayChildren`方法

```javascript
function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    // 子元素也是Array，递归调用
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
      // 基本类型，
    } else if (isPrimitive(c)) {
      // 当前储存的结果，末尾是文本节点，和当前进行合并
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // 无法合并，且不为空，直接创建文本节点
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      // 当前已经是VNode
      // 当前是文本VNode，且可合并
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // 根据_isVList，更新key
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

​		该函数主要功能就是生成`VNode`，合并能合并的`TextVNode`，整体流程如下

- 遍历数组元素，根据元素类型判断
  - 不为空的数组，递归进入当前规范流程
  - 是基本类型，转化为`TextVNode`，合并`TextVNode`
  - 是VNode
    - 根据情况进行合并`TextVNode`
    - children是`Vlist`（根据`_isVList`判断），和`v-for`相关，根据条件更新key❓

​		合并`VNode`的逻辑是，使用`last`和`lastIndex`记录当前已经规范化结果的最后一个点，如果该点为`TextVNode`且当前待处理的点也为`TextVNode`，可以直接进行合并。如果不合并，会直接将当前待处理的点也放入`res`最后，导致有两个连续的`TextVNode`。

##### 创建VNode

​		详细流程在代码中进行注释体现，主要目的就是根据`tag`进行不同逻辑的处理，最终生成`VNode`并返回。生成`VNode`的方法我们直接进行`new VNode`，后面我们介绍下`VNode`构造函数。

#### 总结

​		以上主要是对用户编写的`render`函数进行处理，从`render`函数到`$createElement`调用，在我们编写的`render`函数中，我们实际是调用`$createElement`，传入预定结构，生成`VNode`。

​		对于模板编译的过程是如何调用`_c`进行处理生成`VNode`的，后续有空再进行学习 ~~学不完的~~。