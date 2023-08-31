# Vue3响应式系统实现原理（二）

> 本文根据VueJs核心团队成员霍春阳《Vue.js设计与实现》第四章整理，推荐直接购买正版书籍系统学习
>
> 本文主要内容：
>
> - 分支切换与cleanup
> - 嵌套的effect与effect栈
> - 避免无限递归循环
> - 调度执行

## 1. 分支切换与cleanup

### 1.1 分支切换

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
    </div>
    <button onclick="handleClick()">click</button>
    <button onclick="switchBranch()">切换分支</button>
</body>
</html>
<script>
let activeEffect
const bucket = new WeakMap()
const obj = {
    ok: false,
    text: 'hello vue3'
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
   ...
}

const trigger = function (target, key, newValue) {
   ...
}

const objProxy = getProxy(obj)
const effectRegister = (fn) => {
    activeEffect = fn
    fn()
}

effectRegister(() => {
    document.getElementById('app').innerHTML = objProxy.ok ? objProxy.text : 'no'
    console.log('effect 执行')
})

let count = 0
const handleClick = () => {
    objProxy.text = `上班的第${count++}天`
    console.log('bucket', bucket)
}

const switchBranch = () => {
    objProxy.ok = !objProxy.ok
}
</script>
```

如上代码所示，在副作用函数中存在一个三目表达式 `document.getElementById('app').innerHTML = objProxy.ok ? objProxy.text : 'no'` 。在执行副作用函数时，会根据 `objProxy.ok ` 的值判断执行某一个分支逻辑。当 `objProxy.ok ` 的值改变时，代码会执行不同的分支逻辑，这就是所谓的 **分支切换**。

我们现有的响应式系统，切换分支会带来异常的响应式效果。分别看我们将 `objProxy.ok` 的值由 `false` 切换为 `true` 再切回 `false` 时，我们响应式对象与副作用函数的依赖关系：

- objProxy.ok = false

![image-20220801222327175](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220801222327175.png)

- objProxy.ok = true

![image-20220801222453955](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220801222453955.png)

- objProxy.ok = false

![image-20220801222602463](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220801222602463.png)

对比objProxy.ok 由 `false` 切换为 `true` 再切换为 `false` 的过程中，bucket中响应式对象的依赖关系可以发现，当由false变为true时，bucket中分别存储了obj.ok 和 obj.text 的副作用函数，但当objProxy.ok 由true再变为false时，bucket中仍然存储了obj.text对应的副作用函数。而正常的情况应该是当分支切换后，对应的依赖关系也会切换。

这就说明，这时候产生了遗留的副作用函数。遗留的副作用函数会导致不必要的更新。

如上面的示例中，当分支切换为objProxy.ok 为false的分支后，副作用函数并未依赖objProxy.text，但其与text的依赖关系仍遗留在bucket中。当objProxy.text值改变时，副作用函数也会执行，这显然是不必要的，因为此时副作用函数不依赖于objProxy.text的值了。

**接下来将探讨如何解决这个问题？**

### 1.2 cleanup

为了解决分支切换造成的 “副作用函数遗留” 的问题，我们需要改进原有的实现方案。具体的思路为：

- 每次副作用函数执行时，可以先把它从所有与之关联的依赖集合中删除；
- 当副作用函数执行完毕后，响应式数据会与副作用函数之间建立新的依赖关系，而分支切换后，与副作用函数没有依赖关系的响应式数据则不会再建立依赖，这样副作用函数遗留的问题就解决了；

下面来看新方案的具体实现：

（1）改进副作用函数注册方法 `effectRegister()`：

```js
 function effectRegister (fn) {
    // 改进副作用函数
    const effectFn = () => {
      console.log('effectFn')
      // 副作用函数执行前，先把副作用函数从所有与之关联的依赖集合中删除 
      cleanup(effectFn)
      // 暂存当前活跃的副作用  
      activeEffect = effectFn
      fn()
    }
	// 定义一个数组，存储与某一个副作用函数关联的依赖
    effectFn.deps = []
    // 执行这个副作用函数 
    effectFn()
 }
