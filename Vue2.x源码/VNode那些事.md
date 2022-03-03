### VNode那些事

​		以下内容直接整理自[掘金-说说VNode节点(Vue.js实现)](https://juejin.cn/post/6844903494835683342)

​		`VNode`是对DOM的描述，在`Vue`是一个类，其中包含静态属性、构造方法，文件中还导出一些关于`VNode`的方法，这是作者参考[snabbdom](https://github.com/snabbdom/snabbdom)进行的一些修改。所以`VNode`的事其实不算多，我们从它的属性开始看。

​		代码内容定义于`core/vdom/vnode.js`。

- 构造函数

  ```javascript
    constructor (
      tag?: string,
      data?: VNodeData,
      children?: ?Array<VNode>,
      text?: string,
      elm?: Node,
      context?: Component,
      componentOptions?: VNodeComponentOptions,
      asyncFactory?: Function
    ) {
      // 节点标签
      this.tag = tag
      // 节点对象数据
      this.data = data
      // 子节点，VNode数组
      this.children = children
      // 文本节点
      this.text = text
      // 虚拟节点对应的真实dom
      this.elm = elm
      // 节点命名空间
      this.ns = undefined
      // 作用域
      this.context = context
      // 节点标识，用于diff优化
      this.key = data && data.key
      // 组件Options
      this.componentOptions = componentOptions
      // 节点对应组件实例
      this.componentInstance = undefined
      // 父节点
      this.parent = undefined
      // 是否为原生HTML或只是普通文本，innerHTML的时候为true
      this.raw = false
      // 静态节点，代表在更新过程中一直不会变化
      this.isStatic = false
      // 是否作为根节点插入
      this.isRootInsert = true
      // 是否注释节点
      this.isComment = false
      // 是否为克隆节点
      this.isCloned = false
      // 是否有v-once指令
      this.isOnce = false
      // 函数化组件作用域
      this.fnContext = undefined
      this.fnOptions = undefined
      this.fnScopeId = undefined
      this.asyncFactory = asyncFactory
      this.asyncMeta = undefined
      this.isAsyncPlaceholder = false
    }
  ```
再看看关于`VNode`的方法
- `createEmptyVNode`

  ```javascript
  export const createEmptyVNode = (text: string = '') => {
    const node = new VNode()
    node.text = text
    node.isComment = true
    return node
  }
  ```

  创建注释节点，内容可通过参数传入

- `createTextVNode`

  ```javascript
  export function createTextVNode (val: string | number) {
    return new VNode(undefined, undefined, undefined, String(val))
  }
  ```

  创建文本节点，和注释节点差不多，只是`isComment`字段不同

- `cloneVNode`

  ```javascript
  export function cloneVNode (vnode: VNode): VNode {
    const cloned = new VNode(
      vnode.tag,
      vnode.data,
      // #7975
      // clone children array to avoid mutating original in case of cloning
      // a child.
      // 深拷贝
      vnode.children && vnode.children.slice(),
      vnode.text,
      vnode.elm,
      vnode.context,
      vnode.componentOptions,
      vnode.asyncFactory
    )
    cloned.ns = vnode.ns
    cloned.isStatic = vnode.isStatic
    cloned.key = vnode.key
    cloned.isComment = vnode.isComment
    cloned.fnContext = vnode.fnContext
    cloned.fnOptions = vnode.fnOptions
    cloned.fnScopeId = vnode.fnScopeId
    cloned.asyncMeta = vnode.asyncMeta
    cloned.isCloned = true
    return cloned
  }
  ```

  将能够通过构造器传入的属性通过构造器传入，其中对`children`进行深拷贝，再将生成的`VNode`其他属性进行赋值，克隆节点标识`isClone`置为`true`

