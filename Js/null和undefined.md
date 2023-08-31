# null和undefined

JavaScript有5种简单数据类型(基本数据类型)和1种复杂书数据类型;

- 基本数据类型:`Undefined`,`nul`,`Boolean`,`Number`,`String` ;
- 复杂数据类型:`Object `;

以下比较一下两种表示空值的数据类型，`null` 和 `undefined`。

## 1.null

`null`表示一个特殊值,常用来描述**"空值"**。

对null执行typeof操作,结果返回字符串"object" ,null可以认为是一个特殊的对象值,含义是非对象。

从逻辑上看，`null` 表示一个空对象指针。

``` javascript
let dog = null;
console.log(typeof(dog)); // object
```

- 实际上,通常认为null是它自有类型的唯一一个成员,可以表示 **数字** , **字符串** ,**对象** 是 **无值的**。

- 如果定义的变量准备在将来用于**保存对象**,最好将该变量初始化为 `null`,而不是其他值.这样一来,只要直接检查`null`值就知道相应的变量是否已经保存了一个对象的引用。

```javascript
if(car!=null)
{
//对car对象执行某些操作
}
```

## 2.undefined

`undefined` 也被用来表示值的空缺,表示**未定义**,`undefined` 值表示更深层次的"空值".所有不存在的值，都表示为 `undefined`。

它是变量的一种取值,表明变量没有初始化,如果声明了一个变量,但未对其进行初始化时,则该变量的类型就是 `undefined`,如下:

```javascript
let a;
console.log(typeof(a));//undefined
```

不对变量进行初始化和将变量初始化为 `undefined` ,其结果是一致的,如下:

```javascript
let b;
console.log(typeof(b));
let c = undefined;
console.log(typeof(b)==typeof(c) ? true : false);//true
```

> 一般而言,不需要显式的把一个变量的值设置为 `undefined` ,该值的引入主要是为了区分 空对象指针 和 未经初始化的变量.

未定义的变量和定义但未初始化的变量的类型都是 `undefined`,

```javascript
let c; //未初始化
//d d未定义
console.log(typeof(c));//undefined
console.log(typeof(d));//undefined
console.log(c);//undefined
console.log(d);//出错
```

> 即便未初始化的变量会被自动赋予undefind值，但显式的初始化变量依然是更好的选择和习惯，如果能做到这一点，那么当typeof返回 `undefined` 时，我们就知道被检测的变量还没有被声明（即不存在），而不是尚未初化。

```javascript
let e = null;
//f不存在
console.log(typeof(e));//object
console.log(typeof(f));//undefined
```

## 3.null和undfined的联系和区别

- `undefined`值 是派生自 `null`值的，两者 在 `==` 下是相等的，但在 `===` （严格相等）下是不相等的。

```javascript
console.log(null == undefined ? true:false);//true
console.log(null === undefined ? true:false);//false
```

- `null` 是一个 object，是存在的， `undefined` 是未定义，表示的是不存在的某个东西。

## 4.参考资料

- 《JavaScript高级程序设计》
- 《JavaScript权威指南》