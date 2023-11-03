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
- 安装npm（注意与node版本兼容，一般node自带，但是不是最新版本）`npm -v 查看版本 npm install -g npm 升级`

## 使用

### 1.指定目录（cmd窗口）

### 2.使用命令进行安装



#### （1）npm命令

```
npm init vue@latest
```

- 这是 Vue CLI 2 的方式，用于创建 Vue.js 3 项目。
- 在使用 `vue init` 时，您需要选择一个预定义的模板（例如 webpack、simple、pwa）来创建项目，然后通过交互式命令行选项进行配置。
- 它生成一个包含配置文件的项目，并且配置文件通常存储在项目根目录下的 `build` 和 `config` 目录中。
- 它通常使用 Webpack 4 作为构建工具。

#### （1）vue CLI

npm install -g @vue/cli

vue create vue my-project

- 这是 Vue CLI 3 的方式，用于创建 Vue.js 3 项目。
- 在使用 `vue create` 时，您可以选择一个预设配置（如默认、Manually select features、TypeScript 等）或自定义配置。
- 它生成一个包含配置的项目，配置文件通常存储在项目根目录下的 `vue.config.js` 中。
- 它使用 Vue CLI 3 的默认构建工具，通常使用 Webpack 4 或更新版本

## 3.目录结构

- .vscode ：vscode工具的配置文件
- node_modules：vue项目的运行依赖文件夹
- public：资源文件夹
- src：源码文件夹
- .gitignore：git忽略文件
- index.html：入口html文件
- package.json：信息描述信息
- README.md：注释文件
- vite.config.js：vue配置文件

# 模板语法

vue模板都是语法层面的html，能被符合规范的浏览器和HTML解析器解析

## 1.文本插值

### 支持单一表达式（不支持语句）

简单来说按一下书写即可将值放在标签内`{{msg}}`，其次也可以对`{{}}`内进行键值操作,如`{{msg.split("").reverse().join("")}}`

```html
<template>
<p>{{msg}}</p>
</template>
<script>
export default{
  data(){
    return{
      msg:"文本插值",
    }
  }
}
</script>
```

### 原始HTML

{{}}会将数据插值为纯文本，而不是HTML，若想插入HTML需要使用v-html指令

```html
<template>
<p>{{msg}}</p>  //会直接展示 <a href="测试">测试</a>
<p v-html="msg"></p>   //会展示 测试
</template>
<script>
export default{
  data(){
    return{
      msg:"<a href="测试">测试</a>"
    }
  }
}
</script>
```

## 2.属性绑定

{{}}不能直接用作响应式的绑定attribute，用v-bind指令

v-bind

1. 指示vue将元素的idttribute)与组件dynamicid属性保持一致
2. 如果绑定的值是null或者undefined，那么该attribute将会从渲染的元素上移除
3. v-bind可以简写，即直接加 ：就行 ，如`:class`

```html
<template>
<div id="msg" class="msg1">APP</div>     //这样的话id = msg    class = msg1
<div v-bind:id="msg" v-bind:class="msg1">APP</div>    //他会把msg解析成 appid   把msg1解析成appclass
<div vibind="msg2">APP</div>  //与以上解析一样，用于动态绑定多个值
</template>
<script>
export default{
  data(){
    return{
      msg:"appid",
      msg1:"appclass"
      msg2:{
        id:"appid",
        class:"appclass"
      }
    }
  }
}
</script>
```

## 3.条件渲染

- v-if：条件性的渲染一条内容，这块内容只有在if为真时才会被渲染
- v-else：与v-if配合使用
- v-else-if：多个条件的时候用
- v-show：按条件显示一个元素的指令
  - 与v-if比较
  - v-if是惰性渲染，在切换时相关组件会被切换和重建，适用于绑定条件多但是切换少的场景
  - v-show渲染的元素始终会被渲染，只有css的display属性会被切换，适用于频繁切换的场景

## 4.列表渲染

- v-for：基于item in items形式的特殊语法，items是源数据的数组，而item是迭代项的别名
  - 支持使用可选的第二个参数表示当前项的位置索引
  - 可以用of代替in
  - 遍历对象

```html
<div>
  <p v-for="item in items">{{item}}</p>
  <p v-for="(item,index) in items">{{index}}:{{item}}</p>
  <p v-for="(item,key,index) of items">{{item}} </p>
</div>
```

### 通过key管理状态

- vue默认就地跟新策略（当列表元素进行改变之后vue会将整个列表进行重新渲染）
- 简单操作就是在标签后加上 `:key={{index}}`
- 一般项目推荐使用网络请求的唯一id

## 5.事件处理

v-on指令（`v-on:click="方法名"`或`@click="方法名"`）

### （1）内联事件处理器：事件被触发时执行的内联JavaScript语句

常用来处理简单场景

简单来说就是标签直接通过指令在标签内部进行事件处理

### （2）方法事件处理器：一个指向组件上定义的方法的属性名或是路径

方法内部可以通过this来读取到data中的数据

```html
<template>
<div @onclick="方法名" class="msg1">APP</div>     //这样的话id = msg    class = msg1
</template>
<script>
export default{
  data(){
    return{
      msg:"appid",
      msg1:"appclass"
      msg2:{
        id:"appid",
        class:"appclass"
      }
    }
  },
  methods:{
   方法名(接收参数){具体操作} 
  }
}
</script>
```

### (3)传参

- 在方法获取参数处获取event对象` 方法名(e){具体操作}`
- 传递参数
  - 在onclick方法名后加上`()`，括号内即为传递的参数，在再方法接收处加上接收参数即可
  - 加上$event就可以传递event对象

## 6.事件修饰符

针对v-on提供了事件修饰符

