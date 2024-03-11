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