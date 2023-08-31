# How to construct A UI Lib

## 1. pnpm init

## 2. install vite

```bash
pnpm i vite -D
```

## 3. test vite and typescript

1. add a new file index.html with blow codes:

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
     <h2>Hello UP</h2>
   </body>
   </html>
   
   ```

2. run `npx vite` or `node_modules/.bin/vite`

   1. access the address: [access address: http://127.0.0.1:5173](http://127.0.0.1:5173)

   2. show the index.html page

   3. you have installed vite successfully

      ![image-20230224123551578](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230224123551578.png)

3. add a new file index.ts with blow codes:

   1. add file index.ts
   2. write blow codes
   3. import it into index.html
   4. run `npx vite` and access the address
   5. open the console, and if you see the result: `greeting:hello Up`, your typescript runs well.

   ```js
   // index.ts
   const greeting: string = 'hello Up'
   console.log('greeting:', greeting)
   ```

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
     <h2>Hello UP</h2>
   </body>
   </html>
   <script src="./index.ts" type="module"></script>
   
   ```

   ![image-20230224124351588](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230224124351588.png)

​				

## 4. add vite dev script in package.json:

```json
{
  "name": "up-ui-alpha",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
      "dev": "vite"// here to add
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "vite": "^4.1.4"
  },
  "dependencies": {
    "vue": "^3.2.47"
  }
}

```



## 5. configure and test vue environment

- install vue: `pnpm i vue`

- configure vite support to compile vue templates. (NOTE： The default package of Vue 3.x does not support template compilation.)

  > 1. install the vue plugin of vite (Plugin Name: @vitejs/plugin-vue)
  >
  > ```bash
  > pnpm i @vitejs/plugin-vue -D
  > ```
  >
  > 2. add the blow config file named vite.config.ts in root directory:
  >
  >    ```ts
  >    import { defineConfig } from "vite"
  >    import vue from "@vitejs/plugin-vue"
  >    
  >    export default defineConfig({
  >      plugins: [vue()]
  >    })
  >    
  >    ```
  >
  > 3. a ts warnning to heed: `找不到模块“./UpTitle/index.vue”或其相应的类型声明。` 
  >
  >    TypeScript  does not support modules of vue type, so you should add a module type defination file (named shims-vue.d.ts )to solve this problem. You may write that like the below codes:
  >
  >    ```ts
  >    // location: src/shims-vue.d.ts
  >    declare module '*.vue' {
  >      import { DefineComponent } from "vue";
  >      const component: DefineComponent<{}, {}, any>
  >      export default component
  >    }
  >    ```

- write a vue component do not use template:

  ```ts
  // location: src/UpButton/index.ts
  import { defineComponent, h } from "vue";
  export default defineComponent({
    name: 'up-button',
    props: {},
    render () {
      return h(
        'button',
        {
          id: 'up-button-container',
          name: 'up-button'
        },
        '普通按钮'
      )
    }
  })
  ```
  
- write a vue component use template:

  ```vue
  <!--location: src/UpTitle/index.vue -->
  <template>
    <div class="up-title-container">
      <slot></slot>
    </div>
  </template>
  <script setup lang="ts">
  </script>
  <style lang="" scoped>
  </style>
  
  ```

## 6. Support Full Import

### 6.1 What is Full Import?

> Import all components at once, in the form of Vue Plugin using Vue.use()

### 6.2 How to relalize?

Follow the below steps"

- Create a entry file of your components.
- Import your components into entry.ts.
- Export a vue plugin which installed all of your components.

The following file is an example:

```js
// location: src/entry.ts
import { App } from "vue"
import UpButton from "./UpButton"
import UpInput from "./UpInput"
import UpWrapper from "./UpWrapper"
import UpTitle from './UpTitle/index.vue'

export {
  UpButton,
  UpInput,
  UpWrapper,
  UpTitle
}

export default {
  install: (app: App): void => {
    app.component(UpButton.name, UpButton)
    app.component(UpInput.name, UpInput)
    app.component(UpWrapper.name, UpWrapper)
    app.component(UpTitle.name, UpTitle)
  }
}
```

## 7. Support Importing on demand

### 7.1 What is Importing on demand?

> Export individual components, people can just import the component they need, but not import all components of your lib.

### 7.2 How to relalize?

