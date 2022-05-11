#### 明确需求

- 支持点击上传和拖拽上传功能
- 可自定义请求头及data数据
- 点击上传时，支持多选功能
- 可选是否自动上传到服务器
- 支持插槽，可自定义上传样式

#### 组件Props

```typescript
interface Props {
  // 上传地址
  url: string,
  // 是否自动上传
  autoUpload?: boolean,
  // 是否多选
  multiple?: boolean,
  // 请求头
  headers?: Object,
  // 额外字段
  data?: Object,
}
```

#### 组件暴露的方法和事件

- 事件

```typescript
interface Emits {
  // 拖拽进入
  (e:'dropIn', event: DragEvent): void,
  // 拖拽离开
  (e:'dropOut', event: DragEvent): void,
  // 上传的文件改变
  (e:'fileChange', files:Array<File>): void,
}
```

- 方法

  `handleUploadFiles`，调用组件方法上传文件，用于非自动上传时父组件主动进行上传。

#### 实现

​		由于重点是`script`中的逻辑，所以先将完整的`<template>`放出来，样式使用的是`Tailwindcss`，有坑的地方也会进行说明。

```vue
<template>
  <div
    class="cursor-pointer"
    @dragenter="handleDragEnter"
    @dragleave="handleDragLeave"
    @drop.prevent="handleDragDrop"
    @dragover.prevent
    @click="handleUpload"
  >
    <input
      ref="fileUploader"
      :multiple="props.multiple"
      class="hidden"
      type="file"
      @change="fileChange"
    >
    <div
      class="w-min h-min pointer-events-none"
    >
      <slot>
        <div
          class="h-20 w-20 flex justify-center items-center rounded border-2 border-dashed border-gray-200"
          :class="uploaderState === 'DROPIN' ? 'shadow-inner shadow-gray-200 border-blue-200' : ''"
        >
          <i class="material-icons text-blue-300">add</i>
        </div>
      </slot>
    </div>
  </div>
</template>
```

##### 入参及事件声明

```typescript
interface Props {
  // 上传地址
  url: string,
  // 是否自动上传
  autoUpload?: boolean,
  // 是否多选
  multiple?: boolean,
  // 请求头
  headers?: Object,
  // 额外字段
  data?: Object,
}

const props = withDefaults(defineProps<Props>(), {
  autoUpload: false,
  multiple: false,
  headers: () => Object,
  data: () => Object,
})

interface Emits {
  (e:'dropIn', event: DragEvent): void,
  (e:'dropOut', event: DragEvent): void,
  (e:'fileChange', files:Array<File>): void,
}
const emits = defineEmits<Emits>()
```



##### 点击操作

​		首先我们来完成最基本的功能，类似于点击`<input type="file">`，这里我们可以直接使用`input`吗？对于该功能来说是可以的，但是我们后续会增加拖拽上传以及可通过插槽自定义上传按钮的功能。所以我们使用一个隐藏的`<input type="file">`，通过触发它的`click`事件来完成点击后的文件选择功能。

```typescript
const fileUploader = ref<HTMLInputElement>()
const handleUpload = function () {
  fileUploader.value?.click()
}
```

选择完文件后通过`files`获取到上传的文件信息，这里可以参考[mdn相关资料](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/Input/file)。将选择的文件放入`cacheFiles`，这里的`multiple`已经通过`input`去做了限制，所以直接将拿到的`FileList`整个放入即可。最终需要将`input`的`value`置空。

```typescript
const cacheFiles:Array<File> = reactive([])
// 点击上传方式
const fileChange = function () {
  if (!fileUploader.value || !fileUploader.value.files) return
  cacheFiles.push(...Array.from(fileUploader.value.files))
  fileUploader.value.value = ''
}
```

##### 上传文件

​		对于点击事件来说我们需要拿到文件信息，然后自己进行上传，而拖拽上传也需要这一步，先将封装这个上传函数。这里主要是将文件添加到表单中去，在这里也预留`header`和`data`入参，完成自定义请求头及`data`数据的需求。

