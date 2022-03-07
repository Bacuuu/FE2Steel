### Vue.component

​		该函数作用是注册或获取全局组件。注册还会自动使用给定的 `id` 设置组件的名称，官方API文档中给出了以下示例

```javascript
// 注册组件，传入一个扩展过的构造器
Vue.component('my-component', Vue.extend({ /* ... */ }))

// 注册组件，传入一个选项对象 (自动调用 Vue.extend)
Vue.component('my-component', { /* ... */ })

// 获取注册的组件 (始终返回构造器)
var MyComponent = Vue.component('my-component')
```

​		关于该函数的定义和`Vue.extend`类似，都是位于`core/global-api/index.js`中，不同的是使用`initAssetRegisters`函数。

```javascript
// shared/constants.js
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

// core/global-api/assets.js
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    // 对Vue.component | Vue.directive | Vue.filter 进行实现
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      // 如果没有第二个参数进行定义，返回已定义，这里对应上面的第三种方式
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
      	// 如果是组件注册，且定义是一个object，使用extend进行构造器生成
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          // 这里的thisoptions._base就是调用ctx
          definition = this.options._base.extend(definition)
        }
      	// 定义指令
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
      	// 进行挂载，将键名增加字母s
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```

​		整体逻辑很简单

- 先确定是否是获取组件，根据传入id获取组件并返回
- 如果传入的第二个参数是`object`，就当作参数传入`extend`，生成构造器作为组件挂载到`this.options.components`下
- 传入参数为非`object`，则默认是构造器，直接进行``this.options.components``挂载。