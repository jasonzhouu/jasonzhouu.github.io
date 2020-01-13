---
layout: post
title:  "Vue 组件的多种表达方式"
date:   2020-01-13
tags:   [前端]
---



## 1. 通过 `Vue.component` 全局注册

```javascript
Vue.component('my-component-name', {
  // ... options ...
})
```

在注册之后可以用在任何新创建的 Vue 根实例 (`new Vue`) 的模板中。

缺点：如果使用一个像 webpack 这样的构建系统，全局注册所有的组件意味着即便你已经不再使用一个组件了，它仍然会被包含在你最终的构建结果中。这造成了用户下载的 JavaScript 的无谓的增加。

<p class="codepen" data-height="265" data-theme-id="default" data-default-tab="js,result" data-user="jasonzhouu" data-slug-hash="XWJqgrX" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="vue-global-component">
  <span>See the Pen <a href="https://codepen.io/jasonzhouu/pen/XWJqgrX">
  vue-global-component</a> by Yusheng Zhou (周宇盛) (<a href="https://codepen.io/jasonzhouu">@jasonzhouu</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>
<p class="codepen" data-height="265" data-theme-id="default" data-default-tab="js,result" data-user="codepen" data-slug-hash="qvGqop" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Vue Child Components &amp;amp; Props">
  <span>See the Pen <a href="https://codepen.io/team/codepen/pen/qvGqop">
  Vue Child Components &amp; Props</a> by CodePen (<a href="https://codepen.io/codepen">@codepen</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>
参考：

- [https://cn.vuejs.org/v2/guide/components.html#%E5%9F%BA%E6%9C%AC%E7%A4%BA%E4%BE%8B](https://cn.vuejs.org/v2/guide/components.html#基本示例)
- [https://cn.vuejs.org/v2/guide/components-registration.html#%E5%85%A8%E5%B1%80%E6%B3%A8%E5%86%8C](https://cn.vuejs.org/v2/guide/components-registration.html#全局注册)



## 2. 局部注册

```javascript
var ComponentA = { /* ... */ }
var ComponentB = { /* ... */ }
var ComponentC = { /* ... */ }

new Vue({
  el: '#app',
  components: {
    'component-a': ComponentA,
    'component-b': ComponentB
  }
})
```

**局部注册的组件在其子组件中\*不可用\***。例如，如果你希望 `ComponentA` 在 `ComponentB` 中可用，则你需要这样写：

```javascript
var ComponentA = { /* ... */ }

var ComponentB = {
  components: {
    'component-a': ComponentA
  },
  // ...
}
```

或者如果你通过 Babel 和 webpack 使用 ES2015 模块，那么代码看起来更像：

```javascript
import ComponentA from './ComponentA.vue'

export default {
  components: {
    ComponentA
  },
  // ...
}
```

参考：

[https://cn.vuejs.org/v2/guide/components-registration.html#%E5%B1%80%E9%83%A8%E6%B3%A8%E5%86%8C](https://cn.vuejs.org/v2/guide/components-registration.html#局部注册)

## 3. 单文件组件

前面2种原生的Javascript的写法，将html用字符串的形式书写，没有高亮。

**单文件组件**将vue组建写在单独的文件中，包含js, html模板，css。然后用webpack/browserify构建工具将其转成浏览器能识别的原生 Javascript。

![单文件组件的示例 ](../assets/2020-01-13-vue-component/vue-component.png)

由于有构建工具的帮助，语言也更加灵活：

![带预处理器的单文件组件的示例 (../assets/2020-01-13-vue-component/vue-component-with-preprocessors-1578895294291.png)](https://cn.vuejs.org/images/vue-component-with-preprocessors.png)

<iframe
     src="https://codesandbox.io/embed/o29j95wx9?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="Simple Todo App with Vue"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

参考：

https://cn.vuejs.org/v2/guide/single-file-components.html