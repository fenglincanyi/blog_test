---
title: Vue 从入门到搬砖
date: 2017-07-19 13:10
categories: 前端
tags: [vue]
---

## 前言
前段时间，捣鼓weex，就想到对vue有所了解。毕竟weex是从vue的基础上扩展而来，诸多特性也是vue的。作为目前前端比较火的框架之一，有所了解，也是有一定好处的，所谓"多不压身"。
Vue涵盖的不仅是一种前端框架，而且是前端生态的一系列设计思想、工具、规范。
本文直奔主题，什么装环境、hello world 之类的琐事就略过...... 

## 几个重要概念
### 构造器
每个 Vue.js 应用都是通过构造函数 Vue 创建一个 Vue 的根实例 启动的：
``` javascript
var vm = new Vue({
  // 属性
  // 方法
  // 各种选项对象...
})
```
在 vue 实例的基础上，提供了一些扩展的接口：

``` javascript
Vue.extend({
})
```
定义自己的组件，复用到不同的页面中，自底向上地构建页面应用。
### Virtual DOM
DOM：文档对象模型
Virtual DOM 是个什么鬼？
初学前端时，我们会直接用 js 去获取 dom 节点，拿到节点的属性和方法进行操作。但是这种操作性能比较差。
虚拟DOM：就是在 JS 和 DOM 之间做了一个缓存。可以类比 CPU 和硬盘，既然硬盘这么慢，我们就在它们之间加个缓存：既然 DOM 这么慢，我们就在它们 JS 和 DOM 之间加个缓存。CPU（JS）只操作内存（Virtual DOM），最后的时候再把变更写入硬盘（DOM）。

**大致过程**：
1. 用 JavaScript 对象结构表示 DOM 树的结构；然后用这个树构建一个真正的 DOM 树，插到文档当中
2. 当状态变更的时候，重新构造一棵新的对象树。然后用新的树和旧的树进行比较，记录两棵树差异
3. 把2所记录的差异应用到步骤1所构建的真正的DOM树上，视图就更新了

### 数据绑定
Vue 几乎屏蔽了直接通过操作 dom 更新的方式，更是趋向于 “数据的更新” 来达到 “视图的更新”。
Vue 实例的 data 的相关字段更新后，与之使用到的视图中也随之更新，重新渲染。
Vue 在不同组件间强制使用单向数据流。父组件通过 prop 向子组件赋值数据，子组件反过来向父组件 emit 发送事件

## 生命周期钩子

``` javascript
var LIFECYCLE_HOOKS = [
  'beforeCreate',//el 和 data 并未初始化 
  'created',// data 初始化
  'beforeMount',// el 初始化 
  'mounted',// 完成挂载
  'beforeUpdate',// 数据完成更新
  'updated',// virtual dom 更新并重新渲染
  'beforeDestroy',// vue 实例调用 desotry
  'destroyed',// 移除组件、事件、监听
  'activated',// 完成激活子组件
  'deactivated'// 停用释放子组件
];
```
<div style="text-align:center">
<img src="https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/vue_lifecycle.png" width="400" height="720"/>
</div>

## 语法概览
### 文本
数据绑定，使用 mustache 语法：

``` xml
<span>Message: {{ msg }}</span>
```
js 表达式：

``` xml
{{ number + 1 }}
{{ ok ? 'YES' : 'NO' }}
```
### 指令

``` xml
<p v-if="seen">现在你看到我了</p>
<a v-on:click="doSomething">
<a v-bind:href="url"></a>
```
### 过滤器
``` xml
{{ message | capitalize }}
```
``` javascript
filters: {
    capitalize: function (value) {
      if (!value) return ''
      value = value.toString()
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
  }
```
### 计算属性

``` xml
<div id="example">
  {{ message.split('').reverse().join('') }}
</div>
```
``` xml
<div id="example">
  {{ reversedMessage }}
</div>
```
```javascript
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    reversedMessage: function () {
      return this.message.split('').reverse().join('')
    }
  }
})
```
计算属性是基于它们的依赖进行缓存的。计算属性只有在它的相关依赖发生改变时才会重新求值。这就意味着只要 message 还没有发生改变，多次访问 reversedMessage 计算属性会立即返回之前的计算结果，而不必再次执行函数。
### 样式绑定
``` xml
<div v-bind:class="{ active: isActive }"></div>
<div v-bind:style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>
```
多个样式之间的切换，可以在 data 中定义：

``` javascript
data: {
  activeClass: 'active',
  errorClass: 'text-danger'
}
```
### 条件渲染
``` xml
<v-if>
<v-else>
<v-else-if>
<v-show>
```
v-if 是“真正的”条件渲染，因为它会确保在切换过程中条件块内的事件监听器和子组件适当地被销毁和重建。
v-if 也是惰性的：如果在初始渲染时条件为假，则什么也不做——直到条件第一次变为真时，才会开始渲染条件块。
v-show 不管初始条件是什么，元素总是会被渲染，并且只是简单地基于 CSS 进行切换。
### 列表渲染
``` xml
<v-for>
```
* key使用
当 Vue.js 用 v-for 正在更新已渲染过的元素列表时，它默认用 “就地复用” 策略
如果数据项的顺序被改变，Vue将不是移动 DOM 元素来匹配数据项的顺序， 而是简单复用此处每个元素，并且确保它在特定索引下显示已被渲染过的每个元素
建议尽可能使用 v-for 来提供 key ，除非迭代 DOM 内容足够简单，或者你是故意要依赖于默认行为来获得性能提升。
* 数组更新检测
由于 JavaScript 的限制， Vue 不能检测以下变动的数组：
1. 当你利用索引直接设置一个项时，例如： vm.items[indexOfItem] = newValue
2. 当你修改数组的长度时，例如： vm.items.length = newLength
解决：
Vue.set(example1.items, indexOfItem, newValue) 或 example1.items.splice(indexOfItem, 1, newValue)
example1.items.splice(newLength)
### 组件

``` javascript
// 注册
Vue.component('my-component', {
  template: '<div>A custom component!</div>'
})
// 创建根实例
new Vue({
  el: '#example'
})
```

``` xml
<!--使用-->
<div id="example">
  <my-component></my-component>
</div>
```
组件通信

<div style="text-align:center">
<img src="https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/props-events.png" width="210" height="200"/>
</div>

**单项数据流**
prop 是单向绑定的：当父组件的属性变化时，将传导给子组件，但是不会反过来


## 过渡效果
配合 css 实现动画效果，元素的出现和消息过渡动画也是非常有意思的

    v-enter
    v-enter-active
    v-enter-to
    v-leave
    v-leave-active
    v-leave-to

``` xml
用 包裹 view 动画元素
<transition name="xxx"></transition>
```

## 路由：vue-router
https://router.vuejs.org/zh-cn/

```xml
<div id="app">
  <!--将来被替换的区域-->
  <router-view></router-view>
</div>

<!--点击跳转配置，可传参数-->
<router-link :to="{ name: 'user', params: { userId: 123 }}">User</router-link>
```
## 其他推荐
饿了么组件库：
http://element.eleme.io/#/zh-CN
vue 网络库：
axios：https://github.com/mzabriskie/axios

<hr/>

附录
Demo:
https://github.com/fenglincanyi/VueDemo1
https://github.com/fenglincanyi/VueDemo2

参考：
https://www.zhihu.com/question/29504639
https://segmentfault.com/a/1190000008010666