- .stop（阻止事件冒泡）
- .prevent（阻止默认事件）
- .once（阻止一次性事件）
- .enter（阻止键盘事件）

### (1)阻止事件

下面这两种方式均能阻止页面点击后跳转

```html
<template>
<a @click.prevent="clickHandle" href="https://***">测试</a>
<a @click="clickHandle1" href="https://***">测试</a>
</template>
<script>
export default{
  data(){
    return{
      msg:"appid",
      msg1:"appclass"
      msg2:{
        id:"appid",
        class:"appclass"
      }
    }
  },
  methods:{
   clickHandle1(e){e.preventDefault();} 
  }
}
</script>
```

### （2）事件冒泡

```html
<template>
<div @click="clickTest1">
<p @click="clickTest2">测试</p>    // <p @click.stop="clickTest2">测试</p>
</div>

</template>
<script>
export default{
  data(){
    return{
      msg:"appid",
      msg1:"appclass"
      msg2:{
        id:"appid",
        class:"appclass"
      }
    }
  },
  methods:{
   clickTest1(){},
   clickTest2(e){e.stopPropagation();} 
  }
}
</script>
```

这时如果没有加以限制，那么两个方法均会被触发，加了阻止事件之后可以选择触发

## 7.数组变换侦听

vue能够侦听响应式数组的变更方法，并在被调用时自动更新

- push（）
- pop（）
- shift（）
- unshift（）
- splice（）
- sort（）
- reverse（）

对于某些方法页面UI不会自动更新（主要是方法本身不是更新原数组而是返回新数组）

- filter（）
- concat（）
- slice（）

## 8.计算属性

避免模板语法里面有过多复杂计算代码

```
<template>
<p>{{Test}}</p>  
</div>

</template>
<script>
export default{
  data(){
    return{
     
      }
    }
  },
 //存放计算属性
  computed:{
      Test(){
         return 加具体条件;    //注意用到data里面的值时用到this
      }
   }
}
</script>
```

### 与方法的区别（调用方法也能实现如上结果）

计算属性：计算属性值会基于响应式依赖被缓存，一个计算属性仅在其响应式依赖更新时才重新计算

方法：方法调用总是会在渲染时发生时再次执行函数

## 9.class绑定

class进行了增强，v-bind里面可以绑定数组（数组指定对应的css样式），对象（标签指定对应的样式，通过true或false来判断是否展示）

哪个为true就展示哪个的效果

```html
<template>
<p :class="{'active':isActive,'text-danger':hasError}">测试</p>  

<p :class="{"classObject"}">测试2</p>   //绑定多个对象

<p :class="[arrActive,arrHasError]">测试3</p>   //数组

<p :class="[isActive ? 'active' : '']">测试4</p>
</div>

</template>
<script>
export default{
  data(){
    return{
       isActive:true,
       hasError:true,
       classObject:{
          'active':true,
           'text-danger':true
       }
        arrActive:"active",
        arrHasError:"text-danger"
      }
    }
  },
 //存放计算属性
  computed:{
      Test(){
         return 加具体条件;    //注意用到data里面的值时用到this
      }
   }
}
</script>
<style>
.active{
font-size:10px;
}
.text-danger{
color:red;
}
</style>
```

> 数组和对象嵌套过程中，只能是数组嵌套对象

## 10.style绑定

- 绑定对象
- 绑定数组（一般不用）

```html
<template>
<p :style="{"color:activeColor,fontSize:fontSize+'px'"}">测试1</p>
<p :style="{"classObject"}">测试2</p> 
</div>
</template>
<script>
export default{
  data(){
    return{
       activeColor:"green",
       fontSize:30,
       classObject:{
          color:"red",
           fontSize:"30px"
       }
      }
    }
  },

</script>
```

## 11.侦听器

可以选择使用watch来监听响应式数据

```html
<template>
<div @click="clickTest1">
<p @click="clickTest2">测试</p>    
</div>

</template>
<script>
export default{
  data(){
    return{
      msg:"appid",
    }
  },
  methods:{
   clickTest1(){
      this.msg= "appid222"
    },
  }，
   watch:{
    message(newValue,oldValue)
    //可以取到新旧值，数据发生变换时自动执行的函数
   }
}
</script>
```

## 12.表单数据绑定

在前端处理表单时，v-model可以帮我们手动简化表单输入内容同步到js中对应变量的过程

```
v-model="变量名"
<template>
<form>
 <input type="text" v-model="message">
 <p>{{ message }}</p>
 <input type="checkbox" id="checkbox" v-model="checked" />
 <label for="checkbox"> {{ checked }} </label>
</form>
</div>

</template>
<script>
export default{
  data(){
    return{
      message:"",
      checked:false
    }
  },
 
}
</script>
```

修饰符

- .lazy
  - `v-model.lazy：`回车/失去焦点后才会获得
- .number
  - `v-model.number`：只接收数字
- .trim
  - `v-model.trim：`区分空格

## 13.模板引用

可以通过操作DOM获取相关操作

标签内使用 `ref="元素名"` 方法内使用 `this.$refs.元素名.相关操作`

# 组件

组件可以扩展 HTML 元素，封装可重用的代码。在较高层面上，组件是自定义元素， Vue.js 的编译器为它添加特殊功能。在有些情况下，组件也可以是原生 HTML 元素的形式，以 is 特性扩展

简单来说就是一个vue文件就是一个组件，具体操作

- 新建vue文件
- 在App.vue中导入组件
- 在App.vue中注入组件
- 在App.vue中使用组件
  - style里面的scoped（添加这个之后只会让当前样式在当前页面生效）

```html
<template>
  <HelloWorld/>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'

export default {
  components: {
    HelloWorld
  }
}
</script>
<style scoped>
</style>
```

## 1.组件嵌套关系