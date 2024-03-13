vue3 中组合式 API 的出现，可以使用函数而不是声明选项的方式书写 Vue 组件

#### 状态 ref

ref函数可以声明响应式状态，当ref修改时触发追踪它的组件重新渲染。setup函数会将ref暴露给组件使用。

```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function increment() {
      count.value++
    }

    return {
      count,
      increment
    }
  }
}
```

也可以简写成以下形式

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}
</script>

<template>
  <button @click="increment">
    {{ count }}
  </button>
</template>
```

ref函数声明的状态具有深层响应性，如果不需要深层响应可以用shallowRef函数

#### 计算属性 computed

```vue
<script setup>
import { reactive, computed } from 'vue'

const author = reactive({
  name: 'John Doe',
  books: [
    'Vue 2 - Advanced Guide',
    'Vue 3 - Basic Guide',
    'Vue 4 - The Mystery'
  ]
})

const publishedBooksMessage = computed(() => {
  return author.books.length > 0 ? 'Yes' : 'No'
})
</script>

<template>
  <span>{{ publishedBooksMessage }}</span>
</template>
```

#### 侦听器

侦听器 watch 可以侦听的数据源包括ref、reactive以及它们相互间的组合，默认深层侦听

```vue
<script setup>
import { ref, watch } from 'vue'

const question = ref('')
const answer = ref('Questions usually contain a question mark. ;-)')
const loading = ref(false)

watch(question, async (newQuestion, oldQuestion) => {
  // ...
})
</script>
```

watchEffect 会自动跟踪回调的响应式依赖，它会在响应式数据变化时同步执行

```js
// todoId变化时自动执行
watchEffect(async () => {
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/todos/${todoId.value}`
  )
  data.value = await response.json()
})
```

#### 组合式函数

vue版自定义hooks

```js
// useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

// 按照惯例，组合式函数名以“use”开头
export function useMouse() {
  // 被组合式函数封装和管理的状态
  const x = ref(0)
  const y = ref(0)

  // 组合式函数可以随时更改其状态。
  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  // 一个组合式函数也可以挂靠在所属组件的生命周期上
  // 来启动和卸载副作用
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  // 通过返回值暴露所管理的状态
  return { x, y }
}

// index.js
<script setup>
import { useMouse } from './mouse.js'

const { x, y } = useMouse()
</script>

<template>Mouse position is at: {{ x }}, {{ y }}</template>
```

#### 相比于选项式API的优势

- 逻辑复用：选项式api利用mixins复用逻辑，但是存在数据来源不清晰、命名空间冲突等缺陷，组合式api借助组合式函数，更加简洁高效
- 代码组织：组合式中可以将相关逻辑放到一起或者移动到外部，而无需拘泥于选项式中的某个选项中。尽管选项式天然自带组织代码的能力，但对于逻辑复杂的场景下组合式更为灵活
- 类型推导：选项式在mixins、依赖注入时的类型推导不太理想，组合式下会更加简洁和友好

#### 和react hooks的区别

react hooks存在这样一些问题或限制

- hooks有严格的调用顺序，不可以在条件语句或循环语句中执行
- useEffect、useCallback等hooks中需要传递正确的依赖，才能在其内部获取到新的值（闭包问题）
- hooks所在的函数组件会随着上层状态或自身状态的变更多次执行，导致其中直接声明的值和函数会重新创建，其引用发生变化，如果作为props传递导致子组件的"rerender"（可能并不是真正的render，但多次执行了）

组合式api基于响应式模型下避免了这些问题

- 不限制调用顺序
- Vue 的响应性系统运行时会自动收集计算属性和侦听器的依赖，无需手动声明
- setup下的代码只会执行一次，因此无需特意进行缓存来避免不必要的组件更新
