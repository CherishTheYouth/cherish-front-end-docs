**摘要：**

本系列主要从小白视角入手，探讨 `create-vue` 脚手架的实现细节，共分为五个部分，本文是该系列的第二篇——执行篇。

本文将主要讲解脚手架是如何执行的。

[create-vue源码解析（一）准备工作 - 掘金 (juejin.cn)](https://juejin.cn/post/7280008213671493647#heading-7)

## 1. 脚手架命令是怎么使用的？🤔️

根据vue3 官方文档，create-vue 的用法如下：

```
npm create vue@3
```

![image-20230727192225526](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f90b1b53f424319bdc931036cf6f3cb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=684&h=499&s=84561&e=png&b=011727)

## 2. 执行 npm create 命令时发生了什么？

我们知道，常见的npm命令有 `npm init`,`npm run`,`npm install`等，但是 `create` 命令很少见，这里我们先看下，运行`npm create`会发生什么：

以下内容摘自参考资料：

> 📖 参考资料：[npm create vite“ 是如何实现初始化 Vite 项目？](https://blog.csdn.net/Cyj1414589221/article/details/128191826)
>
> **npm `init` / `create` 命令**
>
> npm v6 版本给 `init` 命令添加了别名 `create`，俩命令是一样的.
>
> `npm init` 命令除了可以用来创建 `package.json` 文件，还可以用来执行一个包的命令；它后面还可以接一个 `<initializer>` 参数。该命令格式：
>
> ```
> npm init <initializer>
> ```
>
> 参数 `initializer` 是名为 `create-<initializer>` 的 `npm` 包 ( 例如 `create-vite` )，执行 `npm init <initializer>` 将会被转换为相应的 `npm exec` 操作，即会使用 `npm exec` 命令来运行 `create-<initializer>` 包中对应命令 `create-<initializer>`（**package.json 的 bin 字段指定**），例如：
>
> ```
> # 使用 create-vite 包的 create-vite 命令创建一个名为 my-vite-project 的项目
> $ npm init vite my-vite-project
> # 等同于
> $ npm exec create-vite my-vite-project
> ​
> ```
>
> **执行 `npm create vite` 发生了什么？**
>
> 当我们执行 `npm create vite` 时，会先补全包名为 `create-vite`，转换为使用 `npm exec` 命令执行，即 `npm exec create-vite`，接着执行包对应的 `create-vite` 命令（如果本地未安装 `create-vite` 包则先安装再执行）.

根据参考资料，在执行 `npm create vue` 时，会先补全包名为 `create-vue`, 然后转换为使用 `npm exec` 命令执行 `create-vue` 这个依赖包。


所以说，我们在执行命令时，npm 会执行`create-vue` 这个包，来运行 cli ,搭建工程。

具体的执行过程如下：

-   `npm create vue` 转化 `npm exec create-vue` 命令，执行 `create-vue` 包；
-   执行 `create-vue` 具体是执行啥？执行 package.json 中bin 字段对应的 .js 文件。

![image-20230910142157965](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89d0ec0fd29848bc936d3cd7a1f38172~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=456&h=214&s=22296&e=png&b=fefdfd)

```
{
   "name": "create-vue",
  "version": "3.6.4",
  "description": "An easy way to start a Vue project",
  "type": "module",
  "bin": {
    "create-vue": "outfile.cjs"
  }
}
```

这是 package.json 中的部分代码，运行以上命令，就对应会执行 `outfile.cjs` 文件，其最终的结果即：

```
node outfile.cjs
```

如下图，在`create-vue`的包中直接执行`node outfile.cjs` , 执行结果跟执行脚手架一致，这证明了以上的结论正确。

![image-20230910142251313](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42eb200490a140398f0634167bc07103~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=845&h=264&s=29968&e=png&b=0f0f0f)

## 3. 执行文件 outfile.cjs 文件从何而来❓

outfile.cjs 从何而来呢？outfile.cjs是打包后的文件，这是显然的。我们找到脚手架打包的位置，查看一下它的入口文件即可。

根据 script 命令，打包命令`"build": "zx ./scripts/build.mjs",` 指向 `build.mjs`

```
// build.mjs
await esbuild.build({
  bundle: true,
  entryPoints: ['index.ts'],
  outfile: 'outfile.cjs',
  format: 'cjs',
  platform: 'node',
  target: 'node14',
    ...
```

从以上的打包配置代码可以看到，打包的入口文件就是 `index.ts` ， `outfile.cjs` 是由 `index.ts` 打包后生成的输出文件。

## 4. 总结
以上就是 `create-vue` 脚手架的执行过程了，根据上面的分析，我们可以找到分析其实现的核心文件，即 `index.ts`.
    
所以，我们后续的分析将主要集中在 `index.ts`文件。
    
## 原文地址 + 详细注释版源码地址

[create-vue-code-analysis](https://github.com/CherishTheYouth/create-vue-code-analysis)

😊 如果觉得还可以的话，可以给个🌟吗，嘿嘿嘿！
    