# 渐进式框架

1. 单个模块使用
2. 单个页面使用
3. 整个项目使用

# API风格

## 选项式API（2）

- 可以用包含多个选项的对象来描述组件的逻辑，选项所定义的属性都会暴露在函数内部的 `this` 上，它会指向当前的组件实例

```html
<script>
export default {
  // data() 返回的属性将会成为响应式的状态
  // 并且暴露在 `this` 上
  data() {
    return {
      count: 0
    }
  },

  // methods 是一些用来更改状态与触发更新的函数
  // 它们可以在模板中作为事件处理器绑定
  methods: {
    increment() {
      this.count++
    }
  },

  // 生命周期钩子会在组件生命周期的各个不同阶段被调用
  // 例如这个函数就会在组件挂载完成后被调用
  mounted() {
    console.log(`The initial count is ${this.count}.`)
  }
}
</script>

<template>
  <button @click="increment">Count is: {{ count }}</button>
</template>
```

## 组合式API（3）

- 使用导入的 API 函数来描述组件逻辑
- 在单文件组件中，组合式 API 通常会与 `<script setup>` 搭配使用。这个 `setup` attribute 是一个标识，告诉 Vue 需要在编译时进行一些处理，让我们可以更简洁地使用组合式 API

```html
<script setup>
import { ref, onMounted } from 'vue'

// 响应式状态
const count = ref(0)

// 用来修改状态、触发更新的函数
function increment() {
  count.value++
}

// 生命周期钩子
onMounted(() => {
  console.log(`The initial count is ${count.value}.`)
})
</script>

<template>
  <button @click="increment">Count is: {{ count }}</button>
</template>
```

# 安装与使用

## 安装

- 下载node.js（版本大于16）`node -v 查看版本`
- 安装npm（注意与node版本兼容，一般node自带，但是不是最新版本）`npm -v 查看版本  npm install -g npm 升级`

## 使用

### 1.指定目录（cmd窗口）

### 2.使用命令进行安装



#### （1）npm命令

```
npm init vue@latest
```



#### （1）vue CLI

`npm install -g @vue/cli`



`vue create vue  my-project`