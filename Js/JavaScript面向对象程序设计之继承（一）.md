# 继承

## 1. 原型链式继承

### 1.1 原型模式

原型模式是JavaScript中创建对象的一种最常见的方式。JavaScript是一种弱类型的语言，没有类的概念，也不是一种面向对象的语言。但是，在JavaScript中，借助函数的原型（也就是prototype）可以实现类的功能。

使用原型模式创建对象的基本做法如下：

```javascript
function Person (name) {
    this.name = name // 私有属性
}
// 公共方法
Person.prototype.sayName = function () {
    console.log(this.name)
}

// 创建实例
var personA = new Person('A')
console.log(personA.name) // A
personA.sayName() // A

var personB = new Person('B')
console.log(personB.name) // B
personB.sayName() // B
```

在以上代码中，两个实例既拥有各自不同的属性 name, 又共享了 公有的方法 sayName()，这样就实现了类似于强类型语言中类的概念。

这种功能的实现，就得益于构造函数的prototype属性。下图展示了构造函数、原型、实例三者之间的关系。

![](C:\Users\yongbo.zeng\Pictures\Md-Picture\原型模式.png)

每个构造函数都有一个属性 prototype，prototype属性指向一个对象，该对象被构造函数的所有实例所共享，我们称这个对象为构造函数的**原型**。

原型与构造函数之间通过prototype属性和constructor属性相互联系。

实例与原型之间通过一个 `_proto_`属性（是浏览器中存在的一个虚拟的属性，各个浏览器的实现不同，谷歌中为`_proto_`）产生联系。

一个构造函数创建的不同的实例都可以通过 `_proto_`属性 访问到它的原型，这就是为什么上面的示例中两个示例可以访问到共同的 sayName() 方法的原因。

### 1.2 原型链式继承

### 1.2.1 原型链与原型链式继承

构造函数的原型并不是固定不变的，我们可以人为的切断构造函数与其原型之间的联系，也可以给构造函数指定自定义的原型。

```javascript
function Person (name) {
    this.name = name
}
Person.prototype = {
    constructor: Person,
    sayName: function () {
        console.log(this.name + '-hello')
    }
}
var personA = new Person('A')
personA.sayName() // A-hello

```

上面的代码我们自己给Person构造函数指定了一个原型，并添加了一个sayName()的公共方法。

原型链的实现就是基于原型可以重写这一基本前提。

我们结合下图来说明什么是原型链，它与JavaScript中实现类之间的继承的关系。

![](C:\Users\yongbo.zeng\Pictures\Md-Picture\JavaScript-原型链.png)

上图中，绿色线条所形成的链条即被我们称之为原型链。基于原型链，我们可以实现一种继承方式。

以下是一段基于原型链实现继承代码示例，为了区别于Java等语言中的class关键字，这里以Type表示类型，同时以 Super表示父，以Sub表示子，将SuperType称之为超类型，将SubType称之为子类型。

```javascript
function SuperType () {
    this.superName = 'superName'
}
SuperType.prototype.sayName = function () {
    console.log(this.name)
} 
function SubType (subName) {
    this.name = subName || 'subName'
}
SubType.prototype = new SuperType()
var subA = new SubType('A')
subA.sayName() // A
console.log(subA.superName) // superName
```

以上代码中，我们并没有为SubType定义sayName() 方法，SubType中也没有superName属性，但SubType创建的示例subA却可以使用sayName()方法打印出 自己的name属性，同时可以访问到并非自身定义的 superName属性，这是因为我们将SuperType的一个实例指定为了SubType的原型，因此subA与new SuperType()与 SuperType.prototype之间就有了一条原型链，subA因此可以访问到链条能触及的所有属性和方法。

这就是原型链实现继承的一个基本方式。

### 1.2.2 原型链式继承的缺陷

尽管，在理解了原型链后，原型链式的继承方式变得很好理解，使用也很简单。但是，原型链式的继承方式却并非完美的实现继承的解决方案。原型链式的继承也存在其自身的问题。让我们看一段代码：

```javascript
function SuperType () {
    this.element = ['A', 'B', 'C']
}
SuperType.prototype.printElement = function () {
    console.log(this.element)
}
function SubType () {
}
SubType.prototype = new SuperType()

var subA = new SubType()
subA.element.push('D')
subA.printElement() // ["A", "B", "C", "D"]
var subB = new SubType()
subB.element.push('E')
subB.printElement() // ["A,", "B", "C", "D","E"]
```

让我们感到奇怪的是，两个不同的实例分别向element属性中各自添加不同元素后，subB中打印的element属性中竟然包含了subA添加的元素。

