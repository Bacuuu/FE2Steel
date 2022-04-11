# FE2Steel

**📝 好记性不如记下来**

学习、记录，逐步构建前端知识图谱

## Vue2.x 源码

### 组件化
[从Vue源码打包讲起](./Vue2.x源码/从Vue源码打包开始讲起.md)

[\$mount函数在做什么](./Vue2.x源码/$mount函数在做什么.md)

[render函数那些事](./Vue2.x源码/render函数.md)

[VNode那些事](./Vue2.x源码/VNode那些事.md)

[Vue.extend](./Vue2.x源码/Vue.extend.md)

[Vue.component](./Vue2.x源码/Vue.component.md)

[Vue组件创建](./Vue2.x源码/Vue组件创建.md)

[Vue组件-异步组件](./Vue2.x源码/Vue组件-异步函数组件.md)

[Patch--VNode的首次渲染](./Vue2.x源码/VNode的首次渲染.md)

[视图更新流程及Diff](./Vue2.x源码/视图更新流程及Diff.md)
### 响应式原理
[响应式原理中 Observer、Watcher、Dep 代码梳理](./Vue2.x源码/响应式原理中Observer、Watcher、Dep代码梳理.md)

[响应式原理中 渲染Watcher过程](./Vue2.x源码/渲染式Watcher工作流程.md)

[Watcher执行逻辑](./Vue2.x源码/Watcher的执行逻辑.md)

[Observer中对Array方法的重写](./Vue2.x源码/Observer中对Array方法的重写.md)

[解析nextTick](./Vue2.x源码/解析nextTick.md)

[computed Watcher工作流程](./Vue2.x源码/computed%20Watcher工作流程.md)

[自定义Watcher工作流程](./Vue2.x源码/自定义Watcher工作流程.md)

[props源码解析](./Vue2.x源码/props源码解析.md)

## Vue实践
[vue2到vue3的常用功能平替](./Vue实践/vue2到vue3的常用功能平替.md)

## HTTP
[HTTP缓存](./HTTP/HTTP缓存.md)

## Javascript
[JS运行机制-阅读整理](./Javascript/JS运行机制-阅读整理.md)

[JS中部分常用函数的实现](./Javascript/JS中部分常用函数的实现.md)

[JS继承](./Javascript/JS继承.md)
## 一些问题记录
[一次难忘的useState使用](./问题记录/一次难忘的useState使用.md)



## 关于本文档书写的思考

- 对于源码性质的文档，由于较大项目源码较为分散，而一篇文档的目的常常是对一个问题、一种实现或者一个函数的探究，但是在源码阅读过程中常常会出现一些小且不影响当前问题探讨流程的问题。于我而言，很容易进入到这些分支中去，我认为大部分技术工作者都容易进入这样一个状态。一个问题的探究或者说一篇文档的书写其实就像是完成一个工程一样，应该分轻重缓急，项目中有产品经理、项目经理来进行进度的把控。而我们以学习的目的编写文档时，缺失这样一种意识，常常会失去本来的目的，一天下来发现看了很多内容，却感觉又没有什么产出，甚至连原本想解决的问题也没有解决甚至忘了问题所在。对于这样的问题，采用以下流程尝试进行解决。
  - 文档编写时，只对当前问题强相关流程进行分析，“强相关”定义针对不同问题和个人知识程度来说是不同的。对于过程中的其他问题分为两种
    - 该问题虽然和当前流程关系较弱，但是在整个源码阅读中重要，拆分为另一篇文档，同时在该文档中标注，ISSUE中进行代办。ISSUE格式：TODO[当前文档名]:新文档标题或大致内容；标签：`TODO`；内容：原文档问题位置，新文档大致内容。
    - 该问题是较小但需要解决的问题，直接在ISSUE中进行记录，解决后归档到本文件中，该问题（主观性地）分为主动解决，在后续继续过程中会进行解决。ISSUE格式：QUESTION[当前文档名]:问题描述；标签：`small but important`、主动解决：`besolved By U`，后续过程中解决`besolved By Time`