```

（2）`cleanup()` 方法的具体实现：

```js
function cleanup (effectFn) {
    console.log('cleanup')
    for (let i = 0; i < effectFn.deps.length; i++) {
        const deps = effectFn.deps[i] // 这个deps就是与响应式数据的key值对应的那个 Set 集合
        deps.delete(effectFn) // 从 Set 集合中把当前这个副作用函数移除
    }
    effectFn.deps.length = 0
}
```

 （3）`track()` 方法改进：

```js
function track (target, key) {
    if (!activeEffect) {
      return
    }

    let depsMap = bucket.get(target)
    if (!depsMap) {
      depsMap = new Map()
      bucket.set(target, depsMap)
    }

    let deps = depsMap.get(key)
    if (!deps) {
      deps = new Set()
      depsMap.set(key, deps)
    }

    deps.add(activeEffect)
    // 增加这一项：deps 存储了 副作用函数，副作用函数也存储了 deps， 依赖和副作用函数之间建立了双向的联系
    activeEffect.deps.push(deps)
}
```

下面来看改进后的效果，看是否解决副作用函数遗留的问题：

- objProxy.ok = true：

![image-20220807141745814](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220807141745814.png)

- objProxy.ok = true； =>  objProxy.ok = false

![image-20220807141918642](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220807141918642.png)

前后对比之下，我们发现切换分支后， objProxy.text中遗留的副作用函数消失了。

改进后的方案成功解决了 “副作用函数遗留” 的问题。

等等，新的问题出现了.......?

![GIF 2022-8-7 14-25-17](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/GIF%202022-8-7%2014-25-17.gif)

呃，陷入无线循环了。。。。

### 1.3 无限循环

问题出在 `trigger()` 方法中，先来看 `trigger()` 的代码：

```js
function trigger (target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }

    const deps = depsMap.get(key)
    deps && deps.forEach(effectFn => {
      effectFn()
    })
}
```

问题就在这段代码：

```js
 deps && deps.forEach(effectFn => {
      effectFn()
 })
```

为什么这里有问题？ 先来看一个示例：

```js
var a = 'a'
var deps = new Set()
deps.add(a)
deps.forEach((item) => {
    console.log(`item: ${item}, deps: ${deps}`)
})
```

看看结果？

set循环内加上这2行：

```js
deps.delete(a)
deps.add(a)
```

再看看结果？

一个奇怪的知识点出现了：

> 在调用循环遍历 Set 集合时，如果一个值已经被访问过了，但该值被删除，并重新添加到集合，如果此时循环遍历没有结束，那该值会被**重新访问**。
>
> 参考资料：[ECMAScript Language Specification](https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set.prototype.foreach)
>
> Each value is normally visited only once. However, a value will be revisited if it is deleted after it has been visited and then re-added before the `forEach` call completes. Values that are deleted after the call to `forEach` begins and before being visited are not visited unless the value is added again before the `forEach` call completes. New values added after the call to `forEach` begins are visited.
>
> ![image-20220807151609364](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220807151609364.png)
>
> 提示：语言规范说的是forEach时是这样的，我实测 for of 遍历Set会有同样的问题。

回到我们 `trigger` 方法：

 ```js
deps && deps.forEach(effectFn => {
      effectFn()
 })
 ```

看看执行 `effectFn()` 做了啥？

- 第一步：cleanup(effectFn)

```js
deps.delete(effectFn) // 删除响应式依赖里的副作用函数 effectFn
```

- 第三步：fn()

```js
// fn()会重新执行函数，重新收集依赖
// track() 方法中
deps.add(activeEffect) // 将副作用函数activeEffect添加到响应式依赖中
```

**解决方法：**

构造另外一个Set集合，并遍历它（深拷贝一份）：

```js
var a = 'a'
var deps = new Set()
deps.add(a)
var newSet = new Set(deps)
newSet.forEach((item) => {
    console.log(`item: ${item}, deps: ${deps}, newSet: ${newSet}`) 
    deps.delete(item)
	deps.add(item)
})
```

现在，用这种方法对 `trigger()` 方法进行改进：

```js
function trigger (target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }

    const deps = depsMap.get(key)
    const depsToRun = new Set(deps)
    depsToRun && depsToRun.forEach(effectFn => {
      effectFn()
    })
}
```

看下效果：

![show-9](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/show-9.gif)

至此，分支切换带来的 “副作用函数遗留问题” 圆满解决了。

## 2. 嵌套的effect与effect栈

### 2.1 effect 嵌套

effect是可以发生嵌套的，如：

```js
 effectRegister(() => {
    console.log(objProxy.count)
      effectRegister(() => {
        console.log(objProxy.ok)
          effectRegister(() => {
            console.log(objProxy.text)
          })
      })
 })
