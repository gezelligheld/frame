#### data

用于声明组件的响应式状态，会返回一个对象，Vue 会将它转换为响应式对象，实例创建后通过 this.$data 访问

```js
export default {
  data() {
    return { a: 1 };
  },
  created() {
    console.log(this.a); // 1
    console.log(this.$data); // { a: 1 }
  },
};
```

#### methods

用于声明一个方法并挂载到 this 上

```js
export default {
  data() {
    return { a: 1 };
  },
  methods: {
    plus() {
      this.a++;
    },
  },
  created() {
    this.plus();
  },
};
```

#### 计算属性 computed

计算属性可以用来描述依赖响应式状态的复杂逻辑，计算属性值会基于其响应式依赖被缓存，仅会在其响应式依赖更新时才重新计算

```js
export default {
  data() {
    return {
      author: {
        name: 'John Doe',
        books: [
          'Vue 2 - Advanced Guide',
          'Vue 3 - Basic Guide',
          'Vue 4 - The Mystery',
        ],
      },
    };
  },
  computed: {
    publishedBooksMessage() {
      return this.author.books.length > 0 ? 'Yes' : 'No';
    },
  },
};
```

计算属性默认是只读的，也可以可写，一般只用来派生状态，而不用来改变状态或处理其他副作用

```js
export default {
  data() {
    return {
      firstName: 'John',
      lastName: 'Doe',
    };
  },
  computed: {
    fullName: {
      // getter
      get() {
        return this.firstName + ' ' + this.lastName;
      },
      // setter
      set(newValue) {
        [this.firstName, this.lastName] = newValue.split(' ');
      },
    },
  },
};
```

#### 侦听器 watch

用于在状态变化时执行一些副作用

```js
export default {
  data() {
    return {
      question: '',
      answer: 'Questions usually contain a question mark. ;-)',
    };
  },
  watch: {
    // 每当 question 改变时，这个函数就会执行
    question(newQuestion, oldQuestion) {
      if (newQuestion.includes('?')) {
        this.getAnswer();
      }
    },
    // 或者
    'some.nested.key'(newValue) {
      // ...
    },
  },
  methods: {
    async getAnswer() {
      this.answer = 'Thinking...';
      try {
        const res = await fetch('https://yesno.wtf/api');
        this.answer = (await res.json()).answer;
      } catch (error) {
        this.answer = 'Error! Could not reach the API. ' + error;
      }
    },
  },
};
```

watch 默认是浅层的，即被侦听的属性仅在被赋新值时才会触发，嵌套属性的变化是不会触发的，如果想侦听所有嵌套的变更需要深层侦听器

```js
export default {
  watch: {
    someObject: {
      handler(newValue, oldValue) {
        // todo
      },
      deep: true,
    },
  },
};
```

watch 默认仅当数据源变化时才触发，如下可以在创建侦听器时立即执行一遍回调

```js
export default {
  watch: {
    question: {
      handler(newQuestion) {
        // 在组件实例创建时会立即调用
      },
      // 强制立即执行回调
      immediate: true,
    },
  },
};
```

默认情况下，用户创建的侦听器回调，都会在 Vue 组件更新之前被调用，如下的设置可以在 Vue 组件更新之后被调用

```js
export default {
  watch: {
    key: {
      handler() {},
      flush: 'post',
    },
  },
};
```
