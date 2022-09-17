### AMD & CMD

#### AMD

`AMD`是`RequireJS`在推广过程中对模块定义的规范化产出，所以下面的内容也就是基于`RequireJs`的。

##### 使用

```html
<script src="require.js"></script>
<script src="a.js"></script>
```

先加载`requirejs`，然后再能进行模块定义、模块加载。

##### 如何定义模块

```javascript
// 定义模块函数
define(id?, dependencies?, factory)
```

- `id`，非必填。模块标识，默认为js文件名。
- `dependencies`，非必填。依赖的模块，由模块名称组成的数组。
- `factory`，必填。模块初始化时要执行的函数或对象。

```javascript
// a.js
define(function(){
    var name = 'morrain'
    var age = 18
    return {
        name,
        getAge: () => age
    }
})
// b.js
define(['a.js'], function(a){
    var name = 'lilei'
    var age = 15
    console.log(a.name) // 'morrain'
    console.log(a.getAge()) // 18
    return {
        name,
        getAge: () => age
    }
})
```

可以看出，`RequireJs`加载是异步的，可以等到依赖模块加载完成后再执行回调函数。

##### 如何引入

```javascript
// contents of other.js:
// 引入
require(['foo'], function(foo) {
	// todo
});
```

通过`require`语法进行模块的引入，第一个参数是依赖项，第二个参数是加载完成时的回调函数。

`define`和`require`的区别在于，`define`是在定义模块时去使用，`require`是使用者在写代码时候主动去引入的。个人理解是`require`的依赖是个人的项目中的`package.json`，`define`是个人项目中`node_modules`依赖的`package.json`。

想要深入了解`define`的实现，推荐[amd-define](https://github.com/ChenShenhai/amd-define)这个仓库。

#### CMD

CMD 是 Sea.js 在推广过程中对模块定义的规范化产出。Sea.js 是阿里的玉伯写的。它的诞生在 RequireJS 之后，玉伯觉得 AMD 规范是异步的，模块的组织形式不够自然和直观。于是他在追求能像 CommonJS 那样的书写形式。于是就有了 CMD 。

##### 使用

```javascript
// 所有模块都通过 define 来定义
define(function(require, exports, module) {

  // 通过 require 引入依赖
  var $ = require('jquery');
  var Spinning = require('./spinning');

  // 通过 exports 对外提供接口
  exports.doSomething = ...

  // 或者通过 module.exports 提供整个接口
  module.exports = ...

});
```

可以看出内部的语法和`CommonJs`类似，这也是`seajs`的初衷。

具体使用参考`Sea.js`的`api`[文档](https://github.com/seajs/seajs/issues/266)

#### AMD CMD区别

CMD的使用更接近于`CommonJs`，和AMD不同的是代码中不会将依赖先声明出来，可以在使用模块之前通过`require`进行引入，避免执行未使用的依赖，只有通过`require`引入时才会去执行依赖。**AMD推崇依赖前置、提前执行，CMD推崇依赖就近、延迟执行**。

> 参考文章
>
> [前端科普系列-CommonJS：不是前端却革命了前端](https://zhuanlan.zhihu.com/p/113009496)