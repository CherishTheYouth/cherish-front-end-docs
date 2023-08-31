# Pinia

1. 使用 defineStore 创建一个 store, 每个 store 要设置一个唯一 id；

   ```ts
   
   import { defineStore } from 'pinia'
   import { ref } from 'vue'
   
   // useStore 可以是 useUser、useCart 之类的任何东西
   // 第一个参数是应用程序中 store 的唯一 id
   export const useMainStore = defineStore('main', {
     // other options...
       const name = ref('cherish')
   	const count = ref(0)
       
       return {
       	name,
           count
   	}
   })
   
   ```

2. 改变state 的值时不需要借助 mutation (pinia中没有mutation),  默认情况下，您可以通过 `store` 实例访问状态来直接读取和写入状态；

   ```ts
   
   const mainStore = useMainStore()
   mainStore.count++
   
   ```

3. 除了直接用 `store.count++` 修改 store，你还可以调用 `$patch` 方法同时应用多个更改; `$patch` 方法也接受一个函数来批量修改集合内部分对象的情况;

   ```ts
   
   const mainStore = useMainStore()
   mainStore.$patch({
     counter: mainStore.counter + 1,
     name: 'yb.z',
   })
   
   ```

4.  useMainStore 中的返回值，不能使用解构语法，否则会失去响应性，但是可以使用 storeToRefs 包裹 store，然后可以使用解构；

   ```ts
   
   import { storeToRefs } from 'pinia'
   
   const mainStore = useMainStore() // 正确
   const { name } = useMainStore() // 错误
   const { name } = storeToRefs(useMainStore()) // 正确
   
   ```

5. 通过调用 store 上的 `$reset()` 方法将状态 **重置** 到其初始值；

   ```ts
   
   const mainStore = useMainStore()
   mainStore.name = 'yb.z'
   mainStore.$reset() // mainStore.name cherish
   
   ```

6. 可以通过将其 `$state` 属性设置为新对象来替换 Store 的整个状态;

   ```ts
   
   const mainStore = useMainStore()
   mainStore.$state = { counter: 666, name: 'yb.z' }
   
   ```

7. 可以通过 store 的 `$subscribe()` 方法查看状态及其变化, **与常规的 `watch()` 相比，使用 `$subscribe()` 的优点是 *subscriptions* 只会在 *patches* 之后触发一次**;

8. 使用**getter** 和 直接在状态中使用 **computed** 计算的效果一样；

9. 如果一个 getter 依赖另一个 getter，通过 `this` 访问任何其他 getter；

   ```ts
   
   export const useStore = defineStore('main', {
     state: () => ({
       counter: 0,
     }),
     getters: {
       // 类型是自动推断的，因为我们没有使用 `this`
       doubleCount: (state) => state.counter * 2,
       // 这里需要我们自己添加类型（在 JS 中使用 JSDoc）。 我们还可以
       // 使用它来记录 getter
       /**
        * 返回计数器值乘以二加一。
        *
        * @returns {number}
        */
       doubleCountPlusOne() {
         // 自动完成 ✨
         return this.doubleCount + 1
       },
     },
   })
   
   ```

10. 同 computed 属性一样，getter 默认是不能传递参数进去的，但是，您可以从 *getter* 返回一个函数以接受任何参数，此时 *getter* 变成了一个普通函数，不再缓存数据。

11. 要使用其他 store 的 getter, 可以直接在一个 store 中引入和使用另一个 store;

    ```ts
    import { useOtherStore } from './other-store'
    
    export const useStore = defineStore('main', {
      state: () => ({
        // ...
      }),
      getters: {
        otherGetter(state) {
          const otherStore = useOtherStore()
          return state.localData + otherStore.data
        },
      },
    })
    ```

12. getter 的访问和 state 一模一样，都是 xxxStore.xxx;

13. 与 getter 一样，actions 操作可以通过 `this` 访问整个 store 实例，**不同的是，`action` 可以是异步的**，当然也可以是同步的，你可以在它们里面 `await` 调用任何 API，以及其他 action！可以把 action 理解为普通的方法；

14. 想要使用另一个 store 的action的话，可以直接在一个 store 中引入和使用另一个 store， 然后直接在 *action* 中调用就好了另一个 store 的action 就好了。

    ```ts
    import { useAuthStore } from './auth-store'
    
    export const useSettingsStore = defineStore('settings', {
      state: () => ({
        preferences: null,
        // ...
      }),
      actions: {
        async fetchUserPreferences() {
          const auth = useAuthStore()
          if (auth.isAuthenticated) {
            this.preferences = await fetchPreferences()
          } else {
            throw new Error('User must be authenticated')
          }
        },
      },
    })
    ```

    