```

effect嵌套的场景在Vue.js中常常出现，如：Vue中的渲染函数（render）就是在一个effect中执行的：

以下是一个组件 Foo 的渲染。

```js
const Foo = {
    render() {
        return ....
    }
}

effectRegister(() => {
  Foo.render() // 响应式数据变化时会触发更新，重新rennder()
})
```

组件是可嵌套的，当在 Foo 中嵌套了 Bar 组件时，

```js
const Bar = {
    render () {
        return ...
    }
}
const Foo = {
    render () {
        return <Bar />
    }
}

effectRegister(() => {
    Foo.render()
    effect(() => {
        Bar.render()
	})
})
```

然而，我们当前的实现的响应式系统，并不支持effect嵌套，使用以下程序进行测试：

```js
<body>
  <div id="app"></div>
  <button onclick="changeFoo()">changeFoo</button>
  <button onclick="changeBar()">changeBar</button>
</body> 
<script>
...
const obj = {
    foo: 0,
    bar: 0
  }
  let temp1, temp2

  const objProxy = getProxy(obj)

  effectRegister(function FnA () {
    console.log('FnA 执行')
    effectRegister(function FnB () {
      console.log('FnB 执行')
      temp2 = objProxy.bar
    })
    temp1 = objProxy.foo
  })

  function changeFoo () {
    objProxy.foo++   
  }

  function changeBar () {
    objProxy.bar++
  }
</script>
```

以上代码中，FnA中嵌套了FnB，FnA的执行会导致FnB的执行，FnB读取了objProxy.bar的值，FnA读取了objProxy.foo的值，理想情况下，他们之间的依赖关系应该为：

```js
obj
└----foo
	   └--FnA	
└----bar 
	   └--FnB
