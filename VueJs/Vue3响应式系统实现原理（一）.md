# 一种JavaScript响应式系统实现

> 根据VueJs核心团队成员霍春阳《Vue.js设计与实现》第四章前三节整理而成

## 1.  响应式数据与副作用函数

### 1.1 **副作用函数**

会产生副作用的函数。

如下示例所示：

```js
function effect () {
    document.body.innerText = 'hello vue3!'
}
```

当effect函数执行时，会设置body的文本内容。但是，除了effect函数之外的任何函数都可以读取或者设置body的文本内容。也就是说，effect函数的执行，会直接或间接影响其他函数的执行，这时，我们说effect函数产生了副作用。

```js
// 此方法的结果就受effect函数的影响
function getInnerText () {
    return document.body.innerText
}
```

副作用很容易产生，例如一个函数修改了全局变量，这其实也是一个副作用，如下面代码所示：
```js
let globalValue = 1
function effect () {
    globalValue = 3 // 修改全局变量，产生副作用
}
```

### 1.2 响应式数据

理解了什么是副作用函数，再来说一说什么是响应式数据。

假设在一个副作用函数中读取了某个对象的属性：

```js
const obj = {
    text: 'hello vue2'
}
function effect () {
    document.body.innerText = obj.text // effect 函数的执行会读取obj.text
}
```

如上面代码所示，副作用函数effect会设置body元素的innerText属性，其值为obj.text。当obj.text的值改变时，我们希望副作用函数effect()会重新执行：

```js
obj.text = 'hello vue3' // 修改obj.text的值，同时希望副作用函数重新执行
```

这句代码修改了字段 obj.text 的值，我们希望当值变化后，副作用函数会重新执行，如果能实现这个目标，那么对象 obj 就是响应式数据。（**某数据改变时，依赖该数据的副作用函数会重新执行，该数据即为响应式数据**）

但是，从上面代码来看，我们还做不到这一点，因为obj是一个普通的对象，当我们修改它的值时，除了值本身发生变化外，不会有任何其他反应。 

## 2. 响应式数据的基本实现

### 2.1  如何让 obj 变为响应式数据？

通过观察，我们可以发现 2 点线索：

- 当副作用函数effect()执行时，会触发字段 obj.text 的**读取**操作；
- 当修改 `obj.text` 的值时，会触发字段 `obj.text` 的**设置**操作；

​	如果我们能拦截一个对象的**读取**和**设置**操作，事情就变得简单了。

- 当读取`obj.text` 时，我们可以把副作用函数effect()存储到一个“桶”里；

  

![image-20220627151018125](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627151018125.png)



- 当设置`obj.text`时，再把effect()副作用函数从 “桶” 里取出，并执行；

  

<img src="https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627151637370.png" alt="image-20220627151637370" style="zoom:80%;" />

### 2.2 如何拦截对象属性的读取和设置操作

在 es5 及之前，只能通过 `Obj.defineProperty` 来实现，这也是vue2中所采用的方式。

es6以后，可以使用代理对象 `Proxy` 来实现，这是vue3中的所采用的方式。

**实现步骤：**

- 创建一个用于**存储副作用函数**的容器（或者通俗点说，一个桶，vue3里用了bucket这个词，以后都称为桶）；
- 定义一个普通对象`obj`作为原始数据；
- 创建 `obj` 对象的代理作为响应式数据，分别设置 `get` 和 `set` 拦截函数，用于拦截读和写操作；
- 当在effect副作用函数中执行响应式数据读操作时，将effect()副作用函数存到桶里；
- 当对响应式数据进行写操作时，先更新原始数据，再从桶中取出依赖了响应式数据的函数，进行执行；

接下来，根据以上思路，使用`Proxy` 实现一下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="app">
    <p id="title"></p>
    <button onclick="changeObj()">修改响应式数据</button>
  </div>
</body>
</html>
<script>
  // 1. 创建一个存储副作用函数的桶      
  const bucket = new Set()
  // 2. 一个普通的对象
  const obj = {
    text: 'hello vue2'
  }
  // 3. 普通对象的代理，一个简单的响应式数据
  const objProxy = new Proxy(obj, {
    get (target, key) {
      bucket.add(effect)
      return target[key]
    },
    set (target, key, newValue) {
      target[key] = newValue
      bucket.forEach((fn) => {
        fn()
      })
      return true
    }
  })

  // 4. 副作用函数 effect(), 给p标签设置值
  const effect = function () {
    document.getElementById('title').innerText = objProxy.text
  }

  // 5. 改变代理对象的元素值
  const changeObj = function () {
    objProxy.text = 'hello vue3!!!!!!'
  }

  // 首次进入时，给p标签设置值
  effect()
