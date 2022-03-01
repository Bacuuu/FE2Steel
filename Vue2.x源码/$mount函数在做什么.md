#### $mount

​		本文主要是对`$mount`方法进行分析，下面以`entry-runtime-with-compiler`这个版本来对这个方法进行分析。

​		下面是`entry-runtime-with-compiler`版本对`$mount`方法重写的部分。

```javascript
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 通过选择器选择挂载元素
  el = el && query(el)

  // 限制挂载的dom元素
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  // 检查是否有render函数，这里为了确保this.options上有render函数
  if (!options.render) {
    let template = options.template
    // 这里的template是我们传入的模板
    if (template) {
      if (typeof template === 'string') {
        // 如果值以 # 开始，则它将被用作选择符，并使用匹配元素的 innerHTML 作为模板。
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        // 处理 传入DOM的情况
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      // 实在没有template，根据el进行获取
      // outerHTML 内容包含描述元素及其后代的序列化HTML片段
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }
	  // 通过compileToFunctions方法根据template生成render方法
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      // 将render挂到$options上
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  // 调用缓存的公用mount方法
  return mount.call(this, el, hydrating)
}
```

​		主要的逻辑在代码中进行了注释，该版本的代码最终也是调用公共的`$mount`方法，进行重写的目的是确保`$options`上有`render`方法，处理逻辑根据不同的`el`和`template`进行`render`函数的生成。其实这里也解释了`compiler`版本的含义，`entry-runtime-with-compiler`和`entry-runtime`区别在于这段`$mount`的重写，官方对两者差异解释在[这里](https://cn.vuejs.org/v2/guide/installation.html#%E8%BF%90%E8%A1%8C%E6%97%B6-%E7%BC%96%E8%AF%91%E5%99%A8-vs-%E5%8F%AA%E5%8C%85%E5%90%AB%E8%BF%90%E8%A1%8C%E6%97%B6)。在使用脚手架时我们实际是使用的`runtime`版本，那我们也没有人为传入`render`函数，也只是写了模板，填入`Vue`的选项而已。其实这一步脚手架或者`webpack`帮我们做了，当使用 `vue-loader` 或 `vueify` 的时候，`*.vue` 文件内部的模板会在构建时预编译成 JavaScript。

​		下面我们来看下运行时和编译器+运行时两个版本都在使用的原味`$mount`

```javascript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

​		实际是返回`mountComponent`函数调用的返回值，以下是`countComponent`的精简代码。

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // 如果还是没有render函数，以创建空VNode函数作为render函数
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
  }
  callHook(vm, 'beforeMount')
  updateComponent = () => {
  	vm._update(vm._render(), hydrating)
  }
  new Watcher(vm, updateComponent, noop, {
    // 组件更新情况下，调用 beforeUpdate HOOK
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  if (vm.$vnode == null) {
      vm._isMounted = true
      callHook(vm, 'mounted')
  }
  return vm
}
```

​		主要步骤是生成渲染式`Watcher`，关于渲染式`Watcher`在[渲染式Watcher工作流程](./渲染式Watcher工作流程.md)中有简单介绍。我们要知道初始化时这里会去执行`vm._update`。

```javascript
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    // 重点在这里，根据prevVnode是否存在决定是初始化还是更新
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```

​		这里主要是执行`__patch__`函数来进行`VNode`到真正`DOM`节点的转换。再啰嗦两句，对于`_update`我们的调用方式是`vm._update(vm._render(), hydrating)`，`vm._update`的接收参数第一个是`VNode`，由此我们可以推断出来`vm.render()`返回的是`VNode`。实际上也确实是这样，在前面分析`compiler`版本时，我们看到 `render`函数的生成来自于`el`和`template`。所以这里可以得出一个流程是

> template模板 --> render函数 --> VNode --> 真正DOM

​		`render`是编译`template`到`VNode`的关键，`__patch__`是实现最后`VNode`到真实`DOM`转换的关键，这两个函数是我们后面分析的重点。

#### 总结

​		`$mount`的目的就是将我们编写的模板变为真实DOM，并挂载到指定的元素上去，其中经历了上述的转换过程。我们先来看看从模板到VNode的`render`函数是怎样的。

[_render函数]()

[__patch__函数]()

