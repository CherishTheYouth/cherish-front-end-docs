# create-vue源码解析（一）准备工作

**摘要：**

本系列主要从小白视角入手，探讨 `create-vue` 脚手架的实现细节，共分为五个部分，本文是该系列的第一篇——准备篇。

本文将主要讲解一个小白在学习源码前需要做的一些准备工作。

## 

## 1. 获取源码 🐯

想要系统的学习源码，当然需要先有一份源码。

所以，我们需要先从官方仓库下载一份官方的源码下来。可以 `clone` 到本地，但我建议最好还是直接 `fork` 一份到自己的 `github` 账户。

毕竟，自己电脑上的资料总有丢失的风险，随时把学习资料同步到 `github` 是一个更好的选择。同时，也可以把自己的学习心得开源出来，帮助他人。最后，对于小小白，平时多多使用 `git` ,对以后的工作也是帮助不小。

### 1.1 源码地址

![image-20230918192900168](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230918192900168.png)

[create-vue](https://github.com/vuejs/create-vue)

> https://github.com/vuejs/create-vue

### 1.2 源码版本

>  3.6.4

### 1.3 目录结构

![image-20230910112302032](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230910112302032.png)

## 2. 用途

既然要分析，总要清楚工具的作用。

`create-vue` 是一款 `vue` 脚手架。是 `vue` 官方脚手架 `vue/cli` 的替代者。

脚手架是建筑行业用来协助搭建工程的工具，类推过来，`create-vue` 是用来协助vue开发人员快速搭建基于vue技术栈前端工程的工具。使用 `create-vue` 可以基于开发者的需要，一站式生成各类不同配置的vue工程。

## 3. 依赖分析👀

如果希望系统分析源码，建议先看看整个项目中用到了那些依赖，那些技术点，都是干啥的？

对于一些不了解的依赖，可以提前了解一下。也可以通过依赖大致分析出项目的解决思路。

以下是 `create-vue` 的 `package.json` 文件。

```json
{
  "name": "create-vue",
  "version": "3.6.4",
  "description": "An easy way to start a Vue project",
  "type": "module",
  "bin": {
    "create-vue": "outfile.cjs"
  },
  "files": [
    "outfile.cjs",
    "template"
  ],
  "engines": {
    "node": ">=v16.20.0"
  },
  "scripts": {
    "prepare": "husky install",
    "format": "prettier --write .",
    "build": "zx ./scripts/build.mjs",
    "snapshot": "zx ./scripts/snapshot.mjs",
    "pretest": "run-s build snapshot",
    "test": "zx ./scripts/test.mjs",
    "prepublishOnly": "zx ./scripts/prepublish.mjs"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/vuejs/create-vue.git"
  },
  "keywords": [],
  "author": "Haoqun Jiang <haoqunjiang+npm@gmail.com>",
  "license": "MIT",
  "bugs": {
    "url": "https://github.com/vuejs/create-vue/issues"
  },
  "homepage": "https://github.com/vuejs/create-vue#readme",
  "devDependencies": {
    "@tsconfig/node18": "^2.0.1", 
    "@types/eslint": "^8.37.0",
    "@types/node": "^18.16.8",
    "@types/prompts": "^2.4.4",
    "@vue/create-eslint-config": "^0.2.1",
    "@vue/tsconfig": "^0.4.0",
    "esbuild": "^0.17.18",
    "esbuild-plugin-license": "^1.2.2",
    "husky": "^8.0.3",
    "kolorist": "^1.8.0",
    "lint-staged": "^13.2.2",
    "minimist": "^1.2.8",
    "npm-run-all": "^4.1.5",
    "prettier": "^2.8.8",
    "prompts": "^2.4.2",
    "zx": "^4.3.0"
  },
  "lint-staged": {
    "*.{js,ts,vue,json}": [
      "prettier --write"
    ]
  }
}

```

下面通过下表一句话简单介绍一下每个依赖的作用：

| 依赖名                    | 功能                                                         |
| ------------------------- | ------------------------------------------------------------ |
| @tsconfig/node18          | node.js18配套的tsconfig                                      |
| @types/eslint             | eslint相关                                                   |
| @types/node               | node.js的类型定义                                            |
| @vue/create-eslint-config | 在Vue.js项目中设置ESLint的实用程序。                         |
| @vue/tsconfig             | 用于Vue项目的TS Configure扩展。                              |
| esbuild                   | JavaScript 和 golang 打包工具，在打包时使用，后文会具体分析，建议预先了解，尤其是 esbuild.build 函数的相关 Api |
| esbuild-plugin-license    | 许可证生成插件，用于在打包时，生成 LICENSE 文件              |
| husky                     | git 钩子，代码提交规范工具                                   |
| kolorist                  | 给stdin/stdout的文本内容添加颜色，建议预先了解。             |
| lint-staged               | 格式化代码                                                   |
| minimist                  | 解析参数选项， 当用户从 terminal 中输入命令指令时，帮助解析各个参数的工具，建议预先了解。 |
| npm-run-all               | 一个CLI工具，用于并行或顺序运行多个npm-script。              |
| prettier                  | 代码格式化                                                   |
| prompts                   | 轻巧、美观、人性化的交互式提示。在 terminal 中做对话交互的。建议预先了解。 |
| @types/prompts            | prompts 库的类型定义                                         |
| zx                        | 该库为开发者在JavaScript中编写shell脚本提供了一系例功能，使开发更方便快捷，主要在编写复杂的 script 命令是使用。只要在打包，快照，预发布阶段编写脚本时使用，建议预先了解。 |

可以看到，这16个依赖，真正和cli功能紧密相关的应该是以下几个：

- esbuild
- kolorist
- minimist
- prompts
- zx

所以，学习 create-vue 的之前，可以先阅读以下几个库的基本使用方法，以便在阅读源码过程中，遇到有知识点盲区的，可以定向学习。

## 4. 项目目录结构及功能模块简介 🪜

以下是 `create-vue` 源码的工程目录结构：

```txt
F:\Study\Vue\Code\VueSourceCode\create-vue
├── CONTRIBUTING.md
├── index.ts // 主文件
├── LICENSE
├── media
├── package.json
├── playground
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── README.md
├── renovate.json
├── scripts
|  ├── build.mjs // esbuild 打包脚本
|  ├── prepublish.mjs // 预发布脚本
|  ├── snapshot.mjs // 生成快照脚本
|  └── test.mjs
├── template // 各类模板
|  ├── base
|  ├── code
|  ├── config
|  ├── entry
|  ├── eslint
|  └── tsconfig
├── tsconfig.json
└── utils
   ├── banners.ts // 终端中彩色提示文字输出
   ├── deepMerge.ts // 工具函数，对文件内容进行深度合并，用于 cli 执行过程中对各template 的依赖进行合并
   ├── directoryTraverse.ts // 文件夹递归操作
   ├── generateReadme.ts // 生成工程的 readme 文件
   ├── getCommand.ts // 
   ├── renderEslint.ts // 生成 eslint 配置文件
   ├── renderTemplate.ts // 生成通用模块模板
   └── sortDependencies.ts // 对 package.json 中的依赖进行分类，dependency devDependency 等
```

可以看到，除开一些配置文件和空文件夹后，真正用到的文件就以下几个部分：

- husky

> 这个是一款git hook 工具，支持在开发人员进行 `git` 操作时，根据不同需求，触发一些自定义动作，通常用于团队工作中对代码提交流程进行约束和规范。

- scripts

> 这个目录下主要是项目打包，快照，预发布，单元测试这几块内容的 `script` 脚本. 在本系列第四个部分将详解这部分内容。

- template

> 这个里面是这个库的核心部分了——模板。我们 cli 在执行时，就是会从这个文件夹中读取各种配置的代码实现，最后把这些代码组合成一个完整的项目，然后给用户快速生成一个项目模板。
>
> 这个模板只要你事先预制好即可。

- utils

> 项目中用到的一些工具方法，可在用到的时候再具体去分析。

- **index.ts**

> 项目的入口文件，也是核心文件，包含了cli 执行的所有逻辑，是create-vue脚手架的核心实现。

以上就是项目中一些具体模块的大致功能了，最最核心的还是 `index.ts ` 文件。



## 原文地址 + 详细注释版源码地址

[create-vue-code-analysis](https://github.com/CherishTheYouth/create-vue-code-analysis)

😊 如果觉得还可以的话，可以给个🌟吗，嘿嘿嘿！