</script>
```

![响应式-show-1](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/响应式-show-1.gif)



### 2.3 缺陷

- 副作用函数名字叫`effect` ，是硬编码的，假如叫其他的名字，就无法正确执行了；

## 3. 完善的响应式系统

### 3.1 如何存储一个任意命名（甚至匿名）的副作用函数？

需要提供一个用来注册副作用函数的机制，如以下代码所示：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="app">
    <p id="title"></p>
    <button onclick="changeObj()">修改响应式数据</button>
  </div>
</body>
</html>
<script>
let activeEffect
function effectRegister (fn) {
  activeEffect = fn
  fn()
}

const bucket = new Set()
const obj = {
  text: 'hello vue2'
}
const objProxy = new Proxy(obj, {
  get (target, key) {
    if (activeEffect) {
      bucket.add(activeEffect)
    }
    return target[key]
  },
  set (target, key, newValue) {
    target[key] = newValue
    bucket.forEach((fn) => {
      fn()
    })
    return true
  }
})
const changeObj = function () {
  objProxy.text = 'hello vue3!!!!!!'
}

effectRegister(() => {
  document.getElementById('title').innerText = objProxy.text
})
</script>

```

以上方式，通过定义一个全局变量`activeEffect`，用来存储(匿名)副作用函数。并提供一个注册副作用函数的**注册函数** `effectRegister`，该函数是一个高阶函数，接受一个函数作为参数，保存并执行该函数。将(匿名)副作用函数保存到`activeEffect` 中，当(匿名)副作用函数执行时，触发响应式数据的读操作，此时将`activeEffect` 存入副作用函数桶中。



**缺陷:**

以上方案成功解决了**匿名的函数**的保存问题，但仍存在一个严重问题：

> 响应式对象内的不同属性和不同副作用函数的对应问题

下面给响应式数据添加一个属性，以上代码微调为如下：

```js
// ... 省略未改变代码

const setObj = function (key, value) {
  objProxy[key] = value
}

const changeObj = function () {
  setObj('text', 'hello vue3!!!!!!')
  setObj('notExist', 'this key is not exist')
  console.log(1, bucket)
}

effectRegister(() => {
  console.log('执行')
  document.getElementById('title').innerText = objProxy.text
})
```

执行结果为：


![image-20220627190632726](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627190632726.png)

可以看到，此时副作用函数与`obj.notExist` 属性并未建立响应关系，但当给 notExist  赋值时，副作用函数也执行了，这显然不对了。

理想情况应该是，a属性与aFunc建立响应式关系，b属性与bFunc建立响应式联系，则 a 改变时，仅 aFunc函数触发执行，b改变时，仅bFunc触发执行。

### 3.2 如何解决响应式对象内的不同属性和不同副作用函数的对应问题？

**问题分析：**

导致该问题的根本原因是，我们**没有在副作用函数与被操作的目标字段之间建立明确的联系**。

例如，但读取属性值时，无论**读取**到哪一个属性，其实都一样，都会把副作用函数收集到“桶”里；无论**设置**的是哪一个属性，也会把“桶”里面的副作用函数取出并执行。响应数据属性和副作用函数之间没有明确的联系。

解决方案很简单，只需要在副作用函数与被操作的字段之间建立联系即可。

要实现不同属性值与副作用函数对应，`Set`类型的数据作为桶已经明显不合适了。

> 科普下es6几大数据类型：
>
> **Set**：ES6 提供了新的数据结构 Set。它类似于数组，但是成员的值都是唯一的，没有重复的值。
>
> **WeakSet**：WeakSet 结构与 Set 类似，也是不重复的值的集合。但是，它与 Set 有两个区别。首先，WeakSet 的成员只能是对象，而不能是其他类型的值。其次，WeakSet 中的对象都是弱引用，即垃圾回收机制不考虑 WeakSet 对该对象的引用，也就是说，如果其他对象都不再引用该对象，那么垃圾回收机制会自动回收该对象所占用的内存，不考虑该对象还存在于 WeakSet 之中。
>
> **Map**：它类似于对象，也是键值对的集合，但是“键”的范围不限于字符串，各种类型的值（包括对象）都可以当作键。也就是说，Object 结构提供了“字符串—值”的对应，Map 结构提供了“值—值”的对应，是一种更完善的 Hash 结构实现。
>
> **WeakMap**：`WeakMap`结构与`Map`结构类似，也是用于生成键值对的集合。`WeakMap`与`Map`的区别有两点。
>
> 首先，`WeakMap`只接受对象作为键名（`null`除外），不接受其他类型的值作为键名。其次，`WeakMap`的键名所指向的对象，不计入垃圾回收机制。

