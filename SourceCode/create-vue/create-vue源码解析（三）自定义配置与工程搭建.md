**摘要：**

本系列主要从小白视角入手，探讨 `create-vue` 脚手架的实现细节，共分为五个部分，本文是该系列的第三篇——自定义配置与脚手架工程的生成。

本文将主要讲解脚手架是如何实现终端自定义命令和如何根据命令生成脚手架工程的。

快速直达本系列其他文章：

[create-vue源码解析（一）准备工作 - 掘金 (juejin.cn)](https://juejin.cn/post/7280008213671493647#heading-7)

## 1. 主体概览

相比于动辄几万行的库来说，index.ts 并不复杂，包括注释，只有短短的460多行。
我们先整体浏览一遍代码，大致了解下这个文件是怎么做到在短短400多行就完成了一个如此优秀的脚手架。

经过3分钟的整体浏览，我们带着一脸问号划到了文件的最底部，终于看到了这个程序的真相，短短3行：

```typescript
init().catch((e) => {
  console.error(e)
})
```

一切的一起都围绕 `init` 方法展开。

那么接下来，我们一步一步，庖丁解牛，揭开美人的面纱。让我们移步 init 方法。

```ts
// index.ts
async function init() {
  ...

  try {
    // Prompts:
    // - Project name:
    //   - whether to overwrite the existing directory or not?
    //   - enter a valid package name for package.json
    // - Project language: JavaScript / TypeScript
    // - Add JSX Support?
    // - Install Vue Router for SPA development?
    // - Install Pinia for state management?
    // - Add Cypress for testing?
    // - Add Playwright for end-to-end testing?
    // - Add ESLint for code quality?
    // - Add Prettier for code formatting?
    result = await prompts(
      [
        {
          name: 'projectName',
          type: targetDir ? null : 'text',
          message: 'Project name:',
          initial: defaultProjectName,
          onState: (state) => (targetDir = String(state.value).trim() || defaultProjectName)
        },
        {
          name: 'shouldOverwrite',
          type: () => (canSkipEmptying(targetDir) || forceOverwrite ? null : 'confirm'),
          message: () => {
            const dirForPrompt =
              targetDir === '.' ? 'Current directory' : `Target directory "${targetDir}"`

            return `${dirForPrompt} is not empty. Remove existing files and continue?`
          }
        },
        ....
      ],
      {
        onCancel: () => {
          throw new Error(red('✖') + ' Operation cancelled')
        }
      }
    )
  } catch (cancelled) {
    console.log(cancelled.message)
    process.exit(1)
  }

  ...

const templateRoot = path.resolve(__dirname, 'template')
  const render = function render(templateName) {
    const templateDir = path.resolve(templateRoot, templateName)
    renderTemplate(templateDir, root)
  }
  // Render base template
  render('base')

  // Add configs.
  if (needsJsx) {
    render('config/jsx')
  }
  if (needsRouter) {
    render('config/router')
  }
  
  ...
}
```

程序有点长，但是流程非常清晰直白，相比于 vue cli 的各种回调方法处理，它没有一丝一毫的拐弯抹角，对于想入门源码分析的同学比较友好。其实也就两个部分：

*   询问用户需要什么配置的脚手架工程；

* 根据用户配置生成相应的脚手架工程；

  

下文我们主要围绕这2部分来进行阅读和分析。首先，我们先看第一部分——获取用户自定义配置。

## 2. 实现终端的交互，获取用户的自定义配置 👍

分析代码总是枯燥的，但是既然是读源码，那再枯燥也得坚持。最终我们还得回到代码上，逐行进行解析。

### code snippet 1：打印标题

```ts
 console.log()
  console.log(
    // 确定 Node.js 是否在终端上下文中运行的首选方法是检查 process.stdout.isTTY 属性的值是否为 true
    process.stdout.isTTY && process.stdout.getColorDepth() > 8
      ? banners.gradientBanner
      : banners.defaultBanner
  )
  console.log()
```

![image-20230829230134126](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f5f8c5ac8c74651bd824f7a8ebc067c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=512\&h=94\&s=4783\&e=png\&b=181818)

这段代码主要就是为了实现脚本执行的这行标题了，判断脚本是否在终端中执行，然后判断终端环境是否能支持渐变色相关的能力，支持则输出一个渐变色的炫酷的 `banner` 提示，否则输出一个默认的朴素的 `banner` 提示。花里胡哨，但你别说，还真的很好看。

### code snippet 2：获取node.js进程执行目录

```ts
  const cwd = process.cwd() // 当前node.js 进程执行时的工作目录
```

也就是你在那个目录执行 `create-vue`, `cwd` 就是相应的目录了。

### code snippet 3: 读取终端输入指令配置

```ts
  // possible options:
  // --default
  // --typescript / --ts
  // --jsx
  // --router / --vue-router
  // --pinia
  // --with-tests / --tests (equals to `--vitest --cypress`)
  // --vitest
  // --cypress
  // --playwright
  // --eslint
  // --eslint-with-prettier (only support prettier through eslint for simplicity)
  // --force (for force overwriting)
  const argv = minimist(process.argv.slice(2), {
    alias: {
      typescript: ['ts', 'TS'],
      'with-tests': ['tests'],
      router: ['vue-router']
    },
    string: ['_'], // 布尔值、字符串或字符串数组，始终视为布尔值。如果为true，则将所有不带等号的'--'参数视为布尔值
    // all arguments are treated as booleans
    boolean: true
  })
  console.log('argv:', argv)
```

以上代码中，`process.argv` 属性返回一个数组，包含启动 Node.js 进程时传入的命令行参数，其中第一个元素是 Node.js 的可执行文件路径, 第二个元素是当前执行的 JavaScript 文件路径，之后的元素是命令行参数。`process.argv.slice(2)` 表示截取第二个参数开始往后的参数（一般是用户自定义的参数）， 然后使用 `minimist` 函数将截取的参数解析为一个对象，方便后续读取这些配置。

接下来通过举例测试说明以上的`possible options:`选项得出的结果：

**--default**

```bash
> ts-node index.ts up-web-vue --default

// result:
argv: { _: [ 'up-web-vue' ], default: true }
```

![image-20230830194343904](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/307dcd4fc5af4c708b3d4f885a2d2063~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=558\&h=129\&s=9548\&e=png\&b=181818)

**--typescript / --ts**

这个别名的配置，官方的解释有点不太直观，通过测试发现，它是这个意思。

```typescript
alias: {
  typescript: ['ts', 'TS'],
  'with-tests': ['tests'],
  router: ['vue-router']
},
```

以 `typescript: ['ts', 'TS']` 为例，输入 `typescript`, `ts`, `TS` 任意一个, 将同时生成 以这 3 个字段为 `key`  的属性，如：

```bash
> ts-node index.ts up-web-vue --ts
result:
argv: 
{
	_: [ 'up-web-vue' ], 
	typescript: true,
	ts: true,
	TS: true
}
```

![image-20230830200226086](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aff4350e99144e5587ab92b8500badec~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=626\&h=130\&s=10539\&e=png\&b=181818)

以上是2个典型的例子。

另外，根据 `minimist` 的配置，当 `options.boolean` 为 `true` 时，所有 `--youroption` 后面不带等号的，都视为`youroption`为布尔值，即: `youroption: true`, 所以，以上选项的结果依次为：

```ts
// --default
  // --typescript / --ts  // { typescript: true, ts: true, TS: true }
  // --jsx // { jsx: true }
  // --router / --vue-router // { router: true, vue-router: true }
  // --pinia // { pinia: true }
  // --with-tests / --tests // { 'with-tests': true, tests: true }
  // --vitest // { vitest: true }
  // --cypress // { cypress: true }
  // --playwright // { playwright: true }
  // --eslint // { eslint: true }
  // --eslint-with-prettier { 'eslint-with-prettier': true }
  // --force { force: true }
```

以上这一部分主要是在用户主动输入 `cli` 指令时的一些选项的实现。另一种更直观的配置方式是通过终端问答交互方式（提示方式）进行的，下面我们来看这部分的功能。



### code snippet 4: 终端提示引导用户选择指令配置

接下来是终端提示功能。提示功能会引导用户根据自己的需求选择不同的项目配置。

如图所示，这是 `create-vue` 的提示模块：

![截屏2023-08-30 23.14.59](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02ebb76cf5ab4957901f3d46220bbe45~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=623\&h=156\&s=31435\&e=png\&b=1d1d1d)

code snippet 5- x是对 code snippet 4 的展开讲解

### code snippet 5：当用户进行手动输入配置时，跳过终端提示

```ts
// if any of the feature flags is set, we would skip the feature prompts
// 翻译一下：如果在上面讲述的部分，已经使用终端指令确定了安装选项，那么下文中相关的提示项就会被跳过。
// 如果在 create-vue project-name 后附加了任何以下的选项，则此状态为true，如：create-vue project-name --ts
// ?? 是 空值合并操作符，当左侧的操作数为 null 或者 undefined 时，返回其右侧操作数，否则返回左侧操作数
  const isFeatureFlagsUsed =
    typeof (
      argv.default ??
      argv.ts ??
      argv.jsx ??
      argv.router ??
      argv.pinia ??
      argv.tests ??
      argv.vitest ??
      argv.cypress ??
      argv.playwright ??
      argv.eslint
    ) === 'boolean'
  
  // 以下是此字段控制的部分逻辑, 可以看到，当 isFeatureFlagsUsed 为 true 时，prompts 的type值为 null.此时此提示会跳过
  result = await prompts(
      [
          ....
      	{
          name: 'needsTypeScript',
          type: () => (isFeatureFlagsUsed ? null : 'toggle'),
          message: 'Add TypeScript?',
          initial: false,
          active: 'Yes',
          inactive: 'No'
        },
        {
          name: 'needsJsx',
          type: () => (isFeatureFlagsUsed ? null : 'toggle'),
          message: 'Add JSX Support?',
          initial: false,
          active: 'Yes',
          inactive: 'No'
        },
        ...
      ])
```

如果在 create-vue project-name 后附加了任何以下的选项，则 `isFeatureFlagsUsed` 为true，如：`create-vue project-name --ts`.

只要输入任意一个选项，则以下的所有提示选项都将跳过，直接执行 cli。以下是测试结果：

![image-20230831192059089](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7679732f8d3841f39e0db2203cdefaab~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=661\&h=233\&s=13773\&e=png\&b=181818)



### code snippet 6: 读取目标目录路径、默认生成的项目名、是否强制覆盖目标目录

```ts
  let targetDir = argv._[0]
  const defaultProjectName = !targetDir ? 'vue-project' : targetDir
```

根据以上对  `argv` 的解析，`argv._ ` 是一个数组，其中的值是我们输入的目标文件夹名称， 如下示例：

```bash
> ts-node index.ts up-web-vue
```

得到的 `argv._` 的值为: `['up-web-vue']`.

![image-20230831193020240](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4389f5f90c614bac93347fa4c68aea63~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=675\&h=141\&s=9318\&e=png\&b=181818)

所以，`targetDir` 即为你希望脚手架为你生成的项目的文件夹名称，`defaultProjectName` 则是未指定文件夹名称时的默认名称。

### code snippet 7：是否强制覆盖目标文件夹

`argv.force` 值即为 `ts-node index.ts up-web-vue --force` 时 `force` 的值，是强制覆盖选项的快捷指令，如果使用此选项，则后面则不会弹出询问是否强制覆盖的选项。

如果目标目录是空母录，也会跳过此提示。`canSkipEmptying()` 方法即用来判断是否是空目录。

```ts
const forceOverwrite = argv.force
...
{
  name: 'shouldOverwrite',
      // forceOverwrite 为 true 时跳过
  type: () => (canSkipEmptying(targetDir) || forceOverwrite ? null : 'confirm'),
  message: () => {
    const dirForPrompt =
      targetDir === '.' ? 'Current directory' : `Target directory "${targetDir}"`

    return `${dirForPrompt} is not empty. Remove existing files and continue?`
  }
},
....
// 判断 dir 文件夹是否是空文件夹
function canSkipEmptying(dir: string) {
  // dir目录不存在，则当然是空目录
  if (!fs.existsSync(dir)) {
    return true
  }

  // 读取dir内的文件
  const files = fs.readdirSync(dir)
  // 无文件，当然是空目录
  if (files.length === 0) {
    return true
  }
  // 仅有.git目录，则同样视为一个空目录
  if (files.length === 1 && files[0] === '.git') {
    return true
  }

  return false
}
```



### code snippet 8：定义一个容器，保存用户根据提示选择的安装配置

```ts
  let result: {
    projectName?: string
    shouldOverwrite?: boolean
    packageName?: string
    needsTypeScript?: boolean
    needsJsx?: boolean
    needsRouter?: boolean
    needsPinia?: boolean
    needsVitest?: boolean
    needsE2eTesting?: false | 'cypress' | 'playwright'
    needsEslint?: boolean
    needsPrettier?: boolean
  } = {}
  // 定义一个result对象，prompts 提示的结果保存在此对象中。
```

以上定义一个 `result` ，从其类型的定义不难看出，该对象用来保存 prompts 提示结束后用户选择的结果。

### code snippet 9：终端提示功能实现

接下来介绍生成项目之前的 prompts 提示部分。

prompts 是一个用于创建交互式命令行提示的 JavaScript 库。它可以方便地与用户进行命令行交互，接收用户输入的值，并根据用户的选择执行相应的操作。（详细用法见 扩展学习 一文）

我们结合代码来分析，每个提示项的作用分别是什么。

```ts
{
  name: 'projectName',
  type: targetDir ? null : 'text',
  message: 'Project name:',
  initial: defaultProjectName,
  onState: (state) => (targetDir = String(state.value).trim() || defaultProjectName)
},
```

项目名称，这里如果在运行 cli 时**未输入**项目名称，则会展示此选项，默认选项为 `defaultProjectName`, 在用户确认操作后，将输入值作为生成目录的文件夹名。

如果运行 cli 时已输入项目名称，则此提示会跳过。

```ts
{
  name: 'shouldOverwrite',
  type: () => (canSkipEmptying(targetDir) || forceOverwrite ? null : 'confirm'),
  message: () => {
      const dirForPrompt =
            targetDir === '.' ? 'Current directory' : `Target directory "${targetDir}"`

      return `${dirForPrompt} is not empty. Remove existing files and continue?`
  }
},
{
  name: 'overwriteChecker',
  type: (prev, values) => {
    if (values.shouldOverwrite === false) {
      throw new Error(red('✖') + ' Operation cancelled')
    }
    return null
  }
},
```

以上两个 prompts 为一组，在目标文件夹为空时，不需要选择强制覆盖的配置，会跳过这2个 prompts 。

当不为空时，首先会询问是否强制覆盖目标目录，其中 message 字段根据用户输入的 目录名动态生成，这里特别考虑了 ‘.’ 这种目录选择方式（当前目录）。

`type` 即可以是字符串，布尔值，也可以是 Function ，当为 Function 时，拥有2个默认参数 `prev` 和 `values`, `prve` 表示前一个选项的选择的值，`values` 表示已经选择了的所有选项的值。

此处根据上一个选项的选择结果来决定下一个选项的类型。这段代码中，当用户选择不强制覆盖目标目录时，则脚手架执行终止，抛出 `Operation cancelled` 的错误提示。

```tsx
{
  name: 'packageName',
  type: () => (isValidPackageName(targetDir) ? null : 'text'),
  message: 'Package name:',
  initial: () => toValidPackageName(targetDir),
  validate: (dir) => isValidPackageName(dir) || 'Invalid package.json name'
},  
...
// 校验项目名
// 匹配一个项目名称，它可以包含可选的 @scope/ 前缀，后面跟着一个或多个小写字母、数字、-、. 或 ~ 中的任意一个字符。这个正则表达式适用于验证类似于 npm 包名的项目名称格式
function isValidPackageName(projectName) {
  return /^(?:@[a-z0-9-*~][a-z0-9-*._~]*\/)?[a-z0-9-~][a-z0-9-._~]*$/.test(projectName)
}

// 将不合法的项目名修改为合法的报名
function toValidPackageName(projectName) {
  return projectName
    .trim()
    .toLowerCase()
    .replace(/\s+/g, '-')
    .replace(/^[._]/, '')
    .replace(/[^a-z0-9-~]+/g, '-')
}
```

![截屏2023-08-31 23.16.14](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebbc6fc34ff44d5b890f45cc9dc2e6f5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=594\&h=115\&s=21222\&e=png\&b=1e1e1e)

以上代码，对项目名进行校验，看是否符合内置的规则(类似于npm包名的格式) ，然后对不合法的字符进行校准，生成一个默认的项目名，用户可直接点击确认选择使用这个默认的项目名，或者重新输入一次项目名，如果用户再次输入不合法的项目名，则会出现提示 `Invalid package.json name`, 然后无法继续往下执行，直到用户修改为合法的 项目名。

![截屏2023-08-31 23.26.36](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68a786562d0e4e668b3364267e0d783f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=559\&h=112\&s=19199\&e=png\&b=1d1d1d)

![截屏2023-08-31 23.29.34](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c086fdf370854876a8706390cad531a6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=569\&h=122\&s=20348\&e=png\&b=1d1d1d)

```ts
{
    name: 'needsTypeScript',
    type: () => (isFeatureFlagsUsed ? null : 'toggle'),
    message: 'Add TypeScript?',
    initial: false,
    active: 'Yes',
    inactive: 'No'
},
{
    name: 'needsJsx',
    type: () => (isFeatureFlagsUsed ? null : 'toggle'),
    message: 'Add JSX Support?',
    initial: false,
    active: 'Yes',
    inactive: 'No'
},
{
    name: 'needsRouter',
    type: () => (isFeatureFlagsUsed ? null : 'toggle'),
    message: 'Add Vue Router for Single Page Application development?',
    initial: false,
    active: 'Yes',
    inactive: 'No'
},
{
    name: 'needsPinia',
    type: () => (isFeatureFlagsUsed ? null : 'toggle'),
    message: 'Add Pinia for state management?',
    initial: false,
    active: 'Yes',
    inactive: 'No'
},  	
{	
    name: 'needsVitest',
    type: () => (isFeatureFlagsUsed ? null : 'toggle'),
    message: 'Add Vitest for Unit Testing?',
    initial: false,
    active: 'Yes',
    inactive: 'No'
},
          
```

以上这一组选项都是类似的，都是询问是否添加某模块，初始值为 `false` 默认不添加，`active` 和 `inactive` 分别表示2个不同的选项，`isFeatureFlagsUsed` 前面已经讲过，这里略过。

所以这一段依次表示询问用户是否需要添加 `TypeScript` 、`JSX Support`、`Vue Router`、`Pinia`、`Vitest` 。

接下来是e2e测试选项的 prompts：

```ts
{
  name: 'needsE2eTesting',
  type: () => (isFeatureFlagsUsed ? null : 'select'),
  message: 'Add an End-to-End Testing Solution?',
  initial: 0,
  choices: (prev, answers) => [
    { title: 'No', value: false },
    {
      title: 'Cypress',
      description: answers.needsVitest
      ? undefined
      : 'also supports unit testing with Cypress Component Testing',
      value: 'cypress'
    },
    {
      title: 'Playwright',
      value: 'playwright'
    }
  ]
}
```

这里是一个选择类型的 `prompts`，询问是否添加 `e2e` 测试框架，共 3 个选项：

*   不添加
*   添加 `cypress`
*   添加 `playwright`

接下里是 `eslint` 和 `prettier` :

```tsx
{
 	name: 'needsEslint',
  type: () => (isFeatureFlagsUsed ? null : 'toggle'),
  message: 'Add ESLint for code quality?',
  initial: false,
  active: 'Yes',
  inactive: 'No'
},
{
  name: 'needsPrettier',
  type: (prev, values) => {
    if (isFeatureFlagsUsed || !values.needsEslint) {
      return null
    }
    return 'toggle'
  },
  message: 'Add Prettier for code formatting?',
  initial: false,
  active: 'Yes',
  inactive: 'No'
}
```

这2个选项为一组，其中如果选择不集成 `eslint`, 则默认也是不集成的 `prettier` 的，只有选择集成`eslint`， 才可继续选择是否集成 `prettier`.

然后是 `prompts` 部分最后一个部分了，异常捕获：

```ts
try {
  ...
} catch (cancelled) {
  console.log(cancelled.message)
  process.exit(1)
}
```

当在选择过程中按下终止快捷键（ctrl + c）时，或者在选择过程中，触发终止条件时（如上文中某选项的 `throw new Error(red('✖') + ' Operation cancelled')` ），则会进入异常捕获中，此时会打印任务执行终止的提示，并结束此进程。

到此，脚手架第一个部分——“用户自定义配置” 部分全都解析完成了，很好理解，就是使用一些更友好的形式（prompts）来收集用户的需求，使用的工具也很简单易懂。

接下里看一个 cli 真正的核心功能，根据用户配置生成完整的项目结构。

## 3. 根据用户选择生成合理的项目工程 🧱

还是先贴代码（后面针对具体的代码还会再结合分析，此处可先大致浏览，然后迅速跳过）：

```ts
// `initial` won't take effect if the prompt type is null
// so we still have to assign the default values here
  const {
    projectName,
    packageName = projectName ?? defaultProjectName,
    shouldOverwrite = argv.force,
    needsJsx = argv.jsx,
    needsTypeScript = argv.typescript,
    needsRouter = argv.router,
    needsPinia = argv.pinia,
    needsVitest = argv.vitest || argv.tests,
    needsEslint = argv.eslint || argv['eslint-with-prettier'],
    needsPrettier = argv['eslint-with-prettier']
  } = result

  const { needsE2eTesting } = result
  const needsCypress = argv.cypress || argv.tests || needsE2eTesting === 'cypress'
  const needsCypressCT = needsCypress && !needsVitest
  const needsPlaywright = argv.playwright || needsE2eTesting === 'playwright'

  const root = path.join(cwd, targetDir)

  if (fs.existsSync(root) && shouldOverwrite) {
    emptyDir(root)
  } else if (!fs.existsSync(root)) {
    fs.mkdirSync(root)
  }

  console.log(`\nScaffolding project in ${root}...`)

  const pkg = { name: packageName, version: '0.0.0' }
  fs.writeFileSync(path.resolve(root, 'package.json'), JSON.stringify(pkg, null, 2))

  // todo:
  // work around the esbuild issue that `import.meta.url` cannot be correctly transpiled
  // when bundling for node and the format is cjs
  // const templateRoot = new URL('./template', import.meta.url).pathname
  const templateRoot = path.resolve(__dirname, 'template')
  const render = function render(templateName) {
    const templateDir = path.resolve(templateRoot, templateName)
    renderTemplate(templateDir, root)
  }
  
  // Add configs.
  if (needsJsx) {
    render('config/jsx')
  }
  if (needsRouter) {
    render('config/router')
  }
  if (needsPinia) {
    render('config/pinia')
  }
  if (needsVitest) {
    render('config/vitest')
  }
  if (needsCypress) {
    render('config/cypress')
  }
  if (needsCypressCT) {
    render('config/cypress-ct')
  }
  if (needsPlaywright) {
    render('config/playwright')
  }
  if (needsTypeScript) {
    render('config/typescript')

    // Render tsconfigs
    render('tsconfig/base')
    if (needsCypress) {
      render('tsconfig/cypress')
    }
    if (needsCypressCT) {
      render('tsconfig/cypress-ct')
    }
    if (needsPlaywright) {
      render('tsconfig/playwright')
    }
    if (needsVitest) {
      render('tsconfig/vitest')
    }
  }

  // Render ESLint config
  if (needsEslint) {
    renderEslint(root, { needsTypeScript, needsCypress, needsCypressCT, needsPrettier })
  }

  // Render code template.
  // prettier-ignore
  const codeTemplate =
    (needsTypeScript ? 'typescript-' : '') +
    (needsRouter ? 'router' : 'default')
  render(`code/${codeTemplate}`)

  // Render entry file (main.js/ts).
  if (needsPinia && needsRouter) {
    render('entry/router-and-pinia')
  } else if (needsPinia) {
    render('entry/pinia')
  } else if (needsRouter) {
    render('entry/router')
  } else {
    render('entry/default')
  }

// Cleanup.

  // We try to share as many files between TypeScript and JavaScript as possible.
  // If that's not possible, we put `.ts` version alongside the `.js` one in the templates.
  // So after all the templates are rendered, we need to clean up the redundant files.
  // (Currently it's only `cypress/plugin/index.ts`, but we might add more in the future.)
  // (Or, we might completely get rid of the plugins folder as Cypress 10 supports `cypress.config.ts`)

  if (needsTypeScript) {
    // Convert the JavaScript template to the TypeScript
    // Check all the remaining `.js` files:
    //   - If the corresponding TypeScript version already exists, remove the `.js` version.
    //   - Otherwise, rename the `.js` file to `.ts`
    // Remove `jsconfig.json`, because we already have tsconfig.json
    // `jsconfig.json` is not reused, because we use solution-style `tsconfig`s, which are much more complicated.
    preOrderDirectoryTraverse(
      root,
      () => {},
      (filepath) => {
        if (filepath.endsWith('.js')) {
          const tsFilePath = filepath.replace(/\.js$/, '.ts')
          if (fs.existsSync(tsFilePath)) {
            fs.unlinkSync(filepath)
          } else {
            fs.renameSync(filepath, tsFilePath)
          }
        } else if (path.basename(filepath) === 'jsconfig.json') {
          fs.unlinkSync(filepath)
        }
      }
    )

    // Rename entry in `index.html`
    const indexHtmlPath = path.resolve(root, 'index.html')
    const indexHtmlContent = fs.readFileSync(indexHtmlPath, 'utf8')
    fs.writeFileSync(indexHtmlPath, indexHtmlContent.replace('src/main.js', 'src/main.ts'))
  } else {
    // Remove all the remaining `.ts` files
    preOrderDirectoryTraverse(
      root,
      () => {},
      (filepath) => {
        if (filepath.endsWith('.ts')) {
          fs.unlinkSync(filepath)
        }
      }
    )
  }

  // Instructions:
  // Supported package managers: pnpm > yarn > npm
  const userAgent = process.env.npm_config_user_agent ?? ''
  const packageManager = /pnpm/.test(userAgent) ? 'pnpm' : /yarn/.test(userAgent) ? 'yarn' : 'npm'

  // README generation
  fs.writeFileSync(
    path.resolve(root, 'README.md'),
    generateReadme({
      projectName: result.projectName ?? result.packageName ?? defaultProjectName,
      packageManager,
      needsTypeScript,
      needsVitest,
      needsCypress,
      needsPlaywright,
      needsCypressCT,
      needsEslint
    })
  )

  console.log(`\nDone. Now run:\n`)
  if (root !== cwd) {
    const cdProjectName = path.relative(cwd, root)
    console.log(
      `  ${bold(green(`cd ${cdProjectName.includes(' ') ? `"${cdProjectName}"` : cdProjectName}`))}`
    )
  }
  console.log(`  ${bold(green(getCommand(packageManager, 'install')))}`)
  if (needsPrettier) {
    console.log(`  ${bold(green(getCommand(packageManager, 'format')))}`)
  }
  console.log(`  ${bold(green(getCommand(packageManager, 'dev')))}`)
  console.log()
```

第一部分是解析出用户的自定义安装配置项：

```ts
// `initial` won't take effect if the prompt type is null
// so we still have to assign the default values here
// 此处兼顾用户从 prompts 配置读取配置和直接使用 -- 指令进行快速配置。根据前面的分析，当使用 -- 指令快速配置时，`prompts` 不生效，则从 result 中解构出来的属性都为 `undefined`, 此时，则会为其制定默认值，也即是以下代码中从 `argv` 中读取的值。
const {
  projectName,
  packageName = projectName ?? defaultProjectName,
  shouldOverwrite = argv.force,
  needsJsx = argv.jsx,
  needsTypeScript = argv.typescript,
  needsRouter = argv.router,
  needsPinia = argv.pinia,
  needsVitest = argv.vitest || argv.tests,
  needsEslint = argv.eslint || argv['eslint-with-prettier'],
  needsPrettier = argv['eslint-with-prettier']
} = result

const { needsE2eTesting } = result
const needsCypress = argv.cypress || argv.tests || needsE2eTesting === 'cypress'
const needsCypressCT = needsCypress && !needsVitest
const needsPlaywright = argv.playwright || needsE2eTesting === 'playwright'
```

此处兼顾用户从 prompts 配置读取配置和直接使用 -- 指令进行快速配置。根据前面的分析，当使用 -- 指令快速配置时，`prompts` 不生效，则从 result 中解构出来的属性都为 `undefined`, 此时，则会为其制定默认值，也即是以下代码中从 `argv` 中读取的值。

紧接着开始进入文件操作阶段了。

```ts
const root = path.join(cwd, targetDir) // 计算目标文件夹的完整文件路径

// 读取目标文件夹状态，看该文件夹是否是一个已存在文件夹，是否需要覆盖
// 文件夹存在，则清空，文件夹不存在，则创建
if (fs.existsSync(root) && shouldOverwrite) {
  emptyDir(root)
} else if (!fs.existsSync(root)) {
  fs.mkdirSync(root)
}
// 一句提示, 脚手架项目在xxx目录
console.log(`\nScaffolding project in ${root}...`)

// emptyDir
function emptyDir(dir) {
// 代码写的很严谨、健壮，即使外层调用的地方已经判断了目录是否存在，在实际操作目录中依然会重新判断一下，与外部的业务代码不产生多余的依赖关系。
  if (!fs.existsSync(dir)) {
    return
  }

  // 遍历目录，清空目录里的文件夹和文件
  postOrderDirectoryTraverse(
    dir,
    (dir) => fs.rmdirSync(dir),
    (file) => fs.unlinkSync(file)
  )
}

// postOrderDirectoryTraverse  from './utils/directoryTraverse'
// /utils/directoryTraverse.ts
// 这个方法递归的遍历给定文件夹，并对内部的 文件夹 和 文件 按照给定的回调函数进行操作。
export function postOrderDirectoryTraverse(dir, dirCallback, fileCallback) {
  // 遍历dir下的文件
  for (const filename of fs.readdirSync(dir)) {
    // 如果文件是 .git 文件，则跳过
    if (filename === '.git') {
      continue
    }
    const fullpath = path.resolve(dir, filename) // 计算文件路径
    // 如果该文件是一个文件夹类型，则递归调用此方法，继续对其内部的文件进行操作
    if (fs.lstatSync(fullpath).isDirectory()) {
      postOrderDirectoryTraverse(fullpath, dirCallback, fileCallback)
      // 当然递归完后，不要忘记对该文件夹自己进行操作
      dirCallback(fullpath)
      continue
    }
    // 如果该文件不是文件夹类型，是纯文件，则直接执行对单个文件的回调操作
    fileCallback(fullpath)
  }
}
```

以上部分主要是判断目标目录的状态，清空目标目录的实现过程。

> fs.unlinkSync: 同步删除文件；
>
> fs.rmdirSync: 同步删除给定路径下的目录;

清空目标目录的实现，其核心是通过 `postOrderDirectoryTraverse` 方法来遍历目标文件夹，通过传入自定义的回调方法来决定对 `file` 和 `directory` 执行何种操作。此处对目录执行 `(dir) => fs.rmdirSync(dir)` 方法，来移除目录，对文件执行 `(file) => fs.unlinkSync(file)` 来移除。

```ts
// 定义脚手架工程的 package.json 文件的内容，这里 packageName 和 projectName是保持一致的。
const pkg = { name: packageName, version: '0.0.0' }
// 创建一个 package.json 文件，写入 pkg 变量定义的内容
fs.writeFileSync(path.resolve(root, 'package.json'), JSON.stringify(pkg, null, 2))
```

> fs.writeFileSync(file, data\[, option])
>
> *   file: 它是一个字符串，Buffer，URL或文件描述整数，表示必须在其中写入文件的路径。使用文件描述符将使其行为类似于fs.write()方法。
> *   data: 它是将写入文件的字符串，Buffer，TypedArray或DataView。
> *   options: 它是一个字符串或对象，可用于指定将影响输出的可选参数。它具有三个可选参数：
>     *   \*\*encoding:\*\*它是一个字符串，它指定文件的编码。默认值为“ utf8”。
>     *   \*\*mode:\*\*它是一个整数，指定文件模式。默认值为0o666。
>     *   \*\*flag:\*\*它是一个字符串，指定在写入文件时使用的标志。默认值为“ w”。
>
> JSON.stringify(value\[, replacer \[, space]])
>
> *   value: 将要序列化成 一个 JSON 字符串的值。
>
> *   replacer: 可选, 如果该参数是一个函数，则在序列化过程中，被序列化的值的每个属性都会经过该函数的转换和处理；如果该参数是一个数组，则只有包含在这个数组中的属性名才会被序列化到最终的 JSON 字符串中；如果该参数为 null 或者未提供，则对象所有的属性都会被序列化。
>
> *   space: 可选, 指定缩进用的空白字符串，用于美化输出（pretty-print）；如果参数是个数字，它代表有多少的空格；上限为 10。该值若小于 1，则意味着没有空格；如果该参数为字符串（当字符串长度超过 10 个字母，取其前 10 个字母），该字符串将被作为空格；如果该参数没有提供（或者为 null），将没有空格。

接下来正式进入模板渲染环节了。

> \_\_dirname 可以用来动态获取当前文件所属目录的绝对路径;
>
> \_\_filename 可以用来动态获取当前文件的绝对路径，包含当前文件 ;
>
> path.resolve() 方法是以程序为根目录，作为起点，根据参数解析出一个绝对路径；

```ts
// 计算模板所在文件加路径
const templateRoot = path.resolve(__dirname, 'template')
// 定义模板渲染 render 方法，参数为模板名
const render = function render(templateName) {
    const templateDir = path.resolve(templateRoot, templateName)
    // 核心是这个 renderTemplate 方法，第一个参数是源文件夹目录，第二个参数是目标文件夹目录
    renderTemplate(templateDir, root)
}
```

以上代码定义了一个模板渲染方法，根据模板名，生成不同的项目模块。

其核心为 `renderTemplate` 方法，下面来看此方法的代码实现：

```ts
// /utils/renderTemplate.ts
/**
 * Renders a template folder/file to the file system,
 * by recursively copying all files under the `src` directory,
 * with the following exception:
 *   - `_filename` should be renamed to `.filename`
 *   - Fields in `package.json` should be recursively merged
 * @param {string} src source filename to copy
 * @param {string} dest destination filename of the copy operation
 */
/** 翻译一下这段函数说明：
 * 通过递归复制 src 目录下的所有文件，将模板文件夹/文件 复制到 目标文件夹，
 * 但以下情况例外：
 * 1. '_filename' 会被重命名为 '.filename';
 * 2. package.json 文件中的字段会被递归合并；
 */
function renderTemplate(src, dest) {
  const stats = fs.statSync(src) // src 目录的文件状态

  // 如果当前src是文件夹，则在目标目录中递归的生成此目录的子目录和子文件，但 node_modules 忽略
  if (stats.isDirectory()) {
    // skip node_module
    if (path.basename(src) === 'node_modules') {
      return
    }

    // if it's a directory, render its subdirectories and files recursively
    fs.mkdirSync(dest, { recursive: true })
    for (const file of fs.readdirSync(src)) {
      renderTemplate(path.resolve(src, file), path.resolve(dest, file))
    }
    return
  }

  // path.basename(path[, ext]) 返回 path 的最后一部分，也即是文件名了。
  const filename = path.basename(src)

  // 如果当前src是单个文件，则直接复制到目标路径下，但有以下几类文件例外，要特殊处理。
  // package.json 做合并操作，并对内部的属性的位置做了排序；
  // extensions.json 做合并操作；
  // 以 _ 开头的文件名转化为 . 开头的文件名；
  // _gitignore 文件，将其中的配置追加到目标目录对应文件中；
  if (filename === 'package.json' && fs.existsSync(dest)) {
    // merge instead of overwriting
    const existing = JSON.parse(fs.readFileSync(dest, 'utf8'))
    const newPackage = JSON.parse(fs.readFileSync(src, 'utf8'))
    const pkg = sortDependencies(deepMerge(existing, newPackage))
    fs.writeFileSync(dest, JSON.stringify(pkg, null, 2) + '\n')
    return
  }

  if (filename === 'extensions.json' && fs.existsSync(dest)) {
    // merge instead of overwriting
    const existing = JSON.parse(fs.readFileSync(dest, 'utf8'))
    const newExtensions = JSON.parse(fs.readFileSync(src, 'utf8'))
    const extensions = deepMerge(existing, newExtensions)
    fs.writeFileSync(dest, JSON.stringify(extensions, null, 2) + '\n')
    return
  }

  if (filename.startsWith('_')) {
    // rename `_file` to `.file`
    dest = path.resolve(path.dirname(dest), filename.replace(/^_/, '.'))
  }

  if (filename === '_gitignore' && fs.existsSync(dest)) {
    // append to existing .gitignore
    const existing = fs.readFileSync(dest, 'utf8')
    const newGitignore = fs.readFileSync(src, 'utf8')
    fs.writeFileSync(dest, existing + '\n' + newGitignore)
    return
  }

  fs.copyFileSync(src, dest)
}

// /utils/deepMerge.ts
const isObject = (val) => val && typeof val === 'object'
// 合并数组时，利用Set数据类型 对数组进行去重
const mergeArrayWithDedupe = (a, b) => Array.from(new Set([...a, ...b]))

/**
 * Recursively merge the content of the new object to the existing one
 * 递归的将两个对象进行合并
 * @param {Object} target the existing object
 * @param {Object} obj the new object
 */
function deepMerge(target, obj) {
  for (const key of Object.keys(obj)) {
    const oldVal = target[key]
    const newVal = obj[key]

    if (Array.isArray(oldVal) && Array.isArray(newVal)) {
      target[key] = mergeArrayWithDedupe(oldVal, newVal)
    } else if (isObject(oldVal) && isObject(newVal)) {
      target[key] = deepMerge(oldVal, newVal)
    } else {
      target[key] = newVal
    }
  }

  return target
}

// /utils/sortDependencies
// 将四种类型的依赖进行一个排序，共4种类型：dependencies、devDependencies、peerDependencies、optionalDependencies
// 这一步没有很大的必要，不要也不影响功能，主要是为了将以上四个类型属性在package.json中的位置统一一下，按照dependencies、devDependencies、peerDependencies、optionalDependencies 这个顺序以此展示这些依赖项。
function sortDependencies(packageJson) {
  const sorted = {}

  const depTypes = ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']

  for (const depType of depTypes) {
    if (packageJson[depType]) {
      sorted[depType] = {}

      Object.keys(packageJson[depType])
        .sort()
        .forEach((name) => {
          sorted[depType][name] = packageJson[depType][name]
        })
    }
  }

  return {
    ...packageJson,
    ...sorted
  }
}
```

> 以上代码已添加了详尽的注释，学习时也可略过相关正文描述

`renderTemplate` 方法主要作用是将模板里的内容按需渲染到脚手架工程中。主要涉及到一些文件的拷贝，合并，调增等操作。以上代码已添加详细注释，这里将其中特殊处理的几个点罗列如下，阅读时着重注意即可：

*   渲染模板时，如果模板中存在 'node\_modules' 文件夹需跳过；
*   package.json 需要与脚手架工程中的package.json做合并操作, 并对内部的属性的位置做了排序；（按照 dependencies , devDependencies , peerDependencies, optionalDependencies 的顺序）
*   extensions.json 需要与脚手架工程中的对应文件做合并操作；
*   以 \_ 开头的文件名转化为 . 开头的文件名；（如\_a.ts => .a.ts）;
*   \_gitignore 文件，需要将其中的配置追加到目标目录对应文件中；

接下来回到 `index.ts` 文件中继续分析主流程：

```ts
// Render base template
// 渲染一个最基础的基于 vite 的 vue3 项目 
render('base')

// Add configs.
if (needsJsx) {
    render('config/jsx')
}
...
// Render ESLint config
if (needsEslint) {
    renderEslint(root, { needsTypeScript, needsCypress, needsCypressCT, needsPrettier })
}

// Render code template.
// prettier-ignore
const codeTemplate =
      (needsTypeScript ? 'typescript-' : '') +
      (needsRouter ? 'router' : 'default')
render(`code/${codeTemplate}`)

// Render entry file (main.js/ts).
if (needsPinia && needsRouter) {
    render('entry/router-and-pinia')
} else if (needsPinia) {
    render('entry/pinia')
} else if (needsRouter) {
    render('entry/router')
} else {
    render('entry/default')
}
```

```ts
  // Render base template
  render('base')
```

首先渲染一个最基础的基于 `vite` 的 `vue3` 项目，除了 `renderTemplate` 方法中的一些特殊点之外， `render('base')` 中需要注意的一点是，这并不是一个完善的能跑的 vue3 工程，这里面缺少了 `main.js` 文件，这个文件会在后面的 `Render entry file ` 部分进行补充。

![截屏2023-09-02 21.43.24](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a5e904fcb2c46a9b9b4774ec699e7a3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=281\&h=188\&s=20125\&e=png\&b=222123)

如图所示，无 `main.j(t)s` 文件.

紧接着是渲染用户选择的一些配置：

*   Jsx 配置：包括 `package.json` 需要安装的依赖 和 `vite.config.js` 中的 相关配置：

```ts
// Add configs.
if (needsJsx) {
  render('config/jsx')
}
```

```ts
// package.json
{
  "dependencies": {
    "vue": "^3.3.2"
  },
  "devDependencies": {
    "@vitejs/plugin-vue-jsx": "^3.0.1", // 主要是这个 plugin-vue-jsx 插件
    "vite": "^4.3.5"
  }
}
// vite.config.js
import { fileURLToPath, URL } from 'node:url'

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue(), vueJsx()],// 主要是这里
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```

*   router 配置：就一个`package.json` 中需要安装的依赖

```ts
if (needsRouter) {
  render('config/router')
}
```

```json
// package.json
{
  "dependencies": {
    "vue": "^3.3.2",
    "vue-router": "^4.2.0" // 这个
  }
}
```

*   pinia 配置：1. package.json 配置；2. 新增一个 `pinia` 使用 `demo`.

```ts
if (needsPinia) {
  render('config/pinia')
}
```

```ts
{
  "dependencies": {
    "pinia": "^2.0.36",
    "vue": "^3.3.2"
  }
}
```

![image-20230902215623065](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c5ad64b02d04c79a6b0ec7bbe476f9e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=974\&h=594\&s=112245\&e=png\&b=1d1d1d)

*   Vitest 配置： 1. package.json 配置；2. vitest.config.js 文件；3. 一个单测用例示例；

```ts
if (needsVitest) {
  render('config/vitest')
}
```

> 具体内容可看源码的模板文件

*   Cypress/cypress-ct/playwright: 与上面操作类似，不赘述，直接看源码的模板文件；

```ts
if (needsCypress) {
  render('config/cypress')
}
if (needsCypressCT) {
  render('config/cypress-ct')
}
if (needsPlaywright) {
  render('config/playwright')
}
```

*   TypeScript 配置：

```ts
if (needsTypeScript) {
  render('config/typescript')
  // Render tsconfigs
  render('tsconfig/base')
  if (needsCypress) {
    render('tsconfig/cypress')
  }
  if (needsCypressCT) {
    render('tsconfig/cypress-ct')
  }
  if (needsPlaywright) {
    render('tsconfig/playwright')
  }
  if (needsVitest) {
    render('tsconfig/vitest')
  }
}
```

`TypeScript` 的配置相对复杂一些，但根本上来说都是一些预先配置好的内容，只要按需从对应模版取正确的配置进行渲染，保证 `TypeScript` 的正常功能即可，亦不赘述。

*   Eslint 配置

接下是 `eslint` 相关配置的渲染，这块是一个单独的渲染函数，跟 `TypeScript`, `Cypress`, `CypressCT` , `Prettier` 这几个模块相关。主要是一些针对行的处理逻辑，核心思路还是一样，通过文件、配置的组合处理，来生成一个完整的功能配置。

```ts
// Render ESLint config
if (needsEslint) {
  renderEslint(root, { needsTypeScript, needsCypress, needsCypressCT, needsPrettier })
}

```

*   模板配置

```ts
// Render code template.
// prettier-ignore
const codeTemplate =
      (needsTypeScript ? 'typescript-' : '') +
      (needsRouter ? 'router' : 'default')
render(`code/${codeTemplate}`)

// Render entry file (main.js/ts).
if (needsPinia && needsRouter) {
  render('entry/router-and-pinia')
} else if (needsPinia) {
  render('entry/pinia')
} else if (needsRouter) {
  render('entry/router')
} else {
  render('entry/default')
}
```

*   main.j(t)s配置

```ts
// Render entry file (main.js/ts).
if (needsPinia && needsRouter) {
  render('entry/router-and-pinia')
} else if (needsPinia) {
  render('entry/pinia')
} else if (needsRouter) {
  render('entry/router')
} else {
  render('entry/default')
}
```

前面提到 base 目录中缺少 main.j(t)s 文件，这里给补上了。

*   ts 和 js 的差异化处理

```ts
// directoryTraverse.ts
function preOrderDirectoryTraverse(dir, dirCallback, fileCallback) {
  // 读取目录文件（同步）
  for (const filename of fs.readdirSync(dir)) {
    // 跳过.git文件
    if (filename === '.git') {
      continue
    }
    const fullpath = path.resolve(dir, filename)
    if (fs.lstatSync(fullpath).isDirectory()) {
      // 使用给定回调函数对文件夹进行处理
      dirCallback(fullpath)
      // in case the dirCallback removes the directory entirely
      // 递归调用方法前，先判断文件夹是否存在，避免文件被删除的情况
      if (fs.existsSync(fullpath)) {
        preOrderDirectoryTraverse(fullpath, dirCallback, fileCallback)
      }
      continue
    }
    fileCallback(fullpath)
  }
}

// We try to share as many files between TypeScript and JavaScript as possible.
// If that's not possible, we put `.ts` version alongside the `.js` one in the templates.
// So after all the templates are rendered, we need to clean up the redundant files.
// (Currently it's only `cypress/plugin/index.ts`, but we might add more in the future.)
// (Or, we might completely get rid of the plugins folder as Cypress 10 supports `cypress.config.ts`)
// 翻译一下：我们尝试在 TypeScript 和 JavaScript 之间复用尽可能多的文件。如果无法实现这一点，我们将同时保留“.ts”版本和“.js”版本。因此，在所有模板渲染完毕后，我们需要清理冗余文件。（目前只有'cypress/plugin/index.ts'是这种情况，但我们将来可能会添加更多。（或者，我们可能会完全摆脱插件文件夹，因为 Cypress 10 支持 'cypress.config.ts）
// 集成 ts 的情况下，对 js 文件做转换，不集成 ts 的情况下，将模板中的 ts 相关的文件都删除
if (needsTypeScript) {
    // Convert the JavaScript template to the TypeScript
    // Check all the remaining `.js` files:
    //   - If the corresponding TypeScript version already exists, remove the `.js` version.
    //   - Otherwise, rename the `.js` file to `.ts`
    // Remove `jsconfig.json`, because we already have tsconfig.json
    // `jsconfig.json` is not reused, because we use solution-style `tsconfig`s, which are much more complicated.
    // 将JS模板转化为TS模板，先扫描所有的 js 文件，如果跟其同名的 ts 文件存在，则直接删除 js 文件，否则将 js 文件重命名为 ts 文件
    // 直接删除 jsconfig.json 文件
    preOrderDirectoryTraverse(
      root,
      () => {},
      (filepath) => {
        // 文件处理回调函数：如果是 js 文件，则将其后缀变为 .ts 文件
        if (filepath.endsWith('.js')) {
          const tsFilePath = filepath.replace(/\.js$/, '.ts') // 先计算js文件对应的ts文件的文件名
          // 如果已经存在相应的 ts 文件，则删除 js 文件，否则将 js 文件重命名为 ts 文件
          if (fs.existsSync(tsFilePath)) {
            fs.unlinkSync(filepath)
          } else {
            fs.renameSync(filepath, tsFilePath)
          }
        } else if (path.basename(filepath) === 'jsconfig.json') { // 直接删除 jsconfig.json 文件
          fs.unlinkSync(filepath)
        }
      }
    )

// Rename entry in `index.html`
// 读取 index.html 文件内容
const indexHtmlPath = path.resolve(root, 'index.html')
const indexHtmlContent = fs.readFileSync(indexHtmlPath, 'utf8')、
// 将 index.html 中的 main.js 的引入替换为 main.ts 的引入
fs.writeFileSync(indexHtmlPath, indexHtmlContent.replace('src/main.js', 'src/main.ts'))
} else {
    // Remove all the remaining `.ts` files
    // 将模板中的 ts 相关的文件都删除
    preOrderDirectoryTraverse(
      root,
      () => {},
      (filepath) => {
        if (filepath.endsWith('.ts')) {
          fs.unlinkSync(filepath)
        }
      }
    )
}
```

最后是对集成 ts 的文件处理，这里依然是递归的对目录进行扫描，针对文件夹和文件传不同的回调函数，做不同的处理。

在模板里面，大部分 js 和 ts 文件是可以复用的，只需要修改名字即可，但某些文件，差异比较大，无法复用，同时保留了 js 文件和 ts 文件2个版本。在处理的时候，对应可复用文件，这里会按照是否集成 ts 对文件名进行修改，对不可复用文件，则会根据集成选项的不同，删除 对应 js 文件或 ts 文件。

*   readme.md 文件

```ts
// Instructions:
// Supported package managers: pnpm > yarn > npm
const userAgent = process.env.npm_config_user_agent ?? '' // process.env.npm_config_user_agent 获取当前执行的包管理器的名称和版本
const packageManager = /pnpm/.test(userAgent) ? 'pnpm' : /yarn/.test(userAgent) ? 'yarn' : 'npm'

// README generation
fs.writeFileSync(
    path.resolve(root, 'README.md'),
    generateReadme({
      projectName: result.projectName ?? result.packageName ?? defaultProjectName,
      packageManager,
      needsTypeScript,
      needsVitest,
      needsCypress,
      needsPlaywright,
      needsCypressCT,
      needsEslint
    })
)

// generateReadme.ts
// 针对不同的操作，根据包管理工具的不同，输出对应的命令，主要是区分 yarn 和 (p)npm 吧，毕竟 pnpm 和 npm 命令差不多
export default function getCommand(packageManager: string, scriptName: string, args?: string) {
  if (scriptName === 'install') {
    return packageManager === 'yarn' ? 'yarn' : `${packageManager} install`
  }

  if (args) {
    return packageManager === 'npm'
      ? `npm run ${scriptName} -- ${args}`
      : `${packageManager} ${scriptName} ${args}`
  } else {
    return packageManager === 'npm' ? `npm run ${scriptName}` : `${packageManager} ${scriptName}`
  }
}

// generateReadme.ts
import getCommand from './getCommand'

const sfcTypeSupportDoc = [
  '',
  '## Type Support for `.vue` Imports in TS',
  '',
  'TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin) to make the TypeScript language service aware of `.vue` types.',
  '',
  "If the standalone TypeScript plugin doesn't feel fast enough to you, Volar has also implemented a [Take Over Mode](https://github.com/johnsoncodehk/volar/discussions/471#discussioncomment-1361669) that is more performant. You can enable it by the following steps:",
  '',
  '1. Disable the built-in TypeScript Extension',
  "    1) Run `Extensions: Show Built-in Extensions` from VSCode's command palette",
  '    2) Find `TypeScript and JavaScript Language Features`, right click and select `Disable (Workspace)`',
  '2. Reload the VSCode window by running `Developer: Reload Window` from the command palette.',
  ''
].join('\n')

export default function generateReadme({
  projectName,
  packageManager,
  needsTypeScript,
  needsCypress,
  needsCypressCT,
  needsPlaywright,
  needsVitest,
  needsEslint
}) {
  const commandFor = (scriptName: string, args?: string) => getCommand(packageManager, scriptName, args)
  let readme = `# ${projectName}

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur) + [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin).
${needsTypeScript ? sfcTypeSupportDoc : ''}
## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).

