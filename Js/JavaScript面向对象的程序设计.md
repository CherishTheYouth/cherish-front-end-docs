# JavaScript面向对象的程序设计（一）——对象的创建

## 一、Object构造函数

类似Java等面向对象语言中创建对象的语法，在 `JavaScript`中可以通过执行 `new`操作符后跟要创建的对象类型的名称来创建。`JavaScript`中通过如下方式可以创建一个`Object`对象：

```javascript
var obj = new Object()
// 可以像这样添加属性
obj.name = 'AAA'
obj.age = 26
```

在 `ECMAScript`中，如果不给构造函数传递参数，则可以省略 `Object`后的括号，写为：

```javascript
var obj = new Object
```

但是这种写法并不推荐。

## 二、对象字面量

创建对象的第二种方法为：**对象字面量**（或对象直接量）

创建对象最简单的一种方式就是在 `JavaScript`代码中使用**对象字面量**，这在我们平时的项目中使用的最多。**对象字面量**是由若干**名/值对**组成的映射表，**名/值对**之间用逗号分隔，整个映射表用花括号括起来。示例如下：

```javascript
var empty = {} // 没有任何属性的对象
var point = {
    x: 0,
    y: 0
} // 两个属性
var book = {
    "main book": "JavaScript", // 属性名字有空格，则必须用字符串表示
    'sub-title': "对象的创建"， // 属性名字有连字符，则必须用字符串表示
    author: {  // 属性可以是一个对象
    	firstname: "David",
    	surname: "Flanagan"
	}
}
```

> 使用**Object构造函数**和**对象字面量** 都可以用来创建**单个对象**，但在碰到使用同一个接口创建很多**同类型对象**时，就会产生大量重复的代码，如下示例：

```javascript
var book1 = {
    "main book": "JavaScript", // 属性名字有空格，则必须用字符串表示
    'sub-title': "对象的创建"， // 属性名字有连字符，则必须用字符串表示
    author: {  // 属性可以是一个对象
    	firstname: "David",
    	surname: "Flanagan"
	}
}
var book2 = {
    "main book": "Java", 
    'sub-title': "面向对象"， 
    author: {  
    	firstname: "爱谁谁",
    	surname: "Flanagan"
	}
}
var book3 = {
    "main book": "C语言", 
    'sub-title': "鼻祖"， 
    author: {  
    	firstname: "爱谁谁",
    	surname: "Flanagan"
	}
}
```

## 三、 工厂模式

使用工厂模式，可以抽象创建具体对象的过程，如下示例所示：

```javascript
function createPerson (name, age, job) {
    var obj = new Object()
    obj.name = name
    obj.age = age
    obj.job = job
    obj.sayName = function () {
        alert(this.name)
    }
    return obj
}
var person1 = createPerson('Bob', 20, 'Student')
var person2 = createPerson('Jack', 26, 'Software Engineer')
```

函数 `createPerson()`能够根据接受的参数来构建一个包含所有必要信息的Person对象，可以无数次的调用这个函数，每次都会返回一个包含三个属性和一个方法的对象。

> 工厂模式虽然解决了创建多个相似对象的问题，但却没有解决对象识别的问题（怎么知道该对象表示的是一个Person的类型）

## 四、 构造函数模式

### 4.1 构造函数模式

ECMAScript中的构造函数可以用创建特定类型的对象，比如Object和Array这样的原生构造函数。

此外，也可以创建自定义的构造函数，从而定义自定义类对象类型的属性和方法。可以使用构造函数对以上的示例进行重写：

```javascript
function Person (name, age, job) {
    this.name = name
    this.age = age
    this.job = job
    this.sayName = function () {
        alert(this.name)
    }
}
var person1 = new Person('Bob', 20, 'Student')
var person2 = new Person('Jack', 26, 'Software Engineer')
```

重写之后，`Perosn()`函数取代了 `createPerson()` 函数，它们之间存在以下不同：