**新的方案：**

首先，来看下如下代码：

```js
  effectRegister(function effectFn () {
    document.getElementById('title').innerText = objProxy.text
  })
```

这段代码存在三个角色：

- 被读取的响应式数据 `objProxy`;
- 被读取的响应式数据的属性名 `text`;
- 使用`effectRegister` 函数注册的副作用函数 `effectFn`;

如果使用 `target` 来表示一个代理对象所代理的原始对象，用`key` 来表示被操作的字段名，用`effectFn` 来表示被注册的副作用函数，那么可以为这三个角色建立如下关系：



![image-20220627213637491](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627213637491.png)

这是一种树型结构，下面举几个例子进行补充说明：

1. 如果有2个副作用函数同时读取同一个对象的属性值：

   ```js
   effectRegister(function effectFn1 () {
       obj.text
   })
   
   effectRegister(function effectFn2 () {
       obj.text
   })
   ```

​			那么这三者关系如下：


![image-20220627214317647](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627214317647.png)

2. 如果一个副作用函数中读取了同一个对象的 2 个不同属性：

   ```js
   effectRegister(function effectFn () {
       obj.text1
       obj.text2
   }
   ```

那么这三者关系如下：

![image-20220627214620629](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627214620629.png)

3. 如果 2 个不同的副作用函数中读取了 2 个不同对象的不同属性：

   ```js
   effectRegister(function effectFn1 () {
       obj1.text1
   })
   
   effectRegister(function effectFn2 () {
       obj2.text2
   })
   ```

   那么这三者关系如下：

   ![image-20220627215041694](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220627215041694.png)

总之，这其实就是一个树形数据结构。这个联系建立起来后，就可以解决前文提到的问题。上文中，如果我们设置了`obj2.text2`的值，就只会导致 `effectFn2` 函数重新执行，并不会导致 `effectFn1` 函数重新执行。

接下来，需要重新设计这个桶。根据对以上几种情况的分析，我们可以总结一下这种数据结构的模型：

![image-20220628103144724](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220628103144724.png)

首先，使用`WeakMap` 代替 `Set` 来作为桶，将 原始对象 `target` 作为`WeakMap` 的key，使用 `Map` 作为value。在`Map` 中又以属性值 作为 key值，使用 `Set` 存储key 对应的副作用函数 `effectFn`，然后修改  `get/set` 拦截器代码。

以下，我们将桶命名为 `bucket`, 每个对象的 属性-副作用函数 存储块 命名为 `depsMap`，将depsMap中的value部分（Set类型，存储副作用函数）命名为 `deps`（与vue3响应式实现源码命名保持一致），根据数据结构图，重新编写`get/set` 拦截器代码：

js:

```js
// WeakMap 类型的桶
const bucket = new WeakMap()
// 初始值为undefined
let activeEffect
// 定义2个对象
const obj_1 = {
    text1: 'hello vue',
    text2: 'hello jquery'
}

const obj_2 = {
    text1: 'hello react',
    text2: 'hello angular'
}

const objProxy1 = new Proxy(obj_1, {
    get (target, key) {
        // 如果没有注册的副作用函数，直接返回key对应的value值
        if (!activeEffect) {
            return target[key]
        }
        
        // 获取桶内 target 对象对应的 depsMap
        let depsMap = bucket.get(target)
        // 如果该对象还没有depsMap,这新建一个
        if (!depsMap) {
            depsMap = new Map()
            bucket.set(target, depsMap)
        }
        
        // 从depsMap中取出属性对应的副作用函数集合
        let deps = depsMap.get(key)
        // 同理，若不存在，则创建
        if (!deps) {
            deps = new Set()
            depsMap.set(key, deps)
        }
        deps.add(activeEffect)
        return target[key]
    },
    set (target,key, newValue) {
        // 给target的key设置新的value
        target[key] = newValue
        
        // 取出该target对应的depsMap
        let depsMap = bucket.get(target)
        // depsMap 不存在或为空直接返回
        if (!depsMap) {
            return
        }
        
        // 取出deps
        let deps = depsMap.get(key)
        // deps 不存在或为空直接返回
        if (!deps) {
            return
        }
        // 依次执行对应副作用函数
        deps.forEach((fn) => {
            fn()
        })
        return true
    }
})

const objProxy2 = new Proxy(obj_2, {
    ...
})

const effectRegister = function (fn) {
    activeEffect = fn
    fn()
}

```

