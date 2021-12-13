👋本文总结下`javascript`各种常用函数代码的实现

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