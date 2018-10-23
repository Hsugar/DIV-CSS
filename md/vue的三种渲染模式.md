vue的三种渲染模式：

1.自定义Render函数

```javascript
1.Vue.component('anchored-heading', {
2.    render: function (createElement) {
3.        return createElement(
4.            'h' + this.level,   
    // tag name 标签名称
5.            this.$slots.default // 子组件中的阵列
6.        )
7.    },
8.    props: {
9.        level: {
10.            type: Number,
11.            required: true
12.        }
13.    }
14.})
```

2. template写法

   ```javascript
   1.var vm = new Vue({
   2.    data: {
   3.        // 以一个空值声明 `msg`
   4.        msg: ''
   5.    },
   6.    template: '<div>{{msg}}</div>'
   7.})
   ```

3. el写法（这个就是入门时最基本的写法）

```javascript
1.var app = new Vue({
2.    el: '#app',
3.    data: {
4.        message: 'Hello Vue!'
5.    }
6.})
```

这三种模式最终都是得到render函数。只不过用户自定义的render函数省去了程序分析的过程，等同于处理过的render函数。而普通的template或者el只是字符串，需要解析成AST,再将AST转化为render函数。

接下来说下什么是AST

它是指抽象语法树（abstract syntax tree），或者语法树（syntax tree），是源代码抽象语法结构的树状表现形式。vue在mount过程中，template会被编译成AST语法树。

然后，经过generate（将AST语法树转化成render function 字符串的过程）得到render函数，返回VNode。VNode是vue的虚拟DOM节点，里面包含签名，子节点，文本等信息