- Create a entry file of your components.
- Import your components into entry.ts.
- Export components separately.

​	The following file is an example:

```js
// location: src/entry.ts
import { App } from "vue"
import UpButton from "./UpButton"
import UpInput from "./UpInput"
import UpWrapper from "./UpWrapper"
import UpTitle from './UpTitle/index.vue'

// Importing on-demand
export {
  UpButton,
  UpInput,
  UpWrapper,
  UpTitle
}

// Full Importing
export default {
  install: (app: App): void => {
    app.component(UpButton.name, UpButton)
    app.component(UpInput.name, UpInput)
    app.component(UpWrapper.name, UpWrapper)
    app.component(UpTitle.name, UpTitle)
  }
}
```

## 8. Test your components

- Import your components into `index.ts`.

- Create a vue app, and mount it to a div(id: 'app').

  ```vue
  <!--App.vue-->
  <template lang="">
    <div>
      <up-button>普通按钮</up-button>
      <up-input>输入框</up-input>
      <up-title>标题</up-title>
    </div>
  </template>
  <script setup lang="ts">
  </script>
  <style lang="" scoped>
  </style>
  ```

  ```js
  // index.ts
  import { createApp } from "vue";
  import entryPlugin from "./src/entry"
  import { UpButton, UpInput, UpWrapper, UpTitle } from "./src/entry"
  import App from './App.vue'
  
  const app = createApp(App)
  app.use(entryPlugin)
  app.mount('#app')
  ```

- import index.ts into index.html.

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
    <div id="app"></div>
  </body>
  </html>
  <script src="./index.ts" type="module"></script>
  ```

  

- run `pnpm dev` or `yarn dev` to run your project.

  ![image-20230225142016180](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230225142016180.png)

- run `pnpm build` to build your project.

  ![image-20230225142059171](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230225142059171.png)

- run `pnpm preview` to preview the built dist file.

![image-20230225142110019](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230225142110019.png)

![image-20230225142121658](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230225142121658.png)

## 9. How to build the components into Lib files?

If you want to export your components as a lib, you need to configure the type of export module and export file name.

The following file is an example:

```js
// vite.config.ts
import { defineConfig } from "vite"
import vue from "@vitejs/plugin-vue"

export default defineConfig({
  plugins: [vue()],
  build: {
    rollupOptions: {
      external: ['vue'],
      output: {
        globals: {
          vue: 'Vue'
        }
      }
    },
    minify: false,
    lib: {
      entry: './src/entry.ts',
      name: 'Up UI',
      fileName: 'up-ui',
      formats: ['es', 'umd', 'cjs', 'iife'] // four type allowed: 'es', 'umd', 'cjs', 'iife'
    }
  }
})

```

Build and Test the Lib:

- yarn build;

- View the dist file;

  ![image-20230225150455577](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230225150455577.png)

- Import components from the lib files;

  ```js
  // index.ts
  import { createApp } from "vue";
  import UpUI, { UpButton, UpInput, UpWrapper, UpTitle } from './dist/up-ui.mjs'
  import App from './App.vue'
  
  createApp(App)
  .use(UpUI)
  .mount('#app')
  ```

  ```html
  <!-- index.html-->
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
  </body>
  </html>
  <script src="./index.ts" type="module"></script>
  ```

  ```vue
  <!--src/App.vue-->
  <template lang="">
    <div>
      <up-button>普通按钮</up-button>
      <up-input>输入框</up-input>
      <up-title>标题</up-title>
    </div>
  </template>
  <script setup lang="ts">
  </script>
  <style lang="" scoped>
  </style>
  ```

![	](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230227122148299.png)

- yarn dev and view if the components show normally;

![image-20230227122537535](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/image-20230227122537535.png)

## 10. Css Style

### 10.1 Atomic/Utility-First CSS and Semantic CSS

- What is Semantic CSS?
- What is Atomic/Utility-First CSS?
- UnoCss
- Why is UnoCss?

### 10.2 How to relalize?

1. #### Install and Configure UnoCss.

   - Install `unocss`(unocss lib) and `iconify-json/ic` (a icon lib);

     ```bash
     pnpm i unocss -D
     ```

     ```bash
     pnpm i @iconify-json/ic -D
     ```

   - Configure UnoCss Plugin in vite.config.ts.

2. #### How to Use UnoCss?

   - 