这也很好理解，因为两个实例都继承自SuperType的同一个实例构成的原型，所以他们共享了超类型构造函数中的element属性，因此，理所应当的，前一个实例对element元素所作的更改会体现在后一个实例上。

但是，这并不是我们想要的结果。

我们知道，使用原型模式创建对象时，会把私有属性（方法）和共有属性（方法）分开定义，私有属性定义在构造函数中，公有属性定义在原型中。因此，显然，当我们使用原型链实现继承时，我们不仅继承了超类型实例的原型中的公有属性，**也继承了其构造函数中定义的私有属性**。**并且本应该是私有的属性，却因为它成了子类型的原型，而变成了子类型的公有属性。**

这是一个令人头疼的问题。

原型链式继承的另一个问题是，**子类型的构造函数中，无法给超类型传递参数**。这也局限了这种方式在实际开发中的应用。

### 缺点

> - 当使用原型链实现继承时，子类型不仅继承了超类型实例的原型中的公有属性，也继承了其构造函数中定义的私有属性。并且本应该是私有的属性，却因为它成了子类型的原型，而变成了子类型的公有属性；
> - 原型链式继承的另一个问题是，**子类型的构造函数中，无法给超类型传递参数**。

## 2. 借用构造函数

### 2.1 借用构造函数实现继承

为了解决原型链式继承所带来的问题，开发人员使用了一种新的技术，这种技术被称为**借用构造函数**的技术。其具体实现方法如下：

```javascript
function SuperType (name) {
    this.name = name || 'superName'
    this.element = ['A', 'B', 'C']
}
function SubType (name) {
    SuperType.apply(this, arguments) // SuperType.call(this, name)
}

var subA = new SubType('subA')
subA.element.push('D') 
console.log(subA.element) // ["A", "B", "C", "D"] 
console.log(subA.name) // subA
var subB = new SubType('subB')
subB.element.push('E')
console.log(subB.element) // ["A", "B", "C", "E"] 
console.log(subB.name) // subB
```

根据程序的执行现象，我们可以看到，SubType的两个实例，分别向element属性中添加了不同的元素，从打印结果发现，subA中添加的元素并没有反映到subB中，SubType的两个实例在继承了SuperType中的自有属性的同时，又各自保留了其属性的副本。

另外，在SubType构造函数中，调用SuperType构造函数时，子类型可以给超类型传递参数。

借用构造函数看似简单，却解决了原型链式继承中存在的两个让人头疼的问题。

实际上，借用构造函数的思想与我们平时编码时常用的技巧 **函数的提取** 类似，下面我们通过一段代码来简要的讲解一下这个过程（以下代码部分为伪代码，在此不探讨其合理性）：

```javascript
function SubType () {
    this.propertyA = 'propertyA'
    this.propertyB = 'propertyB'
    this.propertyC = 'propertyC'
    this.propertyD = 'propertyD'
    this.propertyE = 'propertyE'
}
```

以上是一个子类型的构造函数，其中有诸多属性，在许多其他的子类型中，这些属性也被需要。

这跟函数提取的场景很类似，在一个函数中，某些代码具备一定的复用性，此时，我们很容易的想到，把这些代码提取到一个单独的函数中，此后只要有需要的函数，都可以复用这些代码。这一步可以简单实现为如下：

```javascript
function SubType () {
   SuperType()
}
function SuperType () {
    this.propertyA = 'propertyA'
    this.propertyB = 'propertyB'
    this.propertyC = 'propertyC'
    this.propertyD = 'propertyD'
    this.propertyE = 'propertyE'
}
```

SubType构造函数内部调用SuperType构造函数，将SuperType中的属性复用到SubType中。

但是，如只是简单的提取，则会产生一个问题，SuperType中的this在SuperType被定义时，即已确定。SuperType被定义在全局作用域下，因此this指向全局作用域（一般是window）。简单的提取调用后，则SubType中实际上会变成这样：

```javascript
function SubType () {
    window.propertyA = 'propertyA'
    window.propertyB = 'propertyB'
    window.propertyC = 'propertyC'
    window.propertyD = 'propertyD'
    window.propertyE = 'propertyE'
}
```

当创建SubType的新实例时，通过调用相应的属性，会得到如下结果：

```javascript
function SubType () {
   SuperType()
}
function SuperType () {
    this.propertyA = 'propertyA'
    this.propertyB = 'propertyB'
    this.propertyC = 'propertyC'
    this.propertyD = 'propertyD'
    this.propertyE = 'propertyE'
}

var subA = new SubType()
console.log(subA.propertyA) // undefined
console.log(window.propertyA) // propertyA
```

