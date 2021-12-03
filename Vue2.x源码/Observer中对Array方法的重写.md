在`Observer`的构造器中有这样一段代码

```javascript
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
```

当`value`不是数组类型时，对`value`做键值的遍历操作，通过`defineReactive`进行`getter`、`setter`的设置。

当`value`为数组类型时，通过`observeArray`方法对子元素执行`Observe`方法。

在此之前，还根据`hasProto`做了一步操作，这一步的目的是修改该数组的原型方法。

#### hasProto

```javascript
export const hasProto = '__proto__' in {}
```

定义只有一行，作用就是判断浏览器环境中的`Object`是否有`__proto__`，便于我们兼容地使用不同方法更新数组原型方法。下面我们看下两种情况是如何处理的



#### protoAugment(*value*, *arrayMethods*)

有`__proto__`的处理情况是将`value`的`__proto__`直接指向`arrayMethod`。在`valu`e上调用数组方法时，会根据原型链进行查找，也就会在`arrayMethods`中找到相应方法



#### copyAugment（*value*, *arrayMethods*, *arrayKeys*）

没有`__proto__`情况下，该方法使用`Object.defineProperty`将`arrayMethods`所有的方法挂到`value`上



#### arrayMethods

- 首先该对象由`Object.create(Array.prototype)`进行创建，该对象原型链上游是`Array`。在这一层进行数组方法的重写，如果没有重写的方法则会再通过原型链向上查找，也就是使用`Array`原生方法。
  - 重写了以下方法：`push`,  `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`。在重写的过程，基本流程如下：
    - 先通过`Array.prototype`读取原生方法，用于后续执行。
    - 通过`Object.defineProperty`，对`arrMethods`的方法名进行重定义，为一个新函数。
  - 重写的函数中实际上也是执行刚才缓存的相应的原生方法，只是其中多了数据处理相关步骤。
    - 对于`push`、`unshift`、`splice`方法，因为是向数组内添加元素，所以也需要对添加的元素进行响应式处理。通过入参拿到添加的元素，将添加元素数组进行`observeArray`操作，从而达到新添加的数据变为响应式的目的。
    - 此外，最后还执行了`ob.dep.notify`，将更新派发下去，视图也进行更新。