## Project Setup
`

  let npmScriptsDescriptions = `\`\`\`sh
${commandFor('install')}
\`\`\`

### Compile and Hot-Reload for Development

\`\`\`sh
${commandFor('dev')}
\`\`\`

### ${needsTypeScript ? 'Type-Check, ' : ''}Compile and Minify for Production

\`\`\`sh
${commandFor('build')}
\`\`\`
`

  if (needsVitest) {
    npmScriptsDescriptions += `
### Run Unit Tests with [Vitest](https://vitest.dev/)

\`\`\`sh
${commandFor('test:unit')}
\`\`\`
`
  }

  if (needsCypressCT) {
    npmScriptsDescriptions += `
### Run Headed Component Tests with [Cypress Component Testing](https://on.cypress.io/component)

\`\`\`sh
${commandFor('test:unit:dev')} # or \`${commandFor('test:unit')}\` for headless testing
\`\`\`
`
  }

  if (needsCypress) {
    npmScriptsDescriptions += `
### Run End-to-End Tests with [Cypress](https://www.cypress.io/)

\`\`\`sh
${commandFor('test:e2e:dev')}
\`\`\`

This runs the end-to-end tests against the Vite development server.
It is much faster than the production build.

But it's still recommended to test the production build with \`test:e2e\` before deploying (e.g. in CI environments):

\`\`\`sh
${commandFor('build')}
${commandFor('test:e2e')}
\`\`\`
`
  }

  if (needsPlaywright) {
    npmScriptsDescriptions += `
### Run End-to-End Tests with [Playwright](https://playwright.dev)

\`\`\`sh
# Install browsers for the first run
npx playwright install

# When testing on CI, must build the project first
${commandFor('build')}

# Runs the end-to-end tests
${commandFor('test:e2e')}
# Runs the tests only on Chromium
${commandFor('test:e2e', '--project=chromium')}
# Runs the tests of a specific file
${commandFor('test:e2e', 'tests/example.spec.ts')}
# Runs the tests in debug mode
${commandFor('test:e2e', '--debug')}
\`\`\`
`
  }

  if (needsEslint) {
    npmScriptsDescriptions += `
### Lint with [ESLint](https://eslint.org/)

\`\`\`sh
${commandFor('lint')}
\`\`\`
`
  }

  readme += npmScriptsDescriptions
  return readme
}
```

生成 readme.md 的操作，主要是一些文本字符串的拼接操作，根据使用者的包管理工具的不同，生成不同的 readme.md 文档。经过前面的分析，再看这块，就觉得很简单了。

*   打印新项目运行提示

最后一部分是新项目运行提示，也就是当你的脚手架工程生成完毕后，告知你应该如何启动和运行你的脚手架工程。

```ts
console.log(`\nDone. Now run:\n`) // 打印提示
// 如果工程所在目录与当前目录不是同一个目录，则打印 cd xxx 指令
if (root !== cwd) {
  const cdProjectName = path.relative(cwd, root)
  console.log(
    `  ${bold(green(`cd ${cdProjectName.includes(' ') ? `"${cdProjectName}"` : cdProjectName}`))}`
  )
}
// 根据包管理工具的不同，输出不同的安装以来指令
console.log(`  ${bold(green(getCommand(packageManager, 'install')))}`)
// 如果集成了 prettier, 输出格式化指令
if (needsPrettier) {
  console.log(`  ${bold(green(getCommand(packageManager, 'format')))}`)
}
// 根据包管理工具的不同，输出启动脚手架工程的指令
console.log(`  ${bold(green(getCommand(packageManager, 'dev')))}`)
console.log()
// 结束
```

也就是如下图所示的脚手架工程运行指引。

![image-20230904222751185](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15a3a3bc0d8d48da9f0ce8f0298db98e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=709\&h=279\&s=45142\&e=png\&b=1d1d1d)

当项目名称带有空格时，需要带引号输出。

![image-20230904224601508](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6503adef86842bc858a534d02eb650d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=598\&h=298\&s=46373\&e=png\&b=1d1d1d)

到这里，关于 `create-vue` 的全部核心流程就都分析完毕了。