这与以上的描述一致。

为了让SuperType在调用时，其包含的属性能够正确的复用到SubType中，只需要将SuperType的执行环境绑定到SubType上即可。使用apply()方法和call()方法都可以很容易实现这一步。上述实现就变为：

```javascript
function SubType () {
   SuperType.apply(this,arguments)
}
function SuperType () {
    this.propertyA = 'propertyA'
    this.propertyB = 'propertyB'
    this.propertyC = 'propertyC'
    this.propertyD = 'propertyD'
    this.propertyE = 'propertyE'
}

var subA = new SubType()
console.log(subA.propertyA) // propertyA
console.log(window.propertyA) // propertyA
```

以上的过程或许不够准确，但的确可以帮助理解借用构造函数的实现思路。

由以上的代码示例，我们可以看到，**借用构造函数的的确确是解决了原型链链式继承方法的缺陷**。

- 每个实例都可以保持超类型中自有属性的私有性，**每个子类实例中都可以保有超类型中自有属性的一个副本**，子类实例之间对继承而来的自有属性的操作不会相互干扰；
- **子类型的构造函数可以向超类型的构造函数中传递参数**；

### 2.2 借用构造函数的缺陷

但是，借用构造函数中也存在着令人头疼的问题。

细心的读者会发现，在以上的关于借用构造函数讲解示例中，竟没有出现一次公共方法的调用。没错，这正是问题所在。

简单来说就是，我（借用构造函数）做不到啊。

也许你会进行一些尝试，我给超类型的原型定义公共方法行不行呢？我直接在超类型构造函数上定义一个方法行不行呢？一起来看下面这两段代码：

示例一：给超类型的原型定义公共方法

```javascript
function SuperType (name) {
    this.name = name || 'superName'
    this.element = ['A', 'B', 'C']
}
SuperType.prototype.sayName = function () {
    console.log(this.name)
}
function SubType (name) {
    SuperType.apply(this, arguments) // SuperType.call(this, name)
}

var subA = new SubType('subA')
subA.sayName() // Uncaught TypeError: subA.sayName is not a function
```

毫不吃惊，程序执行出错， subA.sayName不是一个function;

借用构造函数的本质仅仅是将超类型中的属性复制一份到子类型中，并将其属性的执行环境绑定到子类型上，因此子类型在执行完超类型构造函数那一刻，子类型和超类型之间就切断了联系，子类型的实例又怎么可能访问到超类型的原型方法呢。

实例二：直接在超类型构造函数上定义方法

```javascript
function SuperType (name) {
    this.name = name || 'superName'
    this.element = ['A', 'B', 'C']
    this.sayName = function () {
        console.log(this.name)
    }
}
function SubType (name) {
    SuperType.apply(this, arguments) // SuperType.call(this, name)
}
var subA = new SubType('subA')
subA.sayName() // 'subA'
var subB = new SubType('subB')
subB.sayName() // 'subB'
```

看似可行，实则？？？

```javascript
console.log(subA.sayName === subB.sayName) // false
```

呃。。。。。

虽然都实现了相同的功能，但两个方法并不是同一个方法。还是上面说的，**借用构造函数仅仅只是复制了一份超类型中的属性和方法，这并不是复用**，借用构造函数无法实现公共方法的复用。

基于借用构造函数的以下两个缺陷：

- **无法定义子类型可复用的公共方法；**
- **无法访问超类型的原型；**

借用构造函数在实际应用中很少单独使用。

## 3. 组合继承

虽然，原型链模式和借用构造函数模式都无法完美实现继承，但所幸二者的缺陷可以互补。自然而然的，一种相对完美的解决方案出现了，即组合继承。

将原型链式继承和借用构造函数继承组合起来，使用原型链模式实现对超类型的公共属性和公共方法的继承，使用借用构造函数模式实现对超类型中自有属性的继承。这样，既通过在原型上定义方法实现了函数的复用，又能够保证每个子类实例都能保有一份超类型中的自有属性。

组合继承的实现方法如下：

```javascript
function SuperType (name) {
    this.name = name || 'superName'
    this.element = ['A', 'B', 'C']
}
SuperType.prototype.sayName = function () {
    console.log(this.name)
}
SuperType.prototype.printElement = function () {
    console.log(this.element)
}
function SubType (name) {
    SuperType.apply(this, arguments)
}
SubType.prototype = new SuperType ()
var subA = new SubType('subA')
subA.element.push('subA')
subA.sayName() // subA
subA.printElement() // ["A", "B", "C", "subA"]
var subB = new SubType('subB')
subB.element.push('subB')
subB.sayName() // subB
subB.printElement() // ["A", "B", "C", "subB"]
console.log(subA.sayName === subB.sayName) // true
```