```

那么，如果修改 objProxy.foo 的值，则FnA会触发执行，而 FnB 嵌套在FnA中，所以FnB也会执行。 理想情况下，输出结果应为：

```js
FnA 执行
FnB 执行
```

实际输出结果：

```js
FnB 执行
```

实际输出的结果显然不符合预期。改变 objProxy.foo 的值后，objProxy.foo 的副作用函数FnA竟然未执行。

这个现象说明， objProxy.foo 的副作用函数不为 FnA，而为FnB。为什么出现这种情况？

接下来我们分别执行 bucket中 objProxy.foo 和 objProxy.bar 对应的副作用函数，对此一探究竟：

```js
bucket.get(obj).get('foo').forEach(fn => fn()) // FnB执行
```

通过读取存储的副作用函数，可以发现，此时  objProxy.foo 的副作用函数竟然是 FnB，而不是 FnA，很显然问题必然出现在 activeEffect 上。

让我们结合代码来看：

```js
let activeEffect
function effectRegister (fn) {
    const effectFn = () => {
      cleanup(effectFn)
      // 这里，当调用副作用函数注册时，会将副作用函数赋值给 activeEffect,此时 effectFn 为FnA的effectFn
      activeEffect = effectFn
      // 这里，会执行FnA，然后在FnA内部会尽力如下步骤：
      // (1) console.log('FnA 执行')
	  // (2) 注册FnB, FnB内部有如下步骤
      //  	(2.1) cleanup(effectFn) 此时 effectFn 为 FnB 的effectFn
      //    (2.2) activeEffect = effectFn 此时 activeEffect 为 FnB的effectFn
      //    (2.3) 执行 FnB, 打印 ‘FnB 执行’，并通过track，将此时的 activeEffect 添加到 bar 的 deps
      // (3) temp1 = objProxy.foo 这一步，会执行 foo 的track，将 此时的 activeEffect 添加到 foo 的 deps,此时问题出现了，在2.2，activeEffect 已经变成了 FnB的effectFn，所以，此时 foo 的副作用函数实际上并不是 FnA,而是 FnB。
      fn()
    }

    effectFn.deps = []
    effectFn()
}
```

我们用全局变量 activeEffect 来存储通过 effectRegister 函数注册的副作用函数，这意味着同一时间 activeEffect 所存储的副作用函数只能有一个，当副作用函数发生嵌套时，内层副作用函数的执行会覆盖外层的 activeEffect 的值，并且无法恢复，如果在 activeEffect 的值被覆盖后，再有响应式数据进行依赖收集，此时它收集到的实际上是内层的副作用函数，这就是问题所在。

### 2.2 effect栈

为了解决这个问题，我们需要一个副作用函数栈 effectStack，在副作用函数执行时，将当前的副作用函数压入栈中，待副作用函数执行完毕后，将其从栈中弹出，并始终让 activeEffect 指向栈顶的副作用函数。这样就能做到一个响应式数据只会收集直接读取其值的副作用函数，而不会出现互相影响的情况。

接下来看具体实现：

```js
let activeEffect
const effectStack = []
function effectRegister (fn) {
    const effectFn = () => {
      cleanup(effectFn)
      activeEffect = effectFn
      // 将当前副作用函数压入栈中
      effectStack.push(activeEffect)
      // 执行 fn(),进行依赖收集（track）  
      fn()
      // 执行完毕，弹出  
      effectStack.pop()
      // 让 activeEffect 始终指向栈顶的副作用函数，若effectStack中无值，则activeEffect还原为undefined
      activeEffect = effectStack[effectStack.length -1]
    }

    effectFn.deps = []
    effectFn()
}
```

定义一个 effectStack 数组用来模拟栈，activeEffect 仍然指向当前正在执行的副作用函数。不同的是，当前执行的副作用函数会被压入栈顶，这样当副作用函数发生嵌套时，栈底存储的就是外层的副作用函数，而栈顶存储的则是内层副作用函数。当内层副作用函数执行完毕时，会从栈顶弹出，此时再将 activeEffect 指向新的栈顶函数，activeEffect 也就自动的还原为外层的副作用函数了。

如此一来，响应式数据就只会收集直接读取其值的副作用函数作为依赖，从而避免发生错乱。

## 3. 避免无限递归循环

下面讨论第三个问题：避免无限递归循环。

以上的方案，假如在同一个副作用函数中同时读取和设置某个响应式数据的值，会产生什么结果呢？

```js
effectRegister(() => {
    objProxy.count = objProxy.count + 1
})
```

![image-20220807161150143](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220807161150143.png)

栈溢出了。

原因很明显了，首先读取objProxy.count，并把副作用函数存储到依赖中，紧接着又修改objProxy.count，把副作用函数取出来执行，其结果就是，副作用函数在自己内部递归调用自己，栈就溢出了。

**解决方法：**

通过分析可以发现，读取和设置操作都是在同一个副作用函数中进行的，此时无论track收集的**副作用函数**还是trigger要触发执行的**副作用函数**，其实都是同一个，也就是当前的 `activeEffect`。因此，可以增加**守卫条件**，当 trigger 要触发执行的副作用函数就是当前正在执行的副作用函数（activeEffect）时，则不触发执行。

```js
  function trigger (target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }

    const deps = depsMap.get(key)
    const depsToRun = new Set(deps)
     deps && deps.forEach((effectFn) => {
      (effectFn !== activeEffect) && depsToRun.add(effectFn)
    })
    depsToRun && depsToRun.forEach(effectFn => effectFn())
  }
```

![image-20220807162526094](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20220807162526094.png)



## 4. 调度执行

### 4.1 可调度

可调度性是响应式系统非常重要的特性。首先，我们要明白什么是可调度性？

所谓可调度，指的是当trigger动作触发副作用函数重新执行时，有能力决定副作用函数**执行的时机、次数、方式**。

首先看以下代码：

```js
const obj = {
    count: 0
}
const objProxy = getProxy(obj)
effectRegister(() => {
    console.log(objProxy.count)
})