```typescript
// 定义上传函数
const axiosUpload = function (url:string, file: string | Blob, headers?: any, data ?: Object) {
  const form = new FormData()
  form.append('file', file)
  // 处理传入的额外参数
  if (data) {
    const kv = Object.entries(data)
    kv.forEach(i => {
      form.append(i[0], i[1])
    })
  }
  axios.post(url, form, {
    headers,
  })
}
```

##### 拖拽上传

拖拽上传主要是依赖DOM的`DragEvent`接口，本来以为`drag`只适用于DOM元素的拖拽。拖拽事件参考[mdn DragEvent](https://developer.mozilla.org/zh-CN/docs/Web/API/DragEvent)

首先是拖拽进入和拖拽离开事件的实现，这里为了避免事件触发过于频繁，进行了节流操作，同时将`uploaderState`更新，该值作用是对拖拽进入、离开后，组件的默认样式进行改变，同时也将事件抛出。

这里的`uploaderState`仅仅是作为更改组件样式的状态变量。

在`<slot>`外部，我套了一层`div`，并且添加了` pointer-events: none;`的样式属性。如果不添加该属性会导致拖拽进入后，再进入`slot`区域，会触发`dragleave`，再拖拽出`slot`，会触发`dragenter`后立刻触发`dragleave`。

关于` pointer-events: none`解释如下：

> 元素永远不会成为鼠标事件的target。但是，当其后代元素的`pointer-events`属性指定其他值时，鼠标事件可以指向后代元素，在这种情况下，鼠标事件将在捕获或冒泡阶段触发父元素的事件侦听器。



```typescript

const uploaderState = ref<'DROPIN' | 'DROPOUT'>('DROPOUT')
const handleDragEnter = throttle((e: DragEvent) => {
  uploaderState.value = 'DROPIN'
  emits('dropIn', e)
}, 300)

const handleDragLeave = throttle((e: DragEvent) => {
  uploaderState.value = 'DROPOUT'
  emits('dropOut', e)
}, 300)
```

拖拽后释放鼠标，使浏览器获得文件信息是通过`@drop`来实现的，使用`@drop.perevent`的目的是阻止默认事件。当我们将一个文件拖入浏览器后释放，会对文件进行下载，这里的默认事件就是指这个下载操作。

文件信息通过拖拽事件里的`dataTransfer`获取，这里的`file`对象是一个`FileList`类型，并非`Array`，也不是一个具有`Iterator`接口的对象，所以不能直接使用`...`进行解构，只能先使用`Array.from`转换，上面的点击事件同理。

当该组件支持多选上传（multiple）时，将整个`fileList`存起来，否则只存入`files.item(0)`，这里的`item`方法`fileList`的方法，根据给定索引获取`file`对象。

```typescript
// 拖动上传方式
const handleDragDrop = function (e: DragEvent) {
  let files = e.dataTransfer?.files
  if (!files) return
  if (props.multiple) {
    cacheFiles.push(...Array.from(files))
  } else {
    const file = files.item(0)
    file && cacheFiles.push(file)
  }
}
```

##### 触发上传操作

无论是点击上传还是拖拽上传，都是将文件存入`cacheFiles`中，首先封装一个将`cacheFiles`中的文件进行上传的函数，该函数也是暴露给父组件的方法。

`handleUploadFiles`中，每次取`cacheFiles`的第一个文件进行上传，直到没有文件。但如果在上传过程中继续向浏览器上传文件，会一并归入到上次触发的上传操作中进行上传服务器操作。

```typescript
const handleUploadFiles = function () {
  let file = cacheFiles.shift()
  // 如果在网络上传过程中仍然在向浏览器上传文件，则会一并上传
  while (file) {
    axiosUpload(props.url, file, null, props.data)
    file = cacheFiles.shift()
  }
}
```

自动上传（autoUpload）功能，只需要监听`cacheFiles`，调用`handleUploadFiles`即可

```typescript
watch(cacheFiles, (n) => {
  emits('fileChange', n)
  if (props.autoUpload) {
    handleUploadFiles()
  }
})
```