**组合继承**，也称之为 **伪经典继承**，指的是**将原型链和借用构造函数的技术组合在一起，从而发挥二者之长的一种继承模式**。

其思路是，**使用原型链实现对原型方法的继承**，**借用构造函数实现对实例属性的继承**。

```javascript
function SuperType () {
     this.element = ['AA', 'BB', 'CC']
}
SuperType.prototype.printElement = function () {
    console.log(this.element)
} 
function SubType () {
    SuperType.apply(this, arguments)
}
SubType.prototype = new SuperType()
SubType.prototype.constructor = SubType

var subA = new SubType()
subA.element.push('DD')
subA.printElement() // ["AA", "BB", "CC", "DD"]

var subB = new SubType()
subB.element.push('EE')
subB.printElement() // ["AA", "BB", "CC", "EE"]

console.log(subA.printElement === subB.printElement) // true
```

组合继承解决了原型链模式和借用构造函数模式的缺陷，融合了它们的优点，是JavaScript中最常用的继承模式。

但组合模式也有自己的缺陷。下文中第5种继承实现方法将解决组合模式的缺陷，在此之前，我们先看第四种继承的实现方法。

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

这种方法没有严格意义上的构造函数，其思想是借助原型，可以**基于已有的对象创建新对象**，同时还不必为此创建自定义的类型。

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

ECMAScript5中的**Object.create()方法**正是运用了这种继承方式，并且规范了这种继承方式。

### 4.2 Object.create() 方法

**Object.create()方法**接受两个参数，一个是**用作新对象原型的对象**，另一个是**为新对象定义额外属性的对象。**

第二个参数的每个属性都是自定义的，以这种方式定义的任何属性**都会覆盖原型对象上的同名属性**。

**`Object.create()`**方法创建一个新对象，使用现有的对象来提供新创建的对象的`__proto__`。

```js
var person = {
  isHuman: false,
  printIntroduction: function() {
    console.log(`My name is ${this.name}. Am I human? ${this.isHuman}`);
  }
};

var me = Object.create(person);

me.name = 'Matthew'; // "name" is a property set on "me", but not on "person"
me.isHuman = true; // inherited properties can be overwritten

me.printIntroduction();
// expected output: "My name is Matthew. Am I human? true"
```

其使用语法如下：

```js
Object.create(proto，[propertiesObject])
```

- **proto**：新创建对象的原型对象。
- **propertiesObject**：可选。需要传入一个对象。如果该参数被指定且不为`undefined`，该传入对象的自有可枚举属性(即其自身定义的属性，而不是其原型链上的枚举属性)将为新创建的对象添加指定的属性值和对应的属性描述符。

### 4.2 寄生式继承

寄生式继承和原型式继承的思路类似。





## 5. 寄生组合式继承





## 6.ES6中的继承

es6 代码转 es5 代码

```js
"use strict";

var _createClass = function () { function defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } } return function (Constructor, protoProps, staticProps) { if (protoProps) defineProperties(Constructor.prototype, protoProps); if (staticProps) defineProperties(Constructor, staticProps); return Constructor; }; }();

var _typeof = typeof Symbol === "function" && typeof Symbol.iterator === "symbol" ? function (obj) { return typeof obj; } : function (obj) { return obj && typeof Symbol === "function" && obj.constructor === Symbol && obj !== Symbol.prototype ? "symbol" : typeof obj; };

function _possibleConstructorReturn(self, call) { if (!self) { throw new ReferenceError("this hasn't been initialised - super() hasn't been called"); } return call && (typeof call === "object" || typeof call === "function") ? call : self; }

function _inherits(subClass, superClass) { if (typeof superClass !== "function" && superClass !== null) { throw new TypeError("Super expression must either be null or a function, not " + typeof superClass); } subClass.prototype = Object.create(superClass && superClass.prototype, { constructor: { value: subClass, enumerable: false, writable: true, configurable: true } }); if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass; }

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var _createClassTemp = function () {
    function defineProperties(target, props) {
        for (var i = 0; i < props.length; i++) {
            var descriptor = props[i];
            descriptor.enumerable = descriptor.enumerable || false;
            descriptor.configurable = true;
            if ("value" in descriptor) descriptor.writable = true;
            Object.defineProperty(target, descriptor.key, descriptor);
        }
    }
    return function (Constructor, protoProps, staticProps) {
        if (protoProps) defineProperties(Constructor.prototype, protoProps);
        if (staticProps) defineProperties(Constructor, staticProps);
        return Constructor;
    };
}();

function _inheritsTemp(subClass, superClass) {
    if (typeof superClass !== "function" && superClass !== null) {
        throw new TypeError("Super expression must either be null or a function, not " + (typeof superClass === "undefined" ? "undefined" : _typeof(superClass)));
    }
    subClass.prototype = Object.create(superClass && superClass.prototype, {
        constructor: {
            value: subClass,
            enumerable: false,
            writable: true,
            configurable: true
        }
    });
    if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
}

var Person = function () {
    function Person() {
        _classCallCheck(this, Person);

        this.drink = function () {
            console.log('drink');
        };

        this.name = 'Cherish';
    }

    _createClass(Person, [{
        key: "travel",
        value: function travel() {
            console.log('travel');
        }
    }], [{
        key: "eat",
        value: function eat() {
            console.log('eat');
        }
    }]);

    return Person;
}();

Person.eat();

var Man = function (_Person) {
    _inherits(Man, _Person);

    function Man() {
        _classCallCheck(this, Man);

        var _this = _possibleConstructorReturn(this, (Man.__proto__ || Object.getPrototypeOf(Man)).call(this));

        _this.name = 'Man';
        return _this;
    }

    _createClass(Man, [{
        key: "code",
        value: function code() {
            console.log('code');
        }
    }]);

    return Man;
}(Person);

console.log(Man.prototype);
var personA = new Person();
personA.code();
```



