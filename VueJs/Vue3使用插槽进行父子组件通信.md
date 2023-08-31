# Vue3使用插槽时的父子组件传值

用法见官方文档深入组件章节，插槽部分：

> 参考文档：[插槽-作用域插槽-插槽prop](https://v3.cn.vuejs.org/guide/component-slots.html)

## 作用域插槽

有时**让插槽内容能够访问子组件中才有的数据**是很有用的。

需求：插槽内容能够访问子组件中才有的数据

## 实现

### 子组件

TodoList.vue

```vue
<template>
  <div v-for="(todoItem, index) in state.todoList">
    <slot :item="todoItem" :index="index"></slot>
  </div>
</template>
<script setup>
import { reactive } from '@vue/reactivity'

const state = reactive({
  todoList: ['Feed a cat', 'Buy milk']
})
</script>
<style lang="less">
  
</style>
```

- 在子组件插槽上定义需要传递的属性，如上代码中的 `item` 和 `index` ;

- 子组件将子组件中定义的数据通过插槽属性传递给父组件;

  

### 父组件

useSlot.vue

```vue
<template>
  <div>
    <todo-list>
      <template v-slot:default="slotProps">
        <button @click="handleClick(slotProps)">{{slotProps.item}}</button>
      </template>
    </todo-list>
    <div>
      <h3>点击按钮</h3>
      <li>{{`${state.slotProps.index + 1}: ${state.slotProps.item}`}}</li>
    </div>
  </div>
</template>
<script setup>
import { reactive } from '@vue/reactivity'
import TodoList from './TodoList.vue'

const state = reactive({
  slotProps: {
    index: 0,
    item: 'default'
  }
})
const handleClick = (slotProps) => {
  state.slotProps = slotProps
}
</script>
<style lang="less">
  
</style>
```

- 父组件中定义插槽属性名字slotProps

  ```vue
  默认插槽
  <template v-slot:default="slotProps">
  ...
  
  当使用具名插槽时
  <template v-slot:other="slotProps">
  ...
  ```

- 属性slotProps获取子组件传递过来的插槽属性



### 注意：

- 属性只能在插槽内部才能获取
- 具名插槽写法

## 演示

![插槽父子组件通信](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/插槽父子组件通信.gif)

