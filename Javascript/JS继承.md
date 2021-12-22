![image-20211218152228647](https://cdn.jsdelivr.net/gh/Bacuuu/sleeping-image@master/uPic/05r5Sj.jpg)

​		JS原型链原则是“子债父偿”。

**父子关系如何产生？**

​		实例是由new操作符产生的，所以实例和原型的关联也是在这里产生的。其中有一个步骤是将`__proto__`指向构造函数的`prototype`。这里就是建立了关联。在调用实例的属性或者方法时，如果没有找到则会通过`__proto__`进行向上的查找，如果仍然找不到，则回一直向上寻找。



#### 原型链继承

```javascript
subConstructor.prototype = new superConstructor()
```

​		该方法就是单纯的将父构造函数生成的实例作为子构造函数的`prototype`，这样当使用子构造函数进行新实例的构建时，所有新实例的`__proto__`会指向共同的 由父构造函数生成的实例

​		第一个问题是，该方法轻易实现了继承，但是由于`__proto__`是共用的一个实例，所以会造成对实例属性（引用类型）的修改。虽然是对同一父实例进行修改，呈现出来的是 两个子实例之间会进行 互相修改。

​		第二个问题是子构造函数无法进行传参。



#### 构造函数继承

```javascript
subConstructor () {
	superConstructor.call(this)
}
```

​		由于**原型链继承**共用一个实例，所以我们想到将父实例属性拷贝一份作为子实例属性。在子构造函数中去使用当前上下文调用父构造函数，达到将父构造函数中属性复制到子属性的目的，多次调用子构造函数时，不会共用一个父实例属性。而且在该方式下，由于是直接调用父构造函数，所以也能够传递参数，从而构造不同的属性值。

​		这个方法虽然解决了**原型链继承**的问题，但是并没有达到**继承**的效果，只是将得到了父级的属性，并没有和父级建立关联。



#### 组合继承

​		组合，很明显是将上述的两种方式进行结合。由于两种方式并没有什么冲突的地方，所以结合方式也很简单。

```javascript
subConstructor () {
	superConstructor.call(this)
}
subConstructor.prototype = new superConstructor()
// 将构造函数指向到本身
subConstructor.prototype.constructor = subConstructor
```

​		组合继承就是将原型链、构造函数两个步骤都执行。这里的最后一步将原型的构造函数指向自己，这一步其实并非必须的。先思考如果没有这一步，`subConstructor.prototype`是父实例，它本身没有`constructor`属性，根据原型链特性，会去`subConstructor.prototype.__proto__`查找，也就是`superConstructor.prototype`，从而查找到`constructor`为`superConstructor`。从原型链的角度来说，这是不正确的。我们写下如下代码

```javascript
const sub = new subConstructor()
sub.__proto__.constructor === subConstructor  // false
```

​		这其实是有些违反我们对原型链的认识的，虽然在使用过程中并不会影响原型链的属性查找。

​		网上说该方法会调用两次`superConstructor`方法，会造成不必要的消耗。我认为用于创建`prototype`这次方法调用时一次性的，后续每次生成子实例的时候才会调用`superConstructor.call(this)`，感觉只多了一次调用。



#### 原型式继承

```javascript
function object(obj) {
	function F(){}
	F.prototype = obj
	return new F()
}
const o = { a: 123}
const b = object(o)
// 达到 b 继承于 o 的目的
```

​		初看觉得和**原型链继承**很像，仔细一想他们的差别主要在于，原型链继承是构造函数进行继承的关系，原型式继承是直接构建出继承某实例的实例。

​		其实上述的`object`方法是`object.create`的一种`polyfill`实现，在ES5+中，能够使用`Object.create()`达到相同的作用，我们顺便看看`Objtct.create`的描述：

>  `Object.create()`方法创建一个新对象，使用现有的对象来提供新创建的对象的__proto__。

​		这样方法的缺点和原型链差不多，指向原型同一属性，会被多个子实例共用；无法进行参数传递。



#### 寄生式继承

```javascript
function createAnother(original){
  var clone = object(original); // 通过调用 object() 函数创建一个新对象
  clone.sayHi = function(){  // 以某种方式来增强对象
    alert("hi");
  };
  return clone; // 返回这个对象
}
```

和原型式继承很相似，只是会对继承后的对象做拓展。



#### 寄生组合式继承

```javascript
function inheritPrototype(subType, superType){
  var prototype = Object.create(superType.prototype); // 创建对象，创建父类原型的一个副本
  prototype.constructor = subType;                    // 增强对象，弥补因重写原型而失去的默认的constructor 属性
  subType.prototype = prototype;                      // 指定对象，将新创建的对象赋值给子类的原型
}
// 父类初始化实例属性和原型属性
function SuperType(name){
  this.name = name;
  this.colors = ["red", "blue", "green"];
}
// 借用构造函数传递增强子类实例属性（支持传参和避免篡改）
function SubType(name, age){
  SuperType.call(this, name);
  this.age = age;
}

// 将父类原型指向子类
inheritPrototype(SubType, SuperType);

```

首先来回顾下

原型式继承是` (实例) -> 实例`，传入实例，构造继承于该实例的新实例，并返回

原型链继承是`F(构造函数)`，直接对构造函数进行改造，设置`prototype`为父实例，进行了原型链修改。

寄生组合式继承是`(构造函数) -> 构造函数 `，而且构造函数本身内部会调用父构造函数，同**构造函数继承**。



##### 和组合继承的差异

```javascript
subConstructor.prototype = new superConstructor()
subConstructor.prototype.constructor = subConstructor

var prototype = Object.create(superConstructor.prototype);
prototype.constructor = subConstructor;
subConstructor.prototype = prototype; 
```

​		两者差异在于 构造父实例的方法。一个是直接通过父构造函数进行`new`操作，一个是通过`Object.create`操作。

​		`Object.create`创建的实例不会继承构造函数本身的属性和方法，只继承原型链上的属性、方法。



##### 为何优于组合继承

​		先看基于组合继承的下列代码

```javascript
const superConstructor = function  () { this.value = 123 }
const subConstructor = function () {superConstructor.call(this)}
// subConstructor和superConstructor已经通过组合继承 关联继承关系
const t = new subConstructor()
// {value: 123}
delete t.value
console.log(t.value)
// 123
```

​		由于子构造函数内部`sub.call(this)`，组合继承复制了父实例的属性；又因为原型链指向父构造函数构建的实例，所以删除本身的数据后，仍然能够根据原型链查找到父实例的数据。这个数据对于我们来说是没有意义的，我们的需求只是想根据父实例拿到原型而已。而通过`Object.create`创建的方式，没有对父构造函数的属性、方法进行继承，避免多调用一次构造函数的消耗。