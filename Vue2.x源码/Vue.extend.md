### Vue.extend

​		在组件化部分的源码中，`extend`函数是常出现的一个函数，同时它也是官方暴露出来的一个API。我们从[官方文档](https://cn.vuejs.org/v2/api/#Vue-extend)开始。

> ### Vue.extend( options )
>
> - **参数**：
>
>   - `{Object} options`
>
> - **用法**：
>
>   使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象。
>
>   ```javascript
>   // 创建构造器
>   var Profile = Vue.extend({
>     template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>',
>     data: function () {
>       return {
>         firstName: 'Walter',
>         lastName: 'White',
>         alias: 'Heisenberg'
>       }
>     }
>   })
>   // 创建 Profile 实例，并挂载到一个元素上。
>   new Profile().$mount('#mount-point')
>   ```
>
>   ```html
>   <p>Walter White aka Heisenberg</p>
>   ```

​		看完官方实例后，似曾相识。回想一下，这样的传参方式不就是`Vue`构造函数的传参吗。带着这样的疑问我们去查看`extend`方法的源码。位置位于`core/global-api/extend.js`中，通过`initGlobalAPI -> initExtend`进行调用。

```javascript
export function initExtend (Vue: GlobalAPI) {
  /**
   * Each instance constructor, including Vue, has a unique
   * cid. This enables us to create wrapped "child
   * constructors" for prototypal inheritance and cache them.
   */
  // 用于标记Vue，Vue构造函数拥有唯一的cid，递增式保持唯一
  Vue.cid = 0
  let cid = 1

  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    // 获取我们缓存的构造函数
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }
	
    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }
    // 构建子类，使用_init方法，和`core/instance/index.js`中相同
    // 这里的options才是我们平时用传入的
    const Sub = function VueComponent (options) {
      this._init(options)
    }
    // 通过寄生组合式继承
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    // 递增式保持唯一cid
    Sub.cid = cid++
    // 合并options，以传入为主，父类兜底
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    // 将父类挂载为super属性
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    // 这里的initProps和initComputed并非在响应式原理中所提到的函数，这是在该文件中单独定义的函数，后面会提到
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    // 继承父类的部分属性、方法
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    // 缓存构造函数
    cachedCtors[SuperId] = Sub
    return Sub
  }
}
```

​		代码详细作用以注释方式进行说明，整个流程不是很复杂，主要是对构造函数进行继承。对于寄生组合式继承不熟悉的可以参考[JS继承](../Javascript/JS继承.md)。下面对我阅读该源码中产生的问题为导向进行整理学习。

#### Vue.extend和Vue本身有什么区别

​		在一开始我们就提到了传参方式和Vue函数如出一辙，看完源码后我们发现他也是对`Vue`构造函数进行了继承。那父子之间有什么差异，我的理解是**子类是对父类的参数预填充**，我们可以通过`Vue.extend`在我们的业务中进行拓展，当我们在`options`传入`template`时，这就是一个组件构造器，像官方示例的`Profile`，我们可以使用`new Profile`生成进行组件挂载。

​		我曾在地图应用开发过程中使用过`extend`方法，大致场景是这样的：前端地图引擎本身会提供诸多API给到我们开发人员，其中有一个`addPopup`方法，供我们在地图上显示DOM弹窗，参数大致是`addPopup(dom, longitude, latitude)`。但是弹窗本身也会有很多的交互操作，事件绑定等，作为Vue开发工程师（不是，已经对原生写法较为陌生。于是`extend`方法就此派上用场，当时使用的方法是，给弹窗传入`<div id="unique-value"><div>`，通过`extend`方法生成我们需要的组件构造函数，使用时直接通过`$mount`函数挂载到`#unique-value`上。

#### 缓存Ctor是如何生效的

​		代码中能够看到通过`cachedCtors`进行`cid`比对，提前获取结果的代码。如果是重新生成的子类，最终会把`Sub`放到`    cachedCtors[SuperId] = Sub`中。顿时我很纳闷，这个变量不会被释放吗？这也不是闭包函数，该变量是怎么做到缓存作用的呢。其实仔细一点就能知道，`cachedCtors`是对`extendOptions._Ctor`的引用，如果`extendOptions`没有`_Ctor`参数，会生成空对象进行赋值。~~然后就没有思路了，还好以前在语雀中有记录这一部分~~

```javascript
> const options = {}
undefined
> const component = Vue.extend(options)
undefined
> options
{_Ctor: {
  0: ƒ VueComponent(options)
}}
> component.cid
15
> component.options._Ctor === options._Ctor
true
> const component2 = Vue.extend(options)
undefined
> component2.cid
15
```

​		由上面的尝试可以知道，当我们使用`option`进行传递时，`extend`内部会改变传入参数，会将父类构造函数放入`_Ctor`中。下次传入相同`options`时，能够知道该参数在当前父类的`extend`执行过，故能够进行缓存结果的提取返回。**所以缓存机制是对于同一入参对象而言有效，对于内部值相同但不是同一对象是仍然是无效的**

#### initProps和initComputed

```javascript
function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}

```

​		根据调用时的注释，我们可以看到这是将传入参数的`props`和`computed`代理到构造函数原型链上，从而避免生成的每个实例都需要将其单独生成。在`initState.js -> initProps`中我们可以看到这样的代码

```javascript
// static props are already proxied on the component's prototype
// during Vue.extend(). We only need to proxy props defined at
// instantiation here.
if (!(key in vm)) {
  proxy(vm, `_props`, key)
}
```

​		这里和上面相对应，先判断属性是否存在，通过构造函数原型链是能找到的，所以不对每个实例进行代理。对于每个实例而言，data是单独的，而`props`和`computed`是可以共用的。