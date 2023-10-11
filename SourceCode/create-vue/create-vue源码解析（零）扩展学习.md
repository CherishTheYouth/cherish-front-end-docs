**摘要：**

本系列主要从小白视角入手，探讨 `create-vue` 脚手架的实现细节，共分为五个部分，本文是该系列的第零篇——扩展学习。

本文主要讲解 `create-vue` 实现过程中需要用的一些工具库的基础信息和使用方法。

### 📖 process.argv

`process.argv` 属性返回数组，其中包含启动 Node.js 进程时传入的命令行参数。

其中第一个元素是 Node.js 的可执行文件路径, 第二个元素是当前执行的 JavaScript 文件路径，之后的元素是命令行参数。

process.argv.slice(2)，可去掉前两个元素，只保留命令行参数部分。

![image-20230829231511469](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e65ffe71c1bc4aa5be120aa6da9efaee~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=985\&h=619\&s=58400\&e=png\&b=1c1c1c)

如图的示例，使用 `ts-node` 执行 `index.ts` 文件，所以 `process.argv` 的第一个参数是 `ts-node` 的可执行文件的路径，第二个参数是被执行的 ts 文件的路径，也就是 `index.ts` 的路径，如果有其他参数，则从 `process.argv` 数组的第三个元素开始。

![image-20230829232834955](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3e1b7c5c72f469a9c379cdaf7630231~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=777\&h=223\&s=16921\&e=png\&b=181818)

### 📖 minimist

[minimist - npm (npmjs.com)](https://www.npmjs.com/package/minimist)

`minimist` **是一个用于解析命令行参数的 JavaScript 函数库**。**它可以将命令行参数解析为一个对象，方便在代码中进行处理和使用**。其中参数名作为对象的属性，参数值作为对象的属性值。

它可以处理各种类型的命令行参数，包括带有**选项标志**的参数、带**有值的参数**以及**没有值**的参数。我们可以通过访问对象的属性来获取相应的命令行参数值。

假设我们在命令行中运行以下命令：

```bash
node example/parse.js -x 3 -y 4 -n 5 -abc --beep=boop foo bar baz
```

那么 `minimist` 将会解析这些命令行参数，并将其转换为以下对象：

```ts
{
  _: ['foo', 'bar', 'baz'],
  x: 3,
  y: 4,
  n: 5,
  a: true,
  b: true,
  c: true,
  beep: 'boop'
}
```

这样，我们就可以在代码中使用 argv.x和 argv.y来获取相应的参数值

除了基本的用法外，minimist 还提供了一些高级功能，例如设置默认值、解析布尔型参数等.

如：

```ts
const argv = minimist(args, opts={})
```

argv.\_ 包含所有没有关联选项的参数;

'--' 之后的任何参数都不会被解析，并将以`argv._` 结束。

options有：

* opts.string： 要始终视为字符串的字符串或字符串数组参数名称；

* opts.boolean: 布尔值、字符串或字符串数组，始终视为布尔值。如果为true，则将所有不带等号的'--'参数视为布尔值。如 opts.boolean 设为 true, 则 --save 解析为 save: true；

* opts.default: 将字符串参数名映射到默认值的对象；

* opts.stopEarly: 当为true时，用第一个非选项后的所有内容填充argv.\_；

* opts.alias: 将字符串名称映射到字符串或字符串参数名称数组，以用作别名的对象；

* opts\['--']: 当为true时，用 '--' 之前的所有内容填充argv.\_; 用 '--' 之后的所有内容填充 argv\['--']，如：

  ```ts
  minimist(('one two three -- four five --six'.split(' '), { '--': true }))
  
  // 结果
  {
    _: ['one', 'two', 'three'],
    '--': ['four', 'five', '--six']
  }
  ```

* opts.unknown: 用一个命令行参数调用的函数，该参数不是在opts配置对象中定义的。如果函数返回false，则未知选项不会添加到argv。



### 📖 prompts

[prompts - npm (npmjs.com)](https://www.npmjs.com/package/prompts)

prompts 是一个用于创建交互式命令行提示的 JavaScript 库。它可以方便地与用户进行命令行交互，接收用户输入的值，并根据用户的选择执行相应的操作。在 prompts 中，问题对象（prompt object）是用于定义交互式提示的配置信息。它包含了一些属性，用于描述问题的类型、提示信息、默认值等。

下面是 `prompt object` 的常用属性及其作用：

 *   type：指定问题的类型，可以是 'text'、'number'、'confirm'、'select'、'multiselect'、‘null’ 等。不同的类型会影响用户交互的方式和输入值的类型。当为 `null` 时会跳过，当前对话。
 *   name：指定问题的名称，用于标识用户输入的值。在返回的结果对象中，每个问题的名称都会作为属性名，对应用户的输入值。
 *   message：向用户展示的提示信息。可以是一个字符串，也可以是一个函数，用于动态生成提示信息。
 *   initial：指定问题的默认值。用户可以直接按下回车键接受默认值，或者输入新的值覆盖默认值。
 *   validate：用于验证用户输入值的函数。它接受用户输入的值作为参数，并返回一个布尔值或一个字符串。如果返回布尔值 false，表示输入值无效；如果返回字符串，则表示输入值无效的错误提示信息。
 *   format：用于格式化用户输入值的函数。它接受用户输入的值作为参数，并返回一个格式化后的值。可以用于对输入值进行预处理或转换。
 *   choices：用于指定选择型问题的选项。它可以是一个字符串数组，也可以是一个对象数组。每个选项可以包含 title 和 value 属性，分别用于展示选项的文本和对应的值。
 *   onRender：在问题被渲染到终端之前触发的回调函数。它接受一个参数 prompt，可以用于修改问题对象的属性或执行其他操作。例如，你可以在 onRender 回调中动态修改提示信息，根据不同的条件显示不同的信息。
 *   onState：在用户输入值发生变化时触发的回调函数。它接受两个参数 state 和 prompt，分别表示当前的状态对象和问题对象。
 *   onSubmit：在用户完成所有问题的回答并提交之后触发的回调函数。它接受一个参数 result，表示用户的回答结果。你可以在 onSubmit 回调中根据用户的回答执行相应的操作，例如保存数据、发送请求等。

