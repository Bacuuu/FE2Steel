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



