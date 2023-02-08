#### 从打包开始讲起

​		如果我们通过`script`标签使用`Vue`时，我们可以使用以下两种方式引入`Vue`。

```html
<!-- 开发环境版本，包含了有帮助的命令行警告 -->
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
<!-- 生产环境版本，优化了尺寸和速度 -->
<script src="https://cdn.jsdelivr.net/npm/vue@2"></script>
```

​		我们来看下这些代码是怎么生成的，在`Vue`源码的`package.json`中`script`我们可以看到各种命令。对于开发环境，诸多的命令都是指向`scripts/config.js`；对生产环境，命令指向`scripts/build.js`，但是`build.js`也会引入`config.js`中的配置，所以我们来看下这个`build.js`。

​		`scripts/config.js`主要作用之一是，根据打包命令不同参数，确定`rollup`打包入口、出口文件以及其他参数。当我们运行命令`npm run dev`时，执行的内容实际是`rollup`命令，通过命令将环境变量的`TARGET`设置为`web-full-dev`

```json

"scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
}
```

​		在`script/config.js`中，有这样一块定义和逻辑

```javascript

const builds = {
    // ...
    'web-full-dev': {
        entry: resolve('web/entry-runtime-with-compiler.js'),
        dest: resolve('dist/vue.js'),
        format: 'umd',
        env: 'development',
        alias: { he: './entity-decoder' },
        banner
  	},
}
// 根据builds来生成rollup配置
function genConfig (name) {}

// 获取到target，这里是web-full-dev
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
}
```

这里进入`web-full-dev`的配置，如果是`npm run build`通过这个配置打包出来的文件也是`dist/vue.js`，所以我们以该配置继续进行分析，他的入口文件指向`web/entry-runtime-with-compiler.js`。主要内容如下：

```javascript
import Vue from './runtime/index'
Vue.prototype.$mount = function () { /** do something */ }
Vue.compile = compileToFunctions
export default Vue
```

​		可以看出，这里是引入`Vue`，并在原型链挂载`$mount`方法。当然`Vue`的定义还是来自于`./runtime/index`。

```javascript
// .runtime/index
import Vue from 'core/index'
Vue.prototype.__patch__ = inBrowser ? patch : noop
// public mount method
Vue.prototype.$mount = function (){ /** do something */ }
export default Vue
```

​		这里我们还是看不出来`Vue`是什么，在这里也是对`Vue`进行了原型链的增强，挂载了`__patch__`和`$mount`方法。在上面代码中我们也进行了`$mount`方法的挂载，覆盖了此处的`$mount`。从注释可以看到，这里使用的是公共的`mount`方法，在其它版本的`Vue`中，例如`web/entry-runtime.js`就没有进行方法重写，使用默认公共`mount`方法。继续溯源

```javascript
// core/index.js
import Vue from './instance/index'
export default Vue	

// core/instance/index.js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

​		从`core/index`我们直接继续分析到`core/instance/index.js`中，可以看到Vue是一个函数，准确来说是一个构造函数，只能通过`new`进行调用。通过下面的`Mixin`增强构造函数特性，这里面也就涉及到`Vue`的各种特性实现。有关响应式原理的部分在之前的文章中有介绍，后续将一边复习一边介绍关于组件化相关的知识。

​		构造函数中只执行了`this._init`方法，这个方法通过下面的`initMixin`中定义。先不急着查看定义，我们先看看`Vue`这个构造函数。现在我们大部分时间通过脚手架的方式进行项目搭建、业务编写。在早些时间，我还在大学使用`Jquery`写页面时，我们也引入了`Vue`，那时在单页中进行引用，如下。

```html
<div id="new_app">
</div>
<script src="https://cdn.bootcss.com/vue/2.6.10/vue.min.js"></script>
<script>
var newApp= new Vue({
  el: '#new_app',
  // other options
})
</script>
```

​		我们先在DOM中声明一个`id = new_app`的元素，在以选择器的方式以`el`字段传入`Vue`构造函数。在#`new_app`内写模板语法，构造函数中传入更多的参数，这样就能够像脚手架搭建之后一样进行开发。

​		现在我们再看下构造函数中的`_init`方法做了什么。

```javascript
Vue.prototype._init = function (options?: Object) {
  // important but ignore
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

​		这里的`vm.$options.el`是我们传入的`el`，传入的参数已经通过一些处理挂载到了`vm.$options`上。这里回到了我们前面多次看到的`$mount`函数，在[$mount函数在做什么](./$mount函数在做什么.md)中我们继续探索。