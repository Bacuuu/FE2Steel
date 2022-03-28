### 视图更新流程

#### 和第一次渲染的区别

​		第一次渲染和后续更新都是调用`_update`函数，具体差异在下面这部分的逻辑

```javascript

// 初始化时，vm._vnode为空，因为vm._vnode的赋值是在下一步进行
const prevVnode = vm._vnode
vm._vnode = vnode
// 这里的preVnode就是上次的VNode
if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
} else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
}
```

​		我们直接分析`__patch__`中的情况，关于patch的主要逻辑在[VNode的首次渲染](.\Vue2.x源码\VNode的首次渲染.md)中有详细介绍，这里主要分析非首次渲染的逻辑

```javascript
const isRealElement = isDef(oldVnode.nodeType)
// 当老节点不是真实dom，避免挂载的情况
// 且新旧节点是相同的
if (!isRealElement && sameVnode(oldVnode, vnode)) {
  // patch existing root node
  patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
} else {
  // 不是相同节点的情况，直接进行节点替换，更多处理
  
  // 初次渲染挂载，不做解释
  if (isRealElement) {}

  // replacing existing element
  // 旧节点的DOM元素
  const oldElm = oldVnode.elm
  // 旧节点的DOM元素的父元素
  const parentElm = nodeOps.parentNode(oldElm)

  // create new node
  // 通过新节点进行DOM创建
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
  // 处理新节点的parent
  if (isDef(vnode.parent)) {
    let ancestor = vnode.parent
    const patchable = isPatchable(vnode)
    // 父节点存在
    while (ancestor) {
      // 对父组件执行destroy HOOK，关于HOOK后续介绍
      for (let i = 0; i < cbs.destroy.length; ++i) {
        cbs.destroy[i](ancestor)
      }
      ancestor.elm = vnode.elm
      if (patchable) {
        // 执行create HOOK
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
      // ❓不清楚继续向上做处理的意义
      ancestor = ancestor.parent
    }
  }

  // destroy old node
  // 摧毁旧节点以及触发相应HOOK
  if (isDef(parentElm)) {
    removeVnodes([oldVnode], 0, 0)
  } else if (isDef(oldVnode.tag)) {
    invokeDestroyHook(oldVnode)
  }
}
invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
return vnode.elm
```

#### 节点不同

​		在上面的`if(sameVnode(oldVnode, vnode))`的`else`分支，主要逻辑是根据新vnode创建DOM，并替换到父节点下。

##### removeVnodes

​		该方法用于移除vnodes

```javascript
  function removeVnodes (vnodes, startIdx, endIdx) {
    // 根据起止节点遍历vnode
    for (; startIdx <= endIdx; ++startIdx) {
      const ch = vnodes[startIdx]
      if (isDef(ch)) {
        // 根据tag区分文本节点
        if (isDef(ch.tag)) {
          removeAndInvokeRemoveHook(ch)
          invokeDestroyHook(ch)
        } else { // Text node
          // 直接从父节点移除
          removeNode(ch.elm)
        }
      }
    }
  }
```

###### removeAndInvokeRemoveHook

​		移除元素，触发remove HOOK，这块熟悉了hooks后再来补充❓

```javascript
  function removeAndInvokeRemoveHook (vnode, rm) {
    if (isDef(rm) || isDef(vnode.data)) {
      let i
      const listeners = cbs.remove.length + 1
      if (isDef(rm)) {
        // we have a recursively passed down rm callback
        // increase the listeners count
        rm.listeners += listeners
      } else {
        // directly removing
        rm = createRmCb(vnode.elm, listeners)
      }
      // recursively invoke hooks on child component root node
      if (isDef(i = vnode.componentInstance) && isDef(i = i._vnode) && isDef(i.data)) {
        removeAndInvokeRemoveHook(i, rm)
      }
      for (i = 0; i < cbs.remove.length; ++i) {
        cbs.remove[i](vnode, rm)
      }
      if (isDef(i = vnode.data.hook) && isDef(i = i.remove)) {
        i(vnode, rm)
      } else {
        rm()
      }
    } else {
      removeNode(vnode.elm)
    }
  }
```