- 没有显式的创建对象；
- 直接将属性和方法赋给this对象
- 没有return语句

要创建Person的新实例，必须使用`new`操作符，以这种方式调用构造函数，实际上会经过4个步骤：

- 创建一个新对象；
- 将构造函数的作用域赋给新对象（因此，this就指向了这个变量）；
- 执行构造函数中的代码，为这个新对象添加属性；
- 返回新对象；

上面例子中，`person1`和`person2`中都保存着`Person`类型的一个不同的实例，这两个对象都有一个`constructor`(构造函数)属性，该属性指向`Person`

```javascript
function Person (name, age, job) {
    this.name = name
    this.age = age
    this.job = job
    this.sayName = function () {
        alert(this.name)
    }
}
var person1 = new Person('Bob', 20, 'Student')
var person2 = new Person('Jack', 26, 'Software Engineer')
console.log( person1.constructor == Person) // true
console.log( person2.constructor == Person) // true
```

> constructor 保存着用于创建当前对象的构造函数，是每个Object对象中都存在的属性。以上实例中，创建当前对象的构造函数就是 Person。constructor属性是来自实例的原型，这在后面会讲到。

创建自定义的构造函数意味着将来可以将它的实例标识为一种特定的实例，这正是构造函数模式胜过工厂模式的地方。

### 4.2 构造函数模式的问题

使用构造函数的问题就在于，每个方法都要在每个实例中重新创建一遍

以上的person1和person2都含有一个sayName()方法，但这两个方法并非相同的 `Function`实例，因此，每定义一个实例，都要在其内部重新创建一个新的sayName()方法。

```javascript
// 以上的实例，从逻辑的角度讲，也可以这样定义
function Person (name, age, job) {
    this.name = name
    this.age = age
    this.job = job
    this.sayName = new Function("alert(this.name)")
}
```

```javascript
// person1和person2的sayName()方法是不相等的
console.log( person1.sayName == person2.sayName) // false
```

以上代码可证明person1和person2的sayName()方法是不相等的。

然而，每个新的Person实例中的sayName()方法所完成的事情实际上是相同的，创建两个完成相同任务的的Function实例的确没有必要。

我们不必在执行代码前就将函数绑定在特定实例上，大可像下面这样，通过把函数定义转移到构造函数外部来解决这个问题。

```javascript
function Person (name, age, job) {
    this.name = name
    this.age = age
    this.job = job
    this.sayName = sayName
}
function sayName () {
    console.log(this.name)
}
var person1 = new Person('Bob', 20, 'Student')
var person2 = new Person('Jack', 26, 'Software Engineer')
console.log(person1.sayName())
```

在以上的例子中，把 `sayName()`函数的定义转移到了构造函数外部，而在构造函数内部，将`sayName`属性设置为全局的`sayName`函数，这样一来，由于sayName包含的是一个指向全局函数的指针，因此，person1和person2对象就共享了全局作用域中定义的同一个sayName()函数，以下代码可以证明，两个实例的`sayName`函数是相同的函数：

```javascript
console.log( person1.sayName == person2.sayName) // true
```

这种处理方式的确解决了多次创建一个实现相同功能的函数的问题，但新的问题出现了，

- 我们定义的这个全局函数，实际上**只能被Person类型的实例调用**，这让全局函数的全局作用域显得有点**名不副实**。
- 如果对象需要定义很多方法，那么就要定义很多个全局函数，这让我们自定义的这个Person引用类型的**封装性极差**了。

这些问题可以通过使用原型模式来解决

## 五、原型模式。

### 5.1 三个属性（三个指针）

#### 5.1.1 prototype 

`prototype`属性是**每个函数**都包含的一个属性，是一个指针，指向一个对象，这个对象的作用是：包含可被**特定的类型**的**所有实例**共享的属性和方法。

> 如，`Person`()函数的的`prototype`属性，指向的就是特定类型 Person 的所有实例`person1`、`person2`所共享的属性和方法。

