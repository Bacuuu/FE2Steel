
前端代码运行环境是在浏览器中，所有变量默认是全局`window`下的。`javascript`代码通过`<script>`标签引入，而各个标签的运行环境是非独立的，所以各个模块中声明的变量作用域是全局的，模块之间可能造成冲突。上面的文章中还提到`NodeJs`和`CommonJs`的关系，这也是促进前端模块化的因素之一。

### CommonJs

`CommonJs`是一种规范，并非一种语法，也就是说它是一种通过`Javascript`语法的实现。

#### 使用

首先我们默认当前环境是在`NodeJs`中，因为在该环境下可以直接使用`CommonJs`规范。使用上主要设计到三个变量：`module`、`exports`、`require`

- `module`，当前模块，可以理解为当前`js`文件
- `exports`，`module`下的一个属性，等同于`module.exports`，用于向模块外导出
- `require`，用于加载别的模块

##### 如何导出

```javascript
// a.js
module.exports = something // 需要导出的东西
```

##### 如何引入

```javascript
// b.js
const a = require('./a.js')
```

使用方式很简单，再通过几个例子来深入了解下

##### 通过`exports`直接导出

```javascript
// a.js
exports.name = 123

// b.js
const a = require('./a.js')
console.log(a)

// 输出
// { name: 123 }
```

由此可以看出我们可以通过上面提到的`exports`代替`module.exports`直接进行模块的导出。再看一个例子。

```javascript
// a.js
exports = {
    name: 123
}

// b.js
const a = require('./a.js')
console.log(a)

// 输出
// {}
```

这次的输出和上面的输出不同，但是`a.js`的`exports`是相同的。原因在于**导出的实际是module.exports，而这里的exports通过赋值后不再是对module.exports的引用**，所以这里的输出为空对象。

#### 深入理解

##### require是进行拷贝

```javascript
// a.js
var age = 18
exports.age = age
exports.setAge = function(a){
    age = a
}
setTimeout(() => {
  console.log(age)
}, 2000)

```

```javascript
// index.js
var a = require('./a.js')
console.log(a.age)
a.setAge(19)
console.log(a.age)

// 输出
> 18
> 18
> 19
```

从这个例子可以看出，两次的输出结果都是一样的。中间我们调用了一次`setAge(19)`方法。对于`index.js`中的变量`a`，`a.age`值并没有进行改变，都是18。但是在`a.js`中通过定时2秒后输出`age`为19，这个就是通过`index.js`中进行改变的。由此可以推出，在`a.setAge(19)`时，原模块的`a.age`已经改变，但是通过`require`进行加载的变量`a`并没有进行改变。因为**`require` 的是被导出的值的拷贝**

##### require具有缓存

```javascript
// e.js
let name = "bacuuu"
exports.name = name
```

```javascript
// index.js
const e = require('./e')
console.log(e.name)
e.name = 're'
console.log(e.name)

// delete(require.cache['c:\\Users\\fangjialong\\Desktop\\e.js'])
// console.log(Object.keys(require.cache))

const e2 = require('./e')
console.log(e2.name)

// 输出
> bacuuu
> re 
> re
```

从上面我们知道，这里的变量`e`是值的拷贝，所以对于前两个输出都是可以理解的。后面再次通过`require`引入`e.js`，如果是重新引入，那`e2.name`应该是`bacuuu`。如果将中间注释的代码放出来，则最后会打印值`bacuuu`。

##### require引入 引用类型变量

```javascript
// a.js
var age = [18]
exports.age = age
exports.pushAge = function(){
    age.push(123)
}
setTimeout(() => {
  console.log('in A', age)
}, 2000)
```

```javascript
// index.js
var a = require('./a.js')
console.log(a.age)
a.pushAge()
console.log(a.age)

```

```javascript
// 输出
> (1) [18]
> (2) [18, 123]
> in A (2) [18, 123]
```

和前一个例子不同，这次在`a.js`中导出的值为数组，是引用类型，非基本类型。从结果可以看出，**虽然`require` 的是被导出的值的拷贝，但是对于引用类型使用的还是相同值**，所以`a.js`中延时打印后会是改变后的值。这里也就是可以理解为`require`是一个“浅拷贝”。

#### commonjs实现

```javascript
// bundle.js
// 一个立即执行函数，加载所有的模块
// 入参是所有模块
// 返回值是对入口js的引用
(function (modules) {
  // 模块管理的实现，储存已经加载过的module
  var installedModules = {}
  /**
   * 定义require的实现，加载模块的业务逻辑实现
   * @param {String} moduleName 要加载的模块名
   */
  var require = function (moduleName) {
    // 如果已经加载过，就直接返回
    if (installedModules[moduleName]) return installedModules[moduleName].exports
    // 如果没有加载，就生成一个 module，并放到 installedModules
    // 挂载exports到module
    var module = installedModules[moduleName] = {
      moduleName: moduleName,
      exports: {}
    }
    // 执行要加载的模块
    // exports是当前模块导出的内容
    modules[moduleName].call(module.exports, module, module.exports, require)
    return module.exports
  }
  // 调用入口文件
  return require('index.js')
})({
  'a.js': function (module, exports, require) {
    // a.js 文件内容
  },
  'b.js': function (module, exports, require) {
    // b.js 文件内容
  },
  'index.js': function (module, exports, require) {
    // index.js 文件内容
  }
})
```

使用立即执行函数加载各个模块，各个模块的生成是由构建工具生成的（其实整个文件都是构建工具生成的）。

### 总结

- `commonjs`主要内容是`module`、`exports`、`require`三个变量，其中`exports = module.exports`，避免代码中对`exports`进行直接赋值，会移除对`module.exports`的指向，失去本身意义。
- `require`有缓存机制，多次引入同一模块不会反复执行模块，会直接使用之前的导出内容，除非删除`require.cache`中对应模块的缓存。
- `require`是对`module`导出值的浅拷贝，对于基本类型，不会影响到`module`本身的值；对于引用类型，会对原本值进行改变。（这里的`module`导出值是指模块引入运行时产生的值，如果清除缓存后再次引入，会产生新的重新计算的`module`导出值）
- `commonjs`是同步加载，`NodeJs`本身提供这样的能力，在浏览器端不适合同步进行加载，于是有了其他加载规范。

> 参考文章
> 
> [前端科普系列-CommonJS：不是前端却革命了前端](https://zhuanlan.zhihu.com/p/113009496)