- createRmCb

  ```javascript
  function createRmCb (childElm, listeners) {
    function remove () {
      // 当remove.listeners === 1时，执行removeNode方法
      if (--remove.listeners === 0) {
        // 将该子元素从父元素中移除
        removeNode(childElm)
      }
    }
    remove.listeners = listeners
    return remove
  }
  ```

###### invokeDestroyHook

```javascript
// 触发vnode的destroy HOOK
// 递归对children调用destroy HOOK
function invokeDestroyHook (vnode) {
  let i, j
  const data = vnode.data
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
    for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
  }
  if (isDef(i = vnode.children)) {
    for (j = 0; j < vnode.children.length; ++j) {
      invokeDestroyHook(vnode.children[j])
    }
  }
}
```



#### 节点相同

##### saveVnode如何判断相同节点

```javascript
function sameVnode (a, b) {
  return (
    // 绑定的key是否相同
    a.key === b.key && (
      (
        // tag、是否注释节点、是否都含有data
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        // 异步函数，判断工厂函数是否相同
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

###### sameInputType

```javascript
function sameInputType (a, b) {
  // input标签才做判断
  if (a.tag !== 'input') return true
  // 声明i用于递增层级进行对比
  let i
  // 对比data、data.attrs、data.attrs.type
  const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
  const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
  // 对比是否相同，或者两个type都符合输入类型
  return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}
```

- isTextInputType

​		该方法定义于`web/util/element`，是和平台相关的方法。该函数用于判断是否`type`为`text,number,password,search,email,tel,url`中的一个。

```javascript
export const isTextInputType = makeMap('text,number,password,search,email,tel,url')
```

- makeMap

​		`makeMap`经常出现在代码中，我们来看下他的实现。

```javascript
/**
 * Make a map and return a function for checking if a key
 * is in that map.
 */
// 使用闭包进行参数封装，返回函数用于判断某个值是否在构建函数时传入的参数中
// 第二个参数用于判断是否将返回的函数的入参进行小写处理
export function makeMap (
  str: string,
  expectsLowerCase?: boolean
): (key: string) => true | void {
  // 创建空Object
  const map = Object.create(null)
  // 处理传入的参数，根据,分割为数组
  const list: Array<string> = str.split(',')
	// 置每个传入参数状态为true
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
	// 根据第二个参数，返回是否存在于map的key中
  return expectsLowerCase
    ? val => map[val.toLowerCase()]
    : val => map[val]
}
```

​		所以它的作用是，生成一个某个值是否在构建函数时传入的参数中，可选参数进行大小写处理。

#### patchVnode

```javascript
  function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    // 新旧节点相同，不做处理
    if (oldVnode === vnode) {
      return
    }
		// ownerArray这里为null，不做讨论
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
      
    // 处理 静态节点
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    // 执行prepatch HOOK
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    // 执行update HOOK
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    // 非文本节点
    if (isUndef(vnode.text)) {
      // 如果新旧 都存在子节点
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        // 只有新 存在子节点
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 只有旧 存在子节点
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 都不存在子节点，旧节点为文本节点
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 新节点为文本节点且 新旧文本不同
      nodeOps.setTextContent(elm, vnode.text)
    }
    
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

##### prepatch

​		`prepatch`定义于`core/vdom/reate-component.js`中，顾名思义只有在组件中才会存在`prepatch`钩子，看下实现。

```javascript
  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },
```

​		由于篇幅限制，这里不做过多讨论，目的是更新旧组件的`componentInstance`的属性。

##### update