这里可以这样来讲，Person()函数的`prototype`属性就是 `perosn1`和`person2`的原型对象。

> 需要注意的是，prototype属性是只属于函数的，实例和实例的原型都没有这个属性

```javascript
// 控制台下
Person instanceof Function  // true
Person.prototype // 它的原型对象
person1 instanceof Function // false
person1.prototype // undefined
```

使用原型的好处是，可以让所有的实例共享它所包含的属性和方法。不必在构造函数中定义特定对象实例的信息，而是可以将这些信息直接添加到原型对象中

```javascript
function Person () {
}
Person.prototype.name = "Jack"
Person.prototype.age = 26
Person.prototype.job = "Software Engineer"
Person.prototype.sayName = function () {
	alert(this.name)    
}
var person1 = new Person()
person1.sayName() // Jack
var person2 = new Person()
person2.sayName() // Jack
console.log( person1.sayName == person2.sayName) // true
```

#### 5.1.2 [[prototype]]

当调用构造函数创建一个新实例后，该实例的内部将包含一个指针（内部属性）[[protoytype]]，指向构造函数的原型对象，在主流的浏览器中，这个属性就是 `_prpto_`属性。需要明确的是，`_proto_`属性连接的是**实例**和**构造函数的原型对象**，而不是存在于 **实例**和**构造函数**之间。

`_proto_`是任何对象的实例都拥有的一个属性：

- `person1`、`person2`是`Person`构造函数的一个实例，它们的`_proto_`指向起原型实例；
- `Person`本质是一个 `Function`的实例，所以它的`_proto_`指向的是 `Function.prototype`, `Function.prototype`的`_proto_`指向`Object.prototype`;
- `Object`的原型也有`_proto_`属性，但其指向为`null`;

```javascript
// 控制台下
Object.prototype._proto_ // null
```

#### 5.1.3 constructor

默认情况下，所有的原型对象都会自动获取一个 `constructor`属性，这个属性指向 `prototype`所在的函数（或者说我们自定义的特定的类）

`constructor`属性是实例**从原型中继承过来**的属性，虽然从`Person`、`person1`和原型中，都可以获取其 `constructor`的值，但是，`constructor`不是实例自有的属性。

```javascript
person1.hasOwnProperty('name') // true

Person.hasOwnProperty('constructor') // 
Person.prototype.hasOwnProperty('constructor') // true

Function.hasOwnProperty('constructor') // false
Function.prototype.hasOwnProperty('constructor') //true
```

### 5.2 理解原型对象

下图展示了**构造函数**、**原型对象**和**实例**之间的关系

![image-20201102191040862](F:\Study\JavaScript\Doc\imags\构造函数-原型对象-实例之间的关系.png)

> `Person`构造函数的`prototype`指向了`Person`的原型对象，原型对象的`constructor`属性指向`Person` 构造函数。
>
> `Person`的实例 `person1`和`person2`都包含一个`_proto_`属性，这个属性指向的是`Person`的原型对象。对象实例和构造函数之间并没有直接的关系。

![image-20201102192126865](F:\Study\JavaScript\Doc\imags\Person.prototype=person1._proto_.png)

### 5.3 读取对象的属性

每当代码读取某个对象的属性的时候，都会执行一次搜索。搜索首先从**对象实例本身**开始，如果在实例中找到了具有指定名字的属性，则返回该属性的值，并终断搜索；如果在实例中没有找到该属性，则继续搜索实例的`_proto_`属性所指向的**原型对象**，如果找到对应的属性，则返回该属性的值。这种搜索方式，是多个对象实例共享原型所保存的属性和方法的基本原理。

需要注意的是，虽然可以通过实例**访问**原型中的值，但却不能通过对象实例**重写**原型中的值。

当向实例中添加和原型中**同名的属性**时，原型中的属性会被屏蔽，但不会修改原型中的属性。

