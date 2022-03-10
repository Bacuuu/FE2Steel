### VNode的首次渲染

​		在[render函数那些事](./render函数.md)、[Vue组件创建](./Vue组件创建.md)中我们介绍了VNode是如何生成的，在生成了VNode后就该进行VNode到DOM的转换，在前面我们有提到，在`$mount`调用后，初始化时会去执行`vm._update()`，第一个参数就是我们前面介绍生成的VNode。

```javascript
    let updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
```

​		首次调用时

```javascript
vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
```



​		`vm._update`是调用`__patch__`函数，`patch`函数有不同的定义，分别位于`platforms/web/runtime/index`以及`platforms/weex/runtime/index`。这样做的原因是什么呢。其实在`patch`之前，生成了VNode，VNode作为一种描述文档页面的数据结构，并没有和呈现的视图耦合，意思就是这里的VNode是数据而已，不是视图，我们的`patch`才能将其转换为视图，对于浏览器而言就是DOM。而拥有了这样一个通用性的对页面的描述，自然也就可以不只在浏览器进行页面的渲染，所以这也是VNode的好处之一，跨平台。通过对不同`patch`函数的调用，转换为不同平台的视图。后续我们只针对DOM的转化进行分析，也就是分析`platforms/web/runtime/index`中的`patch`。

```javascript
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

`		patch`函数如下，其中`nodeOps`和`platformModules`是`web`目录下的，所以是和平台有关的，所以在`weex`中，这两个参数也是从`weex`中引入的。这就是不同平台的`patch`有差异的原因，通过对`createPatchFunction`函数传入不同的参数，构造出不同的`patch`函数，这也是柯里化的体现。

```javascript

import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

const modules = platformModules.concat(baseModules)
export const patch: Function = createPatchFunction({ nodeOps, modules })

```

​		在`createPatchFunction`中，最后返回的`function`就是`patch`函数，对于首次渲染主要逻辑如下：

```javascript
  return function patch (oldVnode, vnode, hydrating, removeOnly) {

    let isInitialPatch = false
    const insertedVnodeQueue = []
	// 不存在旧结点的情况
    // 对于首次渲染来说，oldVnode我们传入的是$mount('#app')的 app元素
    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      // 这里是处理$mount()的情况
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      // 判断旧节点是否为DOM节点
      const isRealElement = isDef(oldVnode.nodeType)
      // 非DOM，而且是相同节点。非首次渲染，不做讨论
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          // 根据dom节点的tag 去创建一个VNode
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        // oldElm为旧dom节点
        const oldElm = oldVnode.elm
        // parentElm为旧dom节点的父节点
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        // 这部分初次渲染也不涉及，不做说明
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
	// 调用insert hook
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    // 返回节点
    return vnode.elm
  }
```

#### createElm

```javascript
  function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // This vnode was used in a previous render!
      // now it's used as a new node, overwriting its elm would cause
      // potential patch errors down the road when it's used as an insertion
      // reference node. Instead, we clone the node on-demand before creating
      // associated DOM element for it.
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check
    // 首先以组件类型去处理，这块单独 提取文档 处理
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      // 通过document.createElement()创建正确标签的DOM节点
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      // 处理 scoped css
      // 给上面创建的节点增加属性
      setScope(vnode)

      /* istanbul ignore if */
      if (__WEEX__) {
        // 忽略weex的情况
      } else {
        // 遍历子节点，再将元素作为vnode、当前vnode.elm当作父节点传入createElm进行DOM创建
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          // 执行平台预定义的create的钩子函数
          // 调用vnode的create hook
          // 判断vnode.hook.insert 将vnode插入insetedVnodeQueue中
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        // 将vnode.elm插入到parentElm内部的末尾
        // refElm作用是如果存在，插入到该元素之前
        insert(parentElm, vnode.elm, refElm)
      }
    } else if (isTrue(vnode.isComment)) {
      // 注释节点，创建注释节点，插入
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      // 创建文本节点，插入
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
```

​		该方法目的是创建真实DOM并将其插入指定的位置，所以这才是`patch`的关键。在这个过程中有大量的`nodeOps.**`的方法，这就是根据平台来创建DOM元素的方法。

##### createChildren

```javascript
  function createChildren (vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
      // 遍历children，使用createElm进行创建插入操作
      // 这里是个递归操作，深度优先
      for (let i = 0; i < children.length; ++i) {
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
      }
    } else if (isPrimitive(vnode.text)) {
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
    }
  }
```

##### invokeCreateHooks

```javascript
  function invokeCreateHooks (vnode, insertedVnodeQueue) {
    // 执行预设的create hook
    for (let i = 0; i < cbs.create.length; ++i) {
      cbs.create[i](emptyNode, vnode)
    }
    i = vnode.data.hook // Reuse variable
    // 执行vnode的hook
    if (isDef(i)) {
      if (isDef(i.create)) i.create(emptyNode, vnode)
      if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
    }
  }
```

##### insert

```javascript
  function insert (parent, elm, ref) {
    if (isDef(parent)) {
      if (isDef(ref)) {
        // ref是parent内部元素，待插入节点插入到ref前面
        if (nodeOps.parentNode(ref) === parent) {
          nodeOps.insertBefore(parent, elm, ref)
        }
      } else {
        // 未指定ref，直接插入到父节点内部最后
        nodeOps.appendChild(parent, elm)
      }
    }
  }

// nodeOps.appendChild
export function appendChild (node: Node, child: Node) {
  node.appendChild(child)
}
```

​		简单来说，这里`patch`流程是

- `createElm`，根据`vnode`的`tag`创建节点，再根据`children`去调用`createChildren`函数
- `createChildren`，遍历`children`，对每个节点再使用`createElm`进行创建，传入父节点为上级节点，将新生成的DOM节点插入到父节点下
- 上面的过程就是深度遍历子节点，不断创建DOM节点插入父节点。

