---
theme: ./theme
background-color: black
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
transition: slide-left
title: 基于 Vue3 的 Web 开发
mdc: true
---

# 基于 Vue3 的 Web 开发

<div>UMD-R&D 曾咏波 &nbsp 2023年12月xx日</div>

<style>
</style>

---
layout: center
---

# 你眼中的 Vue.js ？

<br />
<v-clicks>

  - 有手就行？
  - MVVM 模式

</v-clicks>

---



# Vue.js 的定位


<div class="absolute inset-0 flex justify-center items-center">
  <h1 style="color: white;"> 渐进式 Web 应用框架 </h1>
</div>

<!--  通俗来说，不同水平的开发者都可以利用 Vue.js 进行不同复杂程度的应用开发。门槛低，上限高，按开发者水平，按应用复杂度，可渐进的应用 Vue.js 的框架能力。 -->


---
layout: center
---

<v-clicks>

  - 无需构建步骤，渐进式增强静态的 HTML
  - 在任何页面中作为 Web Components 嵌入
  - 单页应用 (SPA)
  - 全栈 / 服务端渲染 (SSR)
  - Jamstack / 静态站点生成 (SSG)
  - 开发桌面端、移动端、WebGL，甚至是命令行终端中的界面
  
</v-clicks>


---
layout: intro
---

# 基于 Vue3 的 Web 开发

UMD-R&D

yongbo.zeng


---

# 渐进式Web开发框架

- 初始化一个 vue 项目
<br />

```ts {4,7}
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.mount('#app')
```

<!-- 这是一条备注 -->
<style>
</style>
---

# Vue 中的指令

- v-if 指令

```vue
  <template>
    <div>
      <h1 v-if="showA" @click="handleA">Hide A</h1>
      <h1 v-else @click="handleA">Show A</h1>
    </div>
  </template>

  <script setup lang="ts">
  import { ref, Ref } from 'vue'

  const showA: Ref<boolean> = ref(true)

  const handleA = () => {
    showA.value = !showA.value
  }
  </script>

  <style lang="scss" scoped>
  </style>
```

<!-- ./components/VifDemo.vue -->
<v-if-demo />
---

# Page 4

<v-clicks>

- Item 1
  - item 1.1
  - item 1.2
- Item 2
- Item 3
- Item 4

</v-clicks>

---
layout: iframe-left

# the web page source
url: https://cn.vuejs.org/

# a custom class name to the content
class: my-cool-content-on-the-right
---

# Vue.js

<v-click>

### 渐进式 Web 开发框架

<br />

</v-click>


<v-click>

### 组合式 API

</v-click>




---

# Vue.js 中的指令

- v-if
- v-for
- v-bind
- v-on
- v-slot
- v-html

---

# Vue.js 生命周期

- setup

- onMounted

- onUpdate

- onBeforeMount

- onBeforeUnmount


---

# Vue 生态

- vue-router

- vuex

- pinia

- vite

- vitest

- vue-devtools

- vueuse


---
layout: iframe-left

# the web page source
url: https://router.vuejs.org/zh/

# a custom class name to the content
class: my-cool-content-on-the-right
---

# vue-router

