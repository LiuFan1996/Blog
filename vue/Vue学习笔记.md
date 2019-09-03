# Vue学习笔记

## Vue安装

npm install vue -S

## Vue的基本使用

首先在HTML中使用vue时需要引入vue的包

```js
	<script type="text/javascript" src="../node_modules/vue/dist/vue.js"></script>
```

然后通过new Vue的方式创建一个Vue实例

```js
let vue = new Vue({
    el:"#app", //目的地
    data:{
        //数据属性
        //既可以是是一个对象也可以是一个函数
        msg:"哈哈哈哈",
        str:"啥都没有",
        isTrue:1==1,
    },
    //如果template中定义了内容，那么优先加载template,如果没有定义内容那么加载的是模板例如这个文件中的"#app"
    template:"<div></div>",
});
        //数据发生改变时，视图也会发生改变，简言之，数据驱动视图
        //除了属性，Vue还暴露了一些游泳的实例属性和方法，他们都有前缀$
```

Vue在一定程度的理解上可以理解为数据驱动视图，通过数据动态的渲染视图，同时将数据对象与对应的模板绑定

## Vue之MVVM

MVVM是Model-View-ViewModel的简写。它本质上就是MVC 的改进版。MVVM 就是将其中的View 的状态和行为抽象化，让我们将视图 UI 和业务逻辑分开。当然这些事 ViewModel 已经帮我们做了，它可以取出 Model 的数据同时帮忙处理 View 中由于需要展示内容而涉及的业务逻辑

## Vue基础

### Vue插值

最常用的为{{}}插值，通常作用在元素标签中列入

```vue
<h1>{{msg}}</h1>
```

双大括号会将数据解释为普通文本，而非 HTML 代码

同时{{}}不能作用在html的特性中

同时支持javaScript表达式
```vue
<div>{{ number + 1 }}
{{ ok ? 'YES' : 'NO' }}
{{ message.split('').reverse().join('') }}</div>

<div v-bind:id="'list-' + id"></div>
```

### Vue指令

#### v-text：innerText 作用与双大括号插值{{}}类似，都是向标签内插值

```vue
<a v-text="msg">...</a>
```
#### v-html：innerHtml 向标签内插入html内容

```vue
<a v-html="html">...</a>
```
#### v-if：数据属性对应的值如果为假则不再页面中渲染反之亦然 相当与appendChild()与removeChild()

```vue
<a v-if="isShow">...</a>
```
#### v-show：控制dom元素的显示与隐藏相当与display：none | block

```vue
<a v-show="isShow">...</a>
```
v-if与v-show的区别相当去v-if时对应dom的操作，v-show是对元素的css操作，推荐操作频繁的组建使用v-show，不长使用的使用v-if

#### v-bind绑定标签上的和 



#### 属性（内置的属性和自定义的属性）简写直接使用冒号

```vue
<a v-bind:herf="herf">...</a>
```
#### v-on:原生事件名=方法名

```vue
<a v-on:click="doSomething">...</a>

<!-- 缩写 -->
<a @click="doSomething">...</a>
```
#### v-for = ”(item,index) in list“ 指令优先级最大

```html
<ul>
	<li v-for = (item,index) inr list>
	{{item}}
	</li>
</ul>
```

#### v-model ：双向数据绑定
```vue
    <input type="text" v-model = "name">
    <input type="te xt" v-bind:value = "name" v-on:input = "valueChange">
    <p>{{name}}</p>
    new Vue({
        el:"#app",
        data:function () {
            return {
                name:"LiuFan"
            }
        },
        template:'',
        methods:{
            valueChange(e){
               this.name = e.target.value;
            }
        },
    });
```

### Vue计算属性

对于计算属性的使用：我用过的场景有:

​	        在写钱包项目时对以太坊和代币进行过滤和格式转换

​		比如我在后台前请求的数据返回的是{balance:1000000000000000}

​		类似与这样大的一个数据，如果直接使用肯定不行的，我们必须对它进行处理，

​		此时我使用计算属性，通过以下方式对balance进行处理

```js
balanceFilter: function (balance) {
  if (balance >= 10 ** 18) {
    return Math.floor((balance / 10 ** 18) * 1000) / 1000 + ' ETH'
  } else if (balance >= 10 ** 15) {
    return Math.floor((balance / 10 ** 15) * 1000) / 1000 + ' MILLIETH'
  } else if (balance >= 10 ** 12) {
    return Math.floor((balance / 10 ** 12) * 1000) / 1000 + ' MICROETH'
  } else if (balance >= 10 ** 9) {
    return Math.floor((balance / 10 ** 9) * 1000) / 1000 + ' GWEI'
  } else if (balance >= 10 ** 6) {
    return Math.floor((balance / 10 ** 6) * 1000) / 1000 + ' MWEI'
  } else if (balance >= 10 ** 3) {
    return Math.floor((balance / 10 ** 3) * 1000) / 1000 + ' KWEI'
  }
  return balance + ' WEI'
},
```

下面是一个加法器的例子：

```vue
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <div id="app">
        <input type="number" v-model:value="value1">+<input type="number" v-model:value="value2">=<span >{{add}}</span>
    </div>
    <script src="../node_modules/mathjs/dist/math.js"></script>
    <script type="text/javascript" src="../node_modules/vue/dist/vue.js"></script>
    <script type="text/javascript">
        new Vue({
            el : "#app",
            data :function () {
                return{
                    value1:0,
                    value2:0, 
                    value3:0,
                }
            },
            // 计算属性是用computed ，在使用时可以直接通过对应的方法名调用
            // 对与计算属性和监听来说，需要在合适的时候去使用
            // 最直观的区别是，监听不能改变自身的值，不然会陷入死循环，而计算属性可以
            // 监听也不能监听到对象，只能监听对象的某个属性，例如‘obj.msg’这样的格式
            computed:{
                add :function(){
                    console.log(this.value1)
                    console.log(this.value2)
                    return math.add(this.value1,this.value2)
                }
            }
        });
    </script>
</body>
</html>
```

计算属性默认只有 getter ，不过在需要时你也可以提供一个 setter 

### Vue监听

Vue 通过 `watch` 选项提供了一个更通用的方法，来响应数据的变化。当需要在数据变化时执行异步或开销较大的操作时，这个方式是最有用的。

```vue
*/DOCTYPE html>
<html>
    <head>
        <title>

        </title>
    </head>
    <body>
        <div id="app">
            <input type="text" v-model = "msg">
            <h3>{{msg}}</h3>
        </div>
        <script type="text/javascript" src="../node_modules/vue/dist/vue.js"></script>
        <script type="text/javascript">
        //不能直接监听数组或对象，但是可以监听数组或对象的最小单元（需要深度监视）
        //总结watch只能监听基本数据类型
            new Vue({
                el:"#app",m  
                data : function(){
                    return {
                        msg:"",
                    }01
                },	qqv514dxbcnvmbn
               watch:{
                   msg:function(newVal,oldVal){
                       console.log(newVal,oldVal)
                   }
               }
            });
        </script>
    </body>
</html> k   
```

## Vue组件

