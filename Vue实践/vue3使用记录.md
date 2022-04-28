#### vue2到vue3的常用功能平替

​		使用`vue3 setup`方式进行开发过程中，记录`Vue2`中常见功能在`Vue3`中如何使用。

​		目的是能够快速使用vue3进行开发，介绍项目中用到的可行的方案方法，当然也有其他可行方案方法，不逐一介绍。

```vue
<template>
</template>
<script lang="ts" setup>
	// 以下代码都包含于script标签中
</script>
```

#### 组件参数传递

##### 子组件如何暴露参数

[官方文档相关介绍](https://staging-cn.vuejs.org/guide/typescript/composition-api.html#typing-component-props)

```typescript
interface Props {
  tagList: Array<string>,
  defaultVal?: Array<string> | string,
  multiple?: boolean,
  filterable?: boolean
}
const props = defineProps<Props>()

```

​		在子组件中，使用`defineProps`，通过类型定义进行子组件`props`的定义。如果我们使用`?:`进行类型声明，那么该值不是必传参数，对比到`Vue2`也就是`{required: false}`，如果没有给该组件传入该参数则会产生警告，不论是否设置默认值。

##### 如何设置默认参数

```typescript
const props = withDefaults(defineProps<Props>(), {
  multiple: true,
  filterable: false,
  defaultVal: () => [],
})
```

​		通过使用`withDefaults`进行包裹，我们可以对`props`进行默认值设定。

##### withDefaults is not defined

我如此使用后`eslint`报错`withDefaults is not defined`。根据相关[文档](https://eslint.vuejs.org/user-guide/#compiler-macros-such-as-defineprops-and-defineemits-generate-no-undef-warnings)，解决方案如下

- 在`.eslintrc.js`中添加如下配置代码

  ```javascript
    env: {
      'vue/setup-compiler-macros': true,
    },
  ```

  

- `package.json`中升级相关依赖

  ```json
  {
    "eslint": "^7.32.0",
  	"eslint-plugin-vue": "^8.0.0",
  }
  
  ```

#### 组件事件定义

```typescript
// event-demo.vue
// 定义
const emits = defineEmits<{(e: 'change', tags: Array<string>): void}>()
// 触发
emits('change', checkedTag)
```

该方法对于定义时定义的`e`,`tags`会警告未使用，不知道是IDE原因还是目前Vue的缺陷。

事件监听方法和`Vue2`相同

```vue
<event-demo @change="dosth"></event-demo>
```



#### 通过ref调用子组件内部方法

**子组件**

```vue
// DemoTest.vue
<template>
</template>
<script lang="ts" setup>
    const changeValue = function () {}
    defineExpose({
      changeValue,
    })
</script>
```

​		使用`defineExpost`将需要暴露的方法、值暴露出来，供父组件调用。

**父组件**

```vue
<template>
	<demo-test ref="demo" />
</template>
<script lang="ts" setup>
    import DemoTest from '@/components/DemoTest.vue'
    const demo = ref<InstanceType<typeof DemoTest> | null>(null)
    demo.value?.changeMenuValue('')
</script>
```

- 首先是在子组件上使用`ref`属性标注名字，在`setup`中使用`ref`声明同名变量。

- 要获取类型定义，使用`InstanceType<typeof DemoTest> | null`
- 通过`.value?.`进行暴露出来的方法、值的使用。

#### 路由监听

`Vue2`

```javascript
watch: {
	'$route.path': function (n, o) {
        // do sth
    }
}
```

在`Vue2`中，对于`watch`的键为字符串的情况，会去在`this`下进行查找，从而达到监听`this.$route.path`的目的

在`Vue3`中没有`this`上下文，对路由监听实现如下

```typescript
const route = useRoute()
watch(() => route.path, (n) => {
    // do sth
})
```

这里直接使用`function`作为键，达到监听`route.path`的目的。不能直接使用`route.path`是因为它是个字符串，`Vue3`的侦听来源类型说明如下

> `watch` 的第一个参数可以是不同形式的“来源”：它可以是一个 ref (包括计算属性)、一个响应式对象、一个 getter 函数、或多个来源组成的数组：

#### 扩充全局属性

在`Vue2`中，我们可以在`Vue`构造函数上挂载一些数据、方法，以供全局使用。

```javascript
// main.js
import axios from 'axios'
Vue.prototype.$axios = axios
```

`Vue3`提供了[app.config.globalProperties](https://staging-cn.vuejs.org/api/application.html#appconfigglobalproperties)对象

```typescript
// main.ts
import axios from 'axios'
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.config.globalProperties.$axios = axios
```

使用全局属性

```vue

<script lang="ts" setup>
import { getCurrentInstance } from 'vue'

// 先获取实例
const app = getCurrentInstance()

// 通过proxy获取到$axios
app.proxy.$axios('/')
</script>
```

上面的使用方法中的`$axios`类型定义是`any`，这显然不友好。官方也提供了[相应方法](https://staging-cn.vuejs.org/guide/typescript/options-api.html#augmenting-global-properties)

**Vue全局属性类型扩展**

- 新建类型定义，这里在`src`下新建`custom.d.ts`文件

```typescript
// /src/custom.d.ts
import axios from 'axios'

declare module 'vue' {
  interface ComponentCustomProperties {
    $axios: typeof axios
  }
}
```

- `tsconfig.json`中包含该`ts`文件

```json
{
	"include": ["src/custom.d.ts"],
}
```

#### vite打包报错

使用`vite`打包时遇到几个问题，在这里记录下

**optional chaining 可选链操作符**

一开始以为是没有相应`polyfill`，还去安装了`@vitejs/plugin-legacy`插件。结果是`node`版本过低，本机`node`版本为`12.20.0`，可选链操作符是`14`开始支持的，将版本更新为`14.18.0`后解决该问题。

现在想来排查思路有问题，`@vitejs/plugin-legacy`是浏览器端的`polyfill`，而当前问题是打包时的语法问题，是`node`服务端本身的问题。

**TS仍然报错，校验不通过**

在本地使用vscode进行开发时并没有ts报错，但是打包时会报错，和使用了上面介绍的**通过ref调用子组件内部方法**有关。

```typescript
const menu = ref<InstanceType<typeof Menu> | null>(null)
// 仍然报错
menu.value?.changeMenuValue()

// 一种解决方法是使用any 代替null
const menu = ref<InstanceType<typeof Menu> | any>(null)
```

初步感觉是和TS版本有关，排查思路如下：

- 首先从`package.json`可以看到`npm run build`实际是执行`vue-tsc --noEmit && vite build`。我先将其更改为`vite build`，打包成功。说明是前面这部分出现了报错信息。`vue-tsc`为基于IDE插件`Volar`的`Vue3`命令行类型检查工具。

> `vue-tsc`，其文档解释为`Vue 3 command line Type-Checking tool base on IDE plugin [Volar](https://github.com/johnsoncodehk/volar).`

- 既然看到了`volar`，而且我们vscode中也确实有这个插件，去看下`vue-tsc`依赖，有个版本为`0.29.8`的`@volar/shared`依赖。再看`volar`插件版本为`0.34.10`。尝试更新`vue-tsc`为最新版本`0.33.9`。
- 打包成功，猜测升级`vue-tsc`版本这一步骤是解决该问题的真正方法。
- 总结：通过`defineExpose`将组件方法暴露，父组件使用后TS报错，升级`vue-tsc`版本后解决。

#### vite打包部署后浏览器报错Unknown variable dynamic import

能够进入部分页面，但是某些页面报错`Error: Unknown variable dynamic import: ../pages/blogs/List.vue`。

很明显是因为动态引入没有生效，而且是对于嵌套路由的子路由页面会报错。这里我的实现方式是通过对`router.config.ts`类树状结构进行树遍历，通过读取本身的`component`字段，转换为动态引入。

```typescript
// 原本
{
    name: 'login',
    component: 'Login/Index.vue'
}
// 转换后
{
    name: 'login',
    component: () => import(`../pages/${component}.vue`)
}
```

既然这样不能实现，只有换种方式进行。在`vite`的[Glob导入](https://cn.vitejs.dev/guide/features.html#glob-import)中有提到从文件系统中导入模块。我们的实现变成了这样

```typescript
// 将../pages下所有vue加载
let componentsModules = import.meta.glob('../pages/**/*.vue')
// 通过文件路径进行匹配
const component = componentsModules[`../pages/${component}.vue`]
```

现在问题已经解决了，但是产生了一个问题，通过这样的导入方法，如果路由中没有用到`componentsModules`下的某文件，打包是否会打出该文件。尝试新建了一个文件然后不进行引入，打包时仍然将该文件输出了。但是`vite`关于`glob`有这样一段文字。

> 匹配到的文件默认是懒加载的，通过动态导入实现，并会在构建时分离为独立的 chunk。

虽然这里打包会构建相关文件，但是在线上使用过程中并不会去引入，因为代码逻辑中没有使用到该模块。