**演示：**

![响应式-show-3](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/响应式-show-3.gif)

### 3.3 思考：为什么桶的结构使用WeakMap而不是Map?

看一段代码：

```js
const map = new Map()
const weakmap = new WeakMap()
;(function () {
    const foo = { foo: 1}
    const bar = { bar: 2}
    map.set(foo, 1)
    weakmap.set(bar, 2)
})();
console.log(map)
console.log(weakmap)
```

思考，打印结果分别是啥？

**注意**：在浏览器环境下，console打印结果表现出异步行为。

![image-20220628140308855](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220628140308855.png)

**原因分析：**

Map对key是强引用，当立即执行函数结束后，foo仍被map引用，因此map.keys()可以成功打印出key值，而WeakMap对`key`是弱引用，立即执行函数结束后,bar 失去引用，被垃圾回收器回收掉，因此weakmap.keys()无法取出key值。

**结论：**

WeakMap经常存储那些只有当key值所引用的对象存在（没有被垃圾回收）时才有价值的信息，例如示例中的target，如果target对象没有任何引用了，说明用户测不需要它了，这是垃圾回收器会完成回收任务。

如果使用Map来代替WeakMap，那即使用户侧没有任何对target的引用，这个target也不会被回收，最终可能导致内存溢出。

## 4. 总结

以上，我们实现了一个简单的响应系统，核心思路是通过代理来拦截响应数据的读和写的操作，将与对象属性相关的副作用函数进行存储，当对象属性变化时，同步执行相关联的副作用函数，达到响应式的效果。

其实现中混合使用了代理模式和观察者模式。

## 5. 参考资料

- 霍春阳《Vue.js设计与实现》
- [JS中15种常见设计模式](https://www.cnblogs.com/imwtr/p/9451129.html#o5)

## 附：程序源码

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="app">
    <p id="title_1"></p>
    <p id="title_2"></p>
    <button onclick="setObj_1()">修改响应式数据1</button>
    <button onclick="setObj_2()">修改响应式数据2</button>
  </div>
</body>
</html>
<script>
  let activeEffect
  let count = 3
  let version = 3
  const bucket = new WeakMap()
  const obj_1 = {
    text1: 'hello vue2',
    text2: 'hello jq2'
  }

  const obj_2 = {
    text2: 'hello react16',
    text1: 'hello ng2'
  }

  const getProxy = function (obj) {
    return new Proxy(obj, {
      get (target, key) {
        track(target, key)
        return target[key]
      },
      set (target, key, newValue) {
        trigger(target, key, newValue)
        return true
      }
    })
  }

  const track = function (target, key) {
    // 如果无副作用函数，则直接返回原始对象值
    if (!activeEffect) {
      return
    }

    // 以该代理对象的原始对象作为key值，获取depsMap  (属性和副作用函数之间的对应关系 key --> effectFn)
    let depsMap = bucket.get(target)
    // 如果不存在depsMap,则新建一个Map与target关联起来
    if (!depsMap) {
      depsMap = new Map()
      bucket.set(target, depsMap)
    }
    // 根据key从depsMap中获取对应的deps,deps是一个Set
    let deps = depsMap.get(key)
    // 如果不存在，则新建Set,并与key关联
    if (!deps) {
      deps = new Set()
      depsMap.set(key, deps)
    }
    // 最后将当前活跃的副作用函数添加到桶里
    deps.add(activeEffect)
  }

  const trigger = function (target, key, newValue) {
    target[key] = newValue
    let depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }
    let deps = depsMap.get(key)
    if (!deps) {
      return
    }

    deps.forEach(fn => {
      fn()
    })
  }

  const objProxy1 = getProxy(obj_1)
  const objProxy2 = getProxy(obj_2)

  const effectRegister = function (fn) {
    activeEffect = fn
    fn()
  }

  const setObj_1 = function () {
    objProxy1.text1 = `hello, vue${count++}`
    console.log('bucket-1', bucket)
  }

  const setObj_2 = function () {
    objProxy2.text2 = `hello, react${version++}`
    console.log('bucket-2', bucket)
  }

  console.log('bucket-init', bucket)
  effectRegister(function effectFn1 () {
    document.getElementById('title_1').innerText = objProxy1.text1
  })

  effectRegister(function effectFn2 () {
    document.getElementById('title_2').innerText = objProxy2.text2
  })
</script>
```