```javascript
function Person () {
}
Person.prototype.name = "Jack"
Person.prototype.age = 26
Person.prototype.job = "Software Engineer"
Person.prototype.sayName = function () {
	console.log(this.name)    
}
var person1 = new Person()
var person2 = new Person()
person1.name = "Greg"
console.log(person1.name) // Greg
console.log(person2.name) // Jack
```

以上示例中，在`person1`中添加了同名的 `name`属性，分别打印 `person1` 和 `person2`中的 `name`属性，发现 `person1`中输出的是在实例中添加的属性，而 `person2`中输出的则依然是原型中的属性。

使用`delete`操作符可以完全删除实例属性，从而让原型中的属性重新被访问到，如下示例：

```javascript
function Person () {
}
Person.prototype.name = "Jack"
Person.prototype.age = 26
Person.prototype.job = "Software Engineer"
Person.prototype.sayName = function () {
	console.log(this.name)    
}
var person1 = new Person()
var person2 = new Person()
person1.name = "Greg"
console.log(person1.name) // Greg
console.log(person2.name) // Jack
// delete person1中name属性
delete person1.name
console.log(person1.name) // Jack
```

`hasOwnProperty()`方法可以检测一个属性是存在于实例中，还是存在于原型中。该方法只在属性存在于对象实例中时，才返回 `true`，以下是一个示例：

```javascript
function Person () {
}
Person.prototype.name = "Jack"
Person.prototype.age = 26
Person.prototype.job = "Software Engineer"
Person.prototype.sayName = function () {
	console.log(this.name)    
}
var person1 = new Person()
var person2 = new Person()
console.log(person1.hasOwnProperty('name')) // false
person1.name = "Greg"
console.log(person1.name) // Greg
console.log(person1.hasOwnProperty('name')) // true
delete person1.name
console.log(person1.hasOwnProperty('name')) // false
```

下图展示了以上示例在不同情况下 实例与原型的关系：

![image-20201102201817655](F:\Study\JavaScript\Doc\imags\不同情况下实例与原型的对应关系.png)

### 5.4 重写原型

对象的原型是可以重写的，当重写原型后，实例和默认原型之间的联系被切断了。最直观的影响是：重写后的原型的 `constructor`不再指向原来的构造函数，即：`Person.prototype.constructor != Person`.

在给原型添加属性的时候，可以使用**对象字面量**的写法来**重写**整个原型对象，示例如下：

```javascript
function Person () {
}
Person.prototype = {
    name: "Nicholas",
    age: 29,
    job: 'Software Engineer',
    sayName: function () {
        console.log(this.name)
    }
}
```

以上的写法中，将对原型添加属性的方式改为使用对象字面量的方法，最终得到的基本相同，但有一点例外：

这样写，其原型的 `constructor`属性不再指向 `Person`,而是指向`Object()`.

实例的原型的`constructor`的指向也可以通过如下方式进行指定：

```javascript
function Person () {
}
Person.prototype = {
    constructor: Person,
    name: "Nicholas",
    age: 29,
    job: 'Software Engineer',
    sayName: function () {
        console.log(this.name)
    }
}
```

这样，可以让实例的原型的 `constructor`属性指向实例的构造函数。

> 原型的重写是 JavaScript中使用 `prototype`实现面向对象中继承的思想的重要依据。

### 5.5 原型模式的问题

原型模式省略了为构造函数传递初始化参数的这一环节，所有的实例在默认情况下都将取得相同的属性值。这在某种程度上会带来一些不便。

原型模式的属性是被很多实例所共享的，这种共享对于函数非常合适，但对于属性来说，问题就比较突出了。看如下示例：

```javascript
function Person () {
}
Person.prototype = {
    constructor: Person,
    name: "Nicholas",
    age: 29,
    job: 'Software Engineer',
    friends: ['Shelby','Court'],
    sayName: function () {
        console.log(this.name)
    }
}
var person1 = new Person()
var person2 = new Person()
person1.friends.push('Van')
person1.age = 20
console.log(person1.age) // 20
console.log(person2.age) // 29
console.log(person2.friends) // ['Shelby','Court','Van']
```