### 6.1 简介

es6中引入了 `class` 的概念，同时也支持使用新的方式，实现继承，也就是 `extends` 方式。

class 通过 extends 关键字实现继承，这比 ES5 的通过修改原型链实现继承，要清晰和方便很多。

**示例**：

```js
class Point {
    constructor (x, y) {
        this.x = x
        this.y = y
    }
    toString () {
        
    }
}
class ColorPoint extends Point {
    constructor (x, y, color) {
        super (x, y) // 调用父类的constructor(x, y)
        this.color = color
    }
    toString () {
        return this.color + super.toString() // 调用父类的toString()
    }
}
```

**super 关键字**

super表示父类的构造函数，用来新建父类的 this 对象。

子类必须在 constructor 方法中调用 super 方法，否则新建实例时会报错 。

这是因为子类没有自己的 this 对象，而是继承父类的 this 对象，然后对其进行加工。如果不调用 super 方法，子类就得不到 this 对象。  

```js
class Point { /* ... */ }
class ColorPoint extends Point {
    constructor() {
    }
}
let cp = new ColorPoint(); // ReferenceError
```

![image-20210615161024942](C:\Users\yongbo.zeng\Pictures\Md-Picture\image-20210615161024942.png)

**es6 extends 与 es5 继承的差异**

- es5:

> ES5 的继承，实质是**先创造子类的实例对象 this** ，然后**再将父类的方法添加到 this 上**面。

- es6:

> ES6 的继承机制完全不同，实质是**先创造父类的实例对象 this** （所以必须先调用 super 方法），然后**再用子类的构造函数修改 this** 。  

如果子类没有定义 `constructor` 方法，这个方法会被默认添加，也就是说，不管有没有显示定义，任何一个子类都有 `constructor` 方法。

```js
class ColorPoint extends Point {
    
}

// 等同于
class ColorPoint extends Point {
    constructor (...args) {
        super(...args)
    }
}
```

在子类的构造函数中，只有调用 `super` 之后，才可以使用 `this` 关键字，否则会报错。

这是因为，子类实例的构建（子类的this）是基于对父类实例（父类this）的加工，只有 `super` 方法才能返回父类的实例。

示例：

```js
class Point {
    constructor(x, y) {
        this.x = x
        this.y = y
    }
}
class ColorPoint extends Point {
    constructor(x, y, color) {
        this.color = color; // ReferenceError
        super(x, y);
        this.color = color; // 正确
    }
}
```

![image-20210615170148411](C:\Users\yongbo.zeng\Pictures\Md-Picture\image-20210615170148411.png)



### 6.2 super 关键字

super 的两种用法：

- 当作函数使用 ；
- 当作对象使用  ；

**1. 当做函数使用**

> super 作为函数调用时，代表**父类**的构造函数。  
>
> ES6 要求，子类的构造函数必须执行一次 super 函数。  

作为函数时，super() 只能用在子类的构造函数中，用在其他的地方就会报错。

![image-20210615172659131](C:\Users\yongbo.zeng\Pictures\Md-Picture\image-20210615172659131.png)



2. 当做对象使用

### 6.3 原生构造函数的继承



### 6.4 源码分析











