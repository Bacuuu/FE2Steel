👋本文总结下`javascript`各种常用函数代码的实现

参考自[「中高级前端面试」JavaScript手写代码无敌秘籍](https://juejin.cn/post/6844903809206976520#heading-10)、[前端面试常见的手写功能](https://juejin.cn/post/6873513007037546510)

#### new

[它做了什么](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new)：

1. 创建对象

2. 建立原型链关联(`__proto__`)

3. 将新创建的对象作为`this`上下文

4. 如果函数没有返回对象，返回`this`

```javascript
const _new = function (func) {
    // 1
    var res = {};
    // 2
    if (func.prototype !== null) {
        res.__proto__ = func.prototype;
    }
    // 3
    var ret = func.apply(res, Array.prototype.slice.call(arguments, 1));
    // 4
    if ((typeof ret === "object" || typeof ret === "function") && ret !== null) {
        return ret;
    }
    return res;
}
// 调用
_new(A)
```



#### JSON

[介绍](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/JSON)

- JSON.parse

  ```javascript
  const parse = function (sJSON) {
    // 添加括号，避免识别json中的 {} 为代码块
    return eval('(' + sJSON + ')')
  }
  
  // 更推荐的方法，使用IIFE
  const parse = function (obj) {
      return Function('"use strict";return (' + obj + ')')();
  }
  ```

  

- JSON.stringify

  因为传入的值有很多可能性，具体转化细节见[MDN的描述](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify#%E6%8F%8F%E8%BF%B0)

  ```javascript
  const stringify = (function () {
        // toString方法
        var toString = Object.prototype.toString;
        // 兼容判断是否数组
        var isArray = Array.isArray || function (a) { return toString.call(a) === '[object Array]'; };
        // 转译 字典
        var escMap = {'"': '\\"', '\\': '\\\\', '\b': '\\b', '\f': '\\f', '\n': '\\n', '\r': '\\r', '\t': '\\t'};
        // 转译 函数
        var escFunc = function (m) { return escMap[m] || '\\u' + (m.charCodeAt(0) + 0x10000).toString(16).substr(1); };
        var escRE = /[\\"\u0000-\u001F\u2028\u2029]/g;
        return function stringify(value) {
          if (value == null) {
            // null
            return 'null';
          } else if (typeof value === 'number') {
            // 数字，直接转化
            return isFinite(value) ? value.toString() : 'null';
          } else if (typeof value === 'boolean') {
            // 布尔
            return value.toString();
          } else if (typeof value === 'object') {
            // 引用类型
            if (typeof value.toJSON === 'function') {
              // 使用toJSON，例如Date对象
              return stringify(value.toJSON());
            } else if (isArray(value)) {
              // 数组，递归处理元素并拼接
              var res = '[';
              for (var i = 0; i < value.length; i++)
                res += (i ? ', ' : '') + stringify(value[i]);
              return res + ']';
            } else if (toString.call(value) === '[object Object]') {
              // 真正的Object，递归处理键值
              var tmp = [];
              for (var k in value) {
                if (value.hasOwnProperty(k)) tmp.push(stringify(k) + ': ' + stringify(value[k]));
              }
              return '{' + tmp.join(', ') + '}';
            }
          }
          return '"' + value.toString().replace(escRE, escFunc) + '"';
        };
      })()
  ```



#### call

指定函数的`this`，并执行函数。这里的主要问题是如何使函数的上下文更换，很容易想到，将这个函数设置为属性然后进行调用即可。

1. 将函数赋值给新上下文(`content`)的，作为唯一属性（Symbol）
2. 截取传入函数的参数值
3. 在上下文(`content`)中调用函数
4. 删除附加给上下文(`content`)的属性

```javascript
Function.prototype._call = function(content = window) {
  	const symbol = Symbol()
    content.symbol = this;
    let args = [...arguments].slice(1);
    let result = content.symbol(...args);
    delete content.symbol;
    return result;
}

```



#### apply

同`call`，只是处理入参有点区别

```javascript
Function.prototype._apply = function(content = window) {
  	const symbol = Symbol()
    content.symbol = this;
    let args = arguments[1] || [];
    let result = content.symbol(...args);
    delete content.symbol;
    return result;
}
```

#### bind

> `bind()` 方法创建一个新的函数，在 `bind()` 被调用时，这个新函数的 `this` 被指定为 `bind()` 的第一个参数，而其余参数将作为新函数的参数，供调用时使用。
>
> `function.bind(thisArg[, arg1[, arg2[, ...]]])`

​		`call` `apply` `bind`作用都是改变函数上下文，前两者只是在单次调用时改变，而`bind`是创建一个具有新上下文的函数，也不是马上调用。

​		`bind`函数第一个参数是新的上下文，后续参数是对函数参数进行依次填充。对于返回函数，如果继续传入参数值，则是放到`bind`时的参数后面。

```javascript

Function.prototype._bind = function(content) {
  	// 确定调用者是函数类型
    if(typeof this != "function") {
        throw Error("caller is not a function")
    }
  	// fn是 修改函数 的原型
    let fn = this;
  	// 过滤 获取 除去content的参数
    let args = [...arguments].slice(1);
    // 构造一个函数，进行上下文更改的调用
  	// 将参数进行拼接
    let resFn = function() {
      	// 这里的 instanceof 判断的作用在于
      	// 如果将该函数作为构造函数，this instanceof resFn === true
        return fn.apply(this instanceof resFn ? this : content,args.concat(...arguments) )
    }
    // 创建新构造函数，和当前函数的prototype指向同一实例
    function tmp() {}
    tmp.prototype = this.prototype;
  	// 确保新的函数 和 原函数 保持原有的继承关系
    resFn.prototype = new tmp();
    return resFn;
}
```