在对值是**基本类型**的属性来说，问题还不那么大，对单独某个实例的属性进行更改并不会影响到其他的实例（相当于添加同名属性，屏蔽了原型中的属性），但对于哪些**引用类型**的属性来说，就会比较麻烦。

如以上示例中，当 向`person1`的`friends`属性中添加一个成员 `Van`时，其变化也反应在实例 `person2`中，这是因为这两个实例共享了原型中的 `friends` 属性。

如果希望每个实例拥有自己的 `friends`属性，则需要对每个实例分别添加自己的 `friends`属性，对于添加引用类型的属性，这种操作是比较麻烦的（引用类型的数据基本都是拥有很多内部成员，写起来比较麻烦）。

```javascript
function Person () {
}
Person.prototype = {
    constructor: Person,
    name: "Nicholas",
    age: 29,
    job: 'Software Engineer',
    friends: ['Shelby','Court'],
    sayName: function () {
        console.log(this.name)
    }
}
var person1 = new Person()
var person2 = new Person()
person1.friends = ['A','B','C']
person2.friends = [
        { name:'X', age: 25},
        { name:'Y', age: 20},
        { name:'Z', age: 29},
        { name:'W', age: 15}
	]
```

实例一般都应该具有自己的属性，因此这种纯粹的原型模式在实际应用中仍不完美。使用构造函数模式和原型模式结合，能更好的解决这个问题。

## 六、组合使用构造函数模式和原型模式

在设计一个自定义类型的时候，我们可以预先将 **实例的自定义属性和方法**和**类型的公共属性和方法**分离开来，组合使用**构造函数模式**和**原型模式**，将**公共的属性和方法**添加到**原型**中，将**自定义的属性和方法**定义在构造函数中。

以下是一个示例：

```javascript
function Person (name, age, job, friends) {
    this.name = name
    this.age = age
    this.job = job
    this.friends = ['Shelby','Court']
}
Person.prototype = {
    constructor: Person,
    sayName: function () {
        console.log(this.name)
    }
}
var person1 = new Person('Nicholas', 26, 'Software Engineer')
var person2 = new Person('Greg', 33, 'Doctor')
person1.friends.push('Van') 
console.log(person1.friends) // Shelby,Court,Van
console.log(person2.friends) // Shelby,Court
person1.sayName() // Nicholas
person2.sayName() // Greg
console.log(person1.sayName === person2.sayName) //true
```

这种构造函数模式和原型模式组合使用的模式，是JavaScript中使用最广泛，认同度最高的一种创建自定义类型的模式，是用来定义自定义类型的一种默认的模式。

## 七、动态原型模式

动态原型模式是对组合模式的一种改进，组合模式将构造函数的定义和原型的定义分成两部分来写，这对熟悉面向对象语言的人开发人员来说，似乎有点怪怪的。

动态原型模式将原型的初始化过程封装到构造函数中

```javascript
function Person (name, age, job, friends) {
    this.name = name
    this.age = age
    this.job = job
    this.friends = ['Shelby','Court']
    if( typeof this.sayName != 'function') {
        Person.prototype.sayName = function () {
            console.log(this.name)
        }
        Person.prototype.others = {} // 其他属性
    }
}
```

以上这种方法，可以在创建一个实例时，动态的初始化原型，在第一次调用构造函数时，判断某个公共的方法是否存在，如果不存在，则初始化原型，添加所有公共属性和方法，等以后再调用实例时，这个条件不会触发，因此，原型只会被初始化一次。

## 八、其他模式

除了以上7中创建对象的模式外，还有 寄生构造函数模式 和 稳妥构造函数模式

## 十、下一次分享

> JavaScript面向对象程序设计（二）——继承



 

