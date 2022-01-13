#### props的声明形式

```javascript
// 数组
props: ['title', 'likes', 'isPublished', 'commentIds', 'author']

props: {
  // 带有类型指定的对象
  title: String,
  contactsPromise: Promise, // or any other constructor
  // 多个可能的类型
  propB: [String, Number],
  // 必填的字符串
  propC: {
    type: String,
    required: true
  },
  // 带有默认值的数字
  propD: {
    type: Number,
    default: 100
  },
  // 带有默认值的对象
  propE: {
    type: Object,
     // 对象或数组默认值必须从一个工厂函数获取
      default: function () {
        return { message: 'hello' }
      }
  },
  // 自定义验证函数
  propF: {
    validator: function (value) {
      // 这个值必须匹配下列字符串中的一个
      return ['success', 'warning', 'danger'].indexOf(value) !== -1
    }
  }
}
```



#### 源码解析

所有数据有关的初始化都位于`state.js`中，`Props`也不例外。

```javascript
function initProps (vm: Component, propsOptions: Object) {
  // 取出参数
  // propsData 是 传入的参数
  // propsOptions 是 声明的参数
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    // 校验参数
    const value = validateProp(key, propsOptions, propsData, vm)
    // 添加为响应式
    defineReactive(props, key, value)
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    // 添加代理
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

​		这里主要分为三个步骤，参数校验，添加为响应式，添加代理。

##### 参数校验

```javascript
export function validateProp (
  key: string,
  propOptions: Object,
  propsData: Object,
  vm?: Component
): any {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)
  let value = propsData[key]
  // boolean casting
  const booleanIndex = getTypeIndex(Boolean, prop.type)
  if (booleanIndex > -1) {
    if (absent && !hasOwn(prop, 'default')) {
      value = false
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      const stringIndex = getTypeIndex(String, prop.type)
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
      }
    }
  }
  // check default value
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
  }
  if (
    process.env.NODE_ENV !== 'production' &&
    // skip validation for weex recycle-list child component props
    !(__WEEX__ && isObject(value) && ('@binding' in value))
  ) {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}
```

​		这里的`propOptions`是构造时预定义的`prop`结构，`propsData`是真正传入的数据，该段代码是对预定义的`prop`进行遍历校验。这段校验代码大致逻辑如下：

1. 预定义中为或者是包含Boolean类型，根据预定义值(prop.type)确定

   - 如果实际没有传入这个值而且预定义没有`default`，默认为`false`。
   - 如果值为空字符串或者为键本身的连字符形式，对于不包含`string`，或者是`boolean`在预定义的数组中比`string`更靠前，默认为`true`。

   > 空字符串是指 `<demo attr-test></demo>`，这里`attr-test === ''`
   >
   > 连字符形式是指`<demo attr-test="attr-test"></demo>`或`<demo attr-test="attrTest"></demo>`

2. 如果值是`undefined`，也就是说预定义的值没有传入。通过`getPropDefaultValue`去拿预定义中`default`字段的值。当默认为一个函数时，如果定义的`type`不是`Function`，会调用将该函数运行结果作为默认值。

3. 返回经过校验后的`value`值

❓这里有一个疑问，`prop.type`是怎么来的，我们预定义的结构中可能是不含`type`的定义方式。

###### prop.type来源

在`initState`之前，有对`props`参数进行处理，通过调用`mergeOptions`，其中再调用`normalizeProps`进行处理，最终将`props`每个元素处理为含有键为`type`的对象

```javascript
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  }
  options.props = res
}
```



- `props`是数组，也就是本文首先的声明形式的第一种情况，将数组中的元素(key)转为驼峰命名，且`props`更改为`key: { type: null}`的形式
- `props`是对象类型，将该对象的键进行驼峰命名转化。再对值(val)进行处理，如果是一个对象则直接使用，否则处理为`{ type: val }`

##### 添加为响应式

​	通过`defineReactive(props, key, value)`进行响应式处理，非生产环境下会自定义设置`setter`，在对`props`进行设置时进行warning提示。

##### 添加代理

​	将`_props`添加代理，其下属性可以用`vm`直接访问。



#### Props更新

大致流程如下

- 父组件数据更改重新渲染，执行`_update -> patch -> prepatch -> updateChildComponent`，在下面得过程中进行校验和计算新的`Props`值。因为初始化时经过响应式处理，所以对`props[key]`赋值时触发`setter`，更新视图。

  ```javascript
    // updateChildComponent函数，其中vm是子组件
    // update props
    if (propsData && vm.$options.props) {
      toggleObserving(false)
      const props = vm._props
      const propKeys = vm.$options._propKeys || []
      for (let i = 0; i < propKeys.length; i++) {
        const key = propKeys[i]
        const propOptions: any = vm.$options.props // wtf flow?
        props[key] = validateProp(key, propOptions, propsData, vm)
      }
      toggleObserving(true)
      // keep a copy of raw propsData
      vm.$options.propsData = propsData
    }
  ```

本节涉及到组件渲染相关知识，后续进行细节补充。

#### toggleObserving

```javascript
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
```

`shouleObservve`值对`Observer`中是否执行`observe`方法进行控制。

主要有几个地方进行了调用

- `initProps`，在进行`defineReactive`前后，先置为`false`，再置为`true`。目的是在`defineReactive`时，会执行`observe(val)`，如果是对象或数组，这个过程会递归内部执行`observe`添加为响应式，因为`shouldObserve === false`，所以内部不会被添加为响应式。因为`props`是来自父组件的，而父组件的值改变会导致父组件重新渲染，也会导致子组件重新渲染。
- `validateProp`，当没有传入预定义的`prop`时，获取默认的值后，`shouldObserve`置为`true`，执行`observe(val)`，然后置为`false`，将该值进行响应式处理。个人对这里的解释是：对于上面的传入的`props`，由于父组件改变时会进行更新，但是`default`不一样，没有依赖父组件，所以这里需要进行整个数据的响应式处理。因为这个过程是在`initProps`中的，`shouldObserve`可能发生改变，所以需要做`shouldObserve`的处理和还原。
- `updateChildComponent`，这里是对`props`进行更新，所以和`initProps`一样的道理。