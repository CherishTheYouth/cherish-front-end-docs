## 4.  原型式继承和寄生式继承

### 4.1 原型式继承

原型式继承的基本实现方式如下

```javascript
function getSubType (SuperType) {
    function F () {}
    F.prototype = SuperType
    return new F()
}
```

这种方法没有严格意义上的构造函数，其思想是借助原型，可以基于已有的对象创建新对象，同时还不必为此创建自定义的类型。

在getSubType函数内部，先创建一个临时性的构造函数，然后将传入的对象 SuperType 作为这个构造函数的原型，最后返回一个这个临时性构造函数的新实例。

这种原型式继承，要求必须有一个对象作为另一个对象的基础。将该对象传入getSubType函数，再根据具体需求对得到的对象加以修改即可。

```javascript
function getSubType (SuperType) {
    function F () {}
    F.prototype = SuperType
    return new F()
}

var superObject = {
    name: 'SUPER',
    element: ['AA', 'BB', 'CC', 'DD'],
    printElement: function () {
        console.log(this.element)
    }
}

var subA = getSubType(superObject)
subA.name = 'SUBA'
subA.element.push('EE')
subA.printElement() // ["AA", "BB", "CC", "DD", "EE"]
console.log(subA.name) // SUBA

var subB = getSubType(superObject)
subB.printElement() // ["AA", "BB", "CC", "DD", "EE"]
console.log(subB.name) // SUPER
```

ECMAScript5中的Object.create()方法规范化了原型式继承，这个方法接受两个参数，一个是用作新对象原型的对象，另一个是为新对象定义额外属性的对象。

### 4.2 寄生式继承

寄生式继承和原型式继承的思路类似。





## 5. 寄生组合式继承