​		这里也是执行相关的HOOK，但是平时使用中我们常用的是`beforeUpdate`、`updated`，`update`这个生命周期在自定义指令中我们可以看到，[官方文档](https://cn.vuejs.org/v2/guide/custom-directive.html#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0)描述为**所在组件的 VNode 更新时调用**



##### 对比节点，更新DOM

- 新旧都存在子节点

  通过`updateChildren`更新

- 只有新 存在子节点

  如果旧为文本节点，则 将DOM的`textContent`置为空字符串

  >  [`Node`](https://developer.mozilla.org/zh-CN/docs/Web/API/Node) 接口的 `textContent` 属性表示一个节点及其后代的文本内容。

  通过`addVNodes`方法，该方法就是循环调用`createElm`方法，将新节点的`children`更新插入到`vnode.elm`中，

- 只有旧 存在子节点

  通过`removeVnodes`清除节点。这里为什么不直接像上面一样置为空字符串呢，因为对于`children`还需要去触发他们的生命周期，而上面的是文本节点，直接置为空即可。

- 旧 为文本节点

  这里代表新不存在子节点，且不为文本节点。直接将旧节点通过`removeVnodes`清除原本的`children`即可

- 新为文本节点，新旧不同

  新文本替换更新`textContent`，但是这里为什么不用去`removeVnodes`清除原本`children`。

  我的理解是该函数进入的前提是已经通过了`saveVnode`的验证，所以不会出现一个是文本节点，一个不是文本节点的情况。~~到这里发现自己对VNode并不熟悉~~

###### updateChildren

​		这是`vue`的`diff`关键函数

```javascript
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    // 旧节点的children
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    // 新节点的children
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly


    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      // 1
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 2
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 3
        // 新旧 起始节点 是相同节点，继续对比该节点
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        // 将新旧 起始 节点都右移一位
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 4
        // 新旧 末端节点 相同，也继续进行对比
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        // 新旧 末端 节点都左移一位
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 5
        // 旧起始 和 新末端 相同
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        // 将 旧起始节点 插入到旧末端节点后面
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        // 旧起始右移 新末端左移
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 6
        // 旧末端 和 新起始 相同
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        // 将旧末端 插入到 旧起始之前
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        // 旧末端左移 新起始右移
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 7
        // 如果没有建立旧node的key映射，{key: oldIndex}
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        // 获取 新节点 对应的 旧节点的index
        // 如果 新起始节点有key，对比old的key，拿到相同key的旧节点idx
        // 否则 将新起始节点和旧节点 通过samenode 进行对比
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        // 没有值，说明在旧结点中没有找到，为新增元素
        if (isUndef(idxInOld)) { // New element
          // 将新元素插入到 oldStartVnode 旧节点前
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          // 拿到对比出来相同的 旧节点
          vnodeToMove = oldCh[idxInOld]
          // 对于key相同的情况，但是可能不是相同节点 情况做处理
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            // 旧节点置空
            oldCh[idxInOld] = undefined
            // 旧节点 插入到 旧起始节点前
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            // key同，但节点实际不同
          	// 将新元素插入到 oldStartVnode 旧节点前
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        // 新 起始节点右移
        newStartVnode = newCh[++newStartIdx]
      }
    }
    // 旧节点被循环完，说明新节点应该更多，添加新节点 
    if (oldStartIdx > oldEndIdx) {
      // 处理位置，如果有右移元素，则需要插入在未处理的新节点尾部
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // 新节点循环完，说明旧节点更多，删除剩余旧结点
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

有关的DOM方法：

- `nodeOps.insertBefore(parentNode, newNode, referenceNode)`   将`newNode`插入到`referenceNode`前，如果 `referenceNode` 为 `null` 则 `newNode` 将被插入到子节点的末尾
- `nodeOps.nextSibling(node)`  返回其父节点的 [`childNodes`](https://developer.mozilla.org/zh-CN/docs/Web/API/Node/childNodes) 列表中紧跟在其后面的节点

该部分有较多分支，归类下：

- 1 2：旧节点不存在，对应的是通过key对比，相同时将旧vnode置空的情况
- 3 4 5 6：新 起止 节点  和 旧 起止 节点 进行比较，2x2共4种情况
- 7：有key情况

我们的目的是要将新节点进行更新，所以我们的关注点应该在新节点上，将新节点处理完。

- 将新起止和旧起止节点进行比较，如果起止顺序颠倒，以新节点位置为准将旧节点进行调整；
- 根据key或者循环对比，找到和当前新起始节点相同的旧节点
  - 如果有，将旧节点插入到旧起始节点前
  - 如果没有，当作新元素插入到旧起始节点前
- 最后判断是新的还是旧的节点没有处理完
  - 如果是新节点没有处理完，将新节点插入到未处理新节点的尾部
  - 如果是旧节点没处理完，将未处理的旧节点进行删除。

##### postpatch

和指令有关，暂不处理。