objProxy.count++
console.log('结束了')
```

代码执行后输出结果为：

```js
0
1
结束了
```

假设现在变更输出顺序为：

```js
0
结束了
1
```

该如何做？

一种方法是把 objProxy.count++ 和 console.log('结束了')调换位置；

如果在不动代码执行顺序的情况下，该如何实现呢？

此时就需要我们的响应式系统支持**调度**了。

### 4.2 调度器

对 effeectRegister 函数进行改造，给其传入一个 options 选项参数，允许用户为副作用函数指定调度器。

如下代码所示：用户在调用 effectRegister 函数注册副作用函数时，可以传递第二个参数 options。options 中允许指定一个 scheduler 调度函数。

```js
effectRegister(() => {
    console.log(objProxy.count)
}, {
    // 调度器函数，fn是副作用函数
    scheduler (fn) {
        // 控制执行时机
    }
})
```

同时，在 effectRegister 函数中我们需要把 options 挂载到副作用函数上：

```js
function effectRegister (fn, options = {}) {
    const effectFn = () => {
      cleanup(effectFn)
      activeEffect = effectFn
      // 将当前副作用函数压入栈中
      effectStack.push(activeEffect)
      // 执行 fn(),进行依赖收集（track）  
      fn()
      // 执行完毕，弹出  
      effectStack.pop()
      // 让 activeEffect 始终指向栈顶的副作用函数，若effectStack中无值，则activeEffect还原为undefined
      activeEffect = effectStack[effectStack.length -1]
    }

    // 将 options 挂载到effectFn上
    effectFn.options = options
    effectFn.deps = []
    effectFn()
}
```

有了调度函数，我们在 trigger 函数中触发副作用函数重新执行时，就可以直接调用用户传进来的调度函数，从而把控制权交给用户：

```js
function trigger (target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }

    const deps = depsMap.get(key)
    const depsToRun = new Set(deps)
     deps && deps.forEach((effectFn) => {
      (effectFn !== activeEffect) && depsToRun.add(effectFn)
    })
    depsToRun && depsToRun.forEach(effectFn => {
        // 存在调度器时，则调用调度器，由调度器决定执行 effectFn 的时机
        if (effectFn.options.scheduler) {
            effectFn.options.scheduler(effectFn)
        } else {
            effectFn(
            )
        }
    })
}
```

在 trigger 中触发执行 副作用函数时，先判断副作用函数是否存在调度器，如果存在，直接调用调度器，并将副作用函数作为参数传递给调度器，由用户控制如何执行副作用函数，否则就保留之前的行为，直接执行副作用函数。

下面使用改进后的方案来控制4.1中程序的执行顺序：

```js
effectRegister(() => {
      console.log(objProxy.count)
	}, {
      scheduler (fn) {
        setTimeout(() => fn(), 100)
     }
})
objProxy.count++
console.log('结束了')
```

执行结果：
```js
0
结束了
1
```

以上讨论了副作用函数执行时机的简单控制方案。除了控制副作用函数的**执行顺序**，通过调度器还可以控制副作用函数的**执行次数**。这一点也是尤为重要的，像 vue.js 连续修改多次响应式数据，实际上只会触发一次更新操作，其实现思想类似。

以如下代码为例：

```js
effectRegister(() => {
      console.log(objProxy.count)
})
objProxy.count++
objProxy.count++
```

输出结果：

```js
0
1
2
```

在这个示例中，objProxy.count 经过2次自增，值从 0 到 2，打印结果为 0，1，2。如果我们只关心 objProxy.count自增后的结果而不关心过程，那么执行3次打印操作是多余的，我们期望只打印2次，不包含过渡状态的打印结果，即：

```js
0
2
```

基于调度器，可以通过控制实现这个功能：
```js
// 定义一个任务队列
const jobQueue = new Set()
// 使用 Promise.resolve() 创建一个 promise 实例，用它将一个任务添加到微任务队列
const p = Promise.resolve()

// 一个标志，代表是否正在刷新队列
let isFlushing = false
function flushJob () {
    // 如果队列正在刷新，则什么都不做
    if (isFlushing) {
        return
    }

    // 队列未刷新时，改变状态标志为正在刷新
    isFlushing = true
    // 在微任务队列中刷新 jobQueue 队列
    p.then(() => {
        jobQueue.forEach(job => job())
    }).finally(() => {
        // 结束后重置 isFlushing为未刷新状态
        isFlushing = false
    })
}

effectRegister(() => {
    console.log(objProxy.count)
}, {
    scheduler (fn) {
        // 每次调度时，将副作用函数添加到任务队列中
        jobQueue.add(fn)
        // 调用 flushJob 刷新队列
        flushJob()
    }
})

objProxy.count++
objProxy.count++
```

以上代码，首先定义一个 Set 类型的任务队列 jobQueue，以便在添加副作用函数时，可以进行去重。

紧接着看调度器的实现，在每次调度执行时，先把当前副作用函数添加到 jobQueue 队列中，再调用 flushJob 方法刷新队列。flushJob 中通过 isFlushing 标志判断是否执行，只有当其为 false 时才执行，一旦开始执行，立即把 isFlushing 置为true。这样，无论调用多少次 flushJob 方法，但是在一个周期内，它只会执行一次。需要注意的是，在 flushJob 中通过 p.then 将一个副作用函数添加到微任务队列，在微任务队列中完成对 jobQueue 的遍历执行。

## 5. 小结

以上4个部分对上一节内容设计的响应式系统进行了完善和增强。

下一次分享：Vue.js 中两个核心的特性 computed 和 watch 的实现原理。

##  6. 附件

程序源码：

```js
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="app"></div>
  <button onclick="executeSequence()">控制执行顺序</button>
  <button onclick="executeTimes()">控制执行次数</button>
</body>
</html>
<script>
  let activeEffect
  const bucket = new WeakMap()
  const effectStack = []

  function cleanup (effectFn) {
    for (let i = 0; i < effectFn.deps.length; i++) {
      const deps = effectFn.deps[i]
      deps.delete(effectFn)
    }
    effectFn.deps.length = 0
  }

  function effectRegister (fn, options = {}) {
    const effectFn = () => {
      cleanup(effectFn)
      activeEffect = effectFn
      effectStack.push(activeEffect)
      fn()
      effectStack.pop()
      activeEffect = effectStack[effectStack.length -1]
    }

    effectFn.options = options
    effectFn.deps = []
    effectFn()
  }

  function track (target, key) {
    if (!activeEffect) {
      return
    }

    let depsMap = bucket.get(target)
    if (!depsMap) {
      depsMap = new Map()
      bucket.set(target, depsMap)
    }

    let deps = depsMap.get(key)
    if (!deps) {
      deps = new Set()
      depsMap.set(key, deps)
    }

    deps.add(activeEffect)
    activeEffect.deps.push(deps)
  }


  function trigger (target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) {
      return
    }

    const deps = depsMap.get(key)
    const depsToRun = new Set()
    deps && deps.forEach((effectFn) => {
      (effectFn !== activeEffect) && depsToRun.add(effectFn)
    })
    depsToRun && depsToRun.forEach(effectFn => {
      // 如果一个副作用函数存在调度器，则调用改调度器，并将副作用函数作为参数传递，让调度器决定副作用函数执行的时机
      if (effectFn.options.scheduler) {
        effectFn.options.scheduler(effectFn)
      } else {
        effectFn()
      }
    })
  }

  function getProxy (obj) {
    return new Proxy(obj, {
      get (target, key) {
        track(target, key)
        return target[key]
      },
      set (target, key, newValue) {
        target[key] = newValue
        trigger(target, key)
        return true
      }
    })
  }

  const obj = {
    count: 0
  }

  const objProxy = getProxy(obj)

  const executeSequence = () => {
    effectRegister(() => {
      console.log(objProxy.count)
    }, {
      scheduler (fn) {
        setTimeout(() => fn(), 100)
      }
    })

    objProxy.count++

    console.log('结束了')
  }

  const executeTimes = () => {
    // 定义一个任务队列
    const jobQueue = new Set()
    // 使用 Promise.resolve() 创建一个 promise 实例，用它将一个任务添加到微任务队列
    const p = Promise.resolve()

    // 一个标志，代表是否正在刷新队列
    let isFlushing = false
    function flushJob () {
      // 如果队列正在刷新，则什么都不做
      if (isFlushing) {
        return
      }

      // 队列未刷新时，改变状态标志为正在刷新
      isFlushing = true
      // 在微任务队列中刷新 jobQueue 队列
      p.then(() => {
        jobQueue.forEach(job => job())
      }).finally(() => {
        // 结束后重置 isFlushing为未刷新状态
        isFlushing = false
      })
    }

    effectRegister(() => {
      console.log(objProxy.count)
    }, {
      scheduler (fn) {
        // 每次调度时，将副作用函数添加到任务队列中
        jobQueue.add(fn)
        // 调用 flushJob 刷新队列
        flushJob()
      }
    })

    objProxy.count++
    objProxy.count++
  }
</script>

```

