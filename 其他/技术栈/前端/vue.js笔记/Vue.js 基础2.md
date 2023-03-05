# Vue.js 基础2

## 四、Vue组件化开发

### 学习目标：

- 能够知道组件化开发思想
- 能够知道组件的注册方式
- 能够说出组件间的数据交互方式
- 能够说出组件插槽的用法
- 能够说出Vue调试工具的用法
- 能够基于组件的方式实现业务功能

### 4.1 组件化开发思想

#### 4.1.1 组件化基本思想

- 标准：所有代码风格的开发者遵循一套规范
- 分治：独立团队，各自开发不同模块
- 重用：代码的可复用性
- 组合：

### 4.2 组件注册

#### 组件注意事项

- 组件参数的data值必须是函数同时这个函数要求返回一个对象 

- 组件模板必须是单个根元素

  - `template: '<button @click="handle">{{count}}次<button><span></span>',<!--不可以-->`

- 组件模板的内容可以是模板字符串

  ```js
  template:`
  <div>
  	<table>
  		<tr>
  			<td></td>
  		</tr>
  	</table>
  </div>
  `
  ```

- 组件命名方式：

  - 短横线方式：`Vue.component ('my-component', { /* ... */ })`
  - 驼峰方式：`Vue.component ( ' MyComponent' , { /* ... */ })`

  ```html
  1.如果使用驼峰式命名组件，那么在使用组件的时候，只能在字符串模板中用驼峰的方式使用组件，
  2.但是在普通的标签模板中，必须使用短横线的方式使用组件
  3.引用组件必须包含根元素标签且为vue实例<div id="app">...</div>
  4.vue实例在组件构造器、组件注册之下
  <body>
  <div id="app">
      <my-cpn></my-cpn>
  </div>
  <script src="../js/vue.js"></script>
  <script>
      const app = new Vue({
          el:'#app',
          data:{
              msg:'hello world'
          }
      });
      /*语法1：创建组件构造器对象*/
      const cpn = Vue.extend({
          template: `
              <div>
                  <h2>我是标题</h2>
                  <p>我是内容，哈哈哈哈</p>
                  <p>我是内容，呵呵呵呵</p>
              </div>`
  
      });
    	/*语法2：以声明对象的方式*/
      var cpn = {
            template: `
                <div>
                    <h2>我是标题</h2>
                    <p>我是内容，哈哈哈哈</p>
                    <p>我是内容，呵呵呵呵</p>
                </div>`
       };
      /*注册组件*/
      Vue.component('my-cpn', cpn);
  </script>
  </body>
  ```

#### 1.全局注册

- Vue.component('组件名称', { })     第1个参数是标签名称，第2个参数是一个选项对象
- **全局组件**注册后，任何**vue实例**都可以用

```html
<div id="app">
  <button-counter></button-counter>
  <!-- 组件可以重复调用，每次调用的组件就是新的组件实例 -->
  <button-counter></button-counter>
</div>
<script type="text/javascript">
  Vue.component('button-counter', {
    data: function () {
      return {
        count: 0
      }
    },
    template: '<button @click="handle">点击了{{count}}次<button>',
    methods: {
      handle: function () {
        this.count += 2;
      }
    }
  })
  var vm = new Vue({
    el: '#app',
    data: {},
  });
</script>
```

#### 2.局部组件注册

```html
<div id="app">
  <!-- 容器内调用局部组件 -->
  <hello-world></hello-world>
  <hello-tom></hello-tom>
  <!-- 全局组件嵌套局部组件:不可以 -->
  <!-- <test><hello-world></hello-world></test> -->
  <!-- 局部组件嵌套全局组件:不可以 -->
  <!-- <hello-world><test></test></hello-world> -->
</div>
<script type="text/javascript">
  //定义并注册全局组件test
  Vue.component('Test', {
    data: function () {
      return {
        msg: 'test'
      }
    },
    template: '<div>{{msg}}</div>'
  });
  //定义局部组件
  var HelloWorld = {
    data:function(){
      return {
        msg:"hello world"
      }
    },
    template:'<div>{{msg}}</div>'
  }
  var HelloTom = {
    data:function(){
      return {
        msg:"hello tom"
      }
    },
    template:'<div>{{msg}}</div>'
  }
  var vm = new Vue({
    el: '#app',
    data: {},
    //注册局部组件
    components:{
      'hello-world':HelloWorld,
      'hello-tom':HelloTom,
    }
  });
</script>
```

### 4.3 Vue调试工具



### 4.4 Vue组件之间传值

#### 4.4.1 父组件向子组件传值

- 父组件发送的形式是以属性的形式绑定值到子组件身上。
- 然后子组件用属性props接收
- 在props中使用驼峰形式，模板中需要使用短横线的形式
- 字符串形式的模板中没有这个限制

```html
<div id="app">
  <!-- 父组件数据 -->
  <div>{{pmsg}}</div>
  <!-- 来自接受来自父组件的内容，属性写死-->
  <menu-item title='来自父组件的内容'></menu-item>
  <!-- 来自接受来自父组件的内容，属性绑定-->
  <menu-item :title=ptitle></menu-item>
  <!-- 来自接受来自父组件的内容，属性绑定结合写死属性-->
  <menu-item :title=ptitle content='内容'></menu-item>
</div>
<script type="text/javascript">
  Vue.component('menu-item', {
    props:['title','content'],
    data: function () {
      return {
        msg: '子组件本身的数据'
      }
    },
    template: '<div>{{msg +" --- "+ title + " --- " + content}}</div>'
  });
  var vm = new Vue({
    el: '#app',
    data: {
      pmsg:'父组件本身的数据',
      ptitle:'父组件的内容By属性绑定方式'
    },
  });
</script>
```

- props属性值类型(Course259)
  - String
  - Number
  - Boolean
  - Array
  - Object

#### 4.4.2 子组件向父组件传值

- 子组件通过自定义事件向父组件传递信息，用`$emit()`触发事件
  - `<button v-on:click='$emit(enlarge-text)'>扩大字体</button>`
- `$emit()`  第1个参数为 `自定义的事件名称`  第2个参数为`需要传递的数据`
- 父组件用v-on 监听子组件的事件->触发效果
  - `<menu-item v-on:enlarge-text='fontSize+=0.1'></menu-item>`

```html
<div id="app">
  <!-- 父组件数据 -->
  <div :style='{fontSize: fontSize + "px"}'> {{pmsg}}</div>
  <!-- 父组件监听子组件 -->
  <menu-item @enlarge-text='handle($event)'></menu-item>
</div>
<!-- props传递数据原则：单项数据流 -->
<script type="text/javascript">
  Vue.component('menu-item', {
    props:['title','content'],
    data: function () {
      return {
        msg: '子组件本身的数据',
      }
    },
    template: `
<!--<button @click='$emit("enlarge-text")'>增大字体</button>,不传值--> 
<div>
<!--传值用法-->
<button @click='$emit("enlarge-text",5)'>每次增加5</button>
<button @click='$emit("enlarge-text",10)'>每次增加10</button>
  </div>
`
  });
  var vm = new Vue({
    el: '#app',
    data: {
      pmsg:'父组件本身的数据',
      fontSize:10
    },
    methods:{
      handle:function(val){
        //扩大字体
        // this.fontSize += 5; 写死
        this.fontSize +=val;
      }
    }

  });
</script>
```



#### 4.4.3 兄弟之间的传递

- 兄弟之间传递数据需要借助于事件中心，通过事件中心传递数据   
  - 提供事件中心    var hub = new Vue()
- 传递数据方，通过一个事件触发hub.$emit(方法名，传递的数据)
- 接收数据方，通过mounted(){} 钩子中  触发hub.$on()方法名
- 销毁事件 通过hub.$off()方法名销毁之后无法进行传递数据

```html
<body>
  <div id="app">
    <!-- 父组件数据 -->
    <div>{{pmsg}}</div>
    <div><button @click='handle'>销毁事件</button></div>
    <test-tom></test-tom>
    <test-jerry></test-jerry>
  </div>
  <!-- props传递数据原则：单项数据流 -->
  <script type="text/javascript">
    //定义中心组件
    var hub = new Vue();
    //兄弟组件1
    Vue.component('test-tom', {
      props: ['title', 'content'],
      data: function () {
        return {
          num: 0,
        }
      },
      template: ` 
<div>
<div>Tom:{{num}}</div>
<div><button @click='handle'>增加2</button></div>
    </div>
`,
      methods: {
        handle: function () {
          //2、传递数据方，通过一个事件触发hub.$emit(方法名，传递的数据),触发兄弟组件的事件
          // 将数据、事件传给兄弟组件
          hub.$emit('jerry-event', 2);
        }
      },
      mounted: function () {
        // 3、接收数据方，通过mounted(){} 钩子中  触发hub.$on()方法名，val是兄弟组件传来的
        hub.$on('tom-event', (val) => {
          this.num += val;
        });
      }
    });
    //兄弟组件2
    Vue.component('test-jerry', {
      props: ['title', 'content'],
      data: function () {
        return {
          num: 0,
        }
      },
      template: ` 
<div>
<div>jerry:{{num}}</div>
<div><button @click='handle'>增加1</button></div>
    </div>
`,
      methods: {
        handle: function () {
          //2、传递数据方，通过一个事件触发hub.$emit(方法名，传递的数据),触发兄弟组件的事件
          // 将数据、事件传给兄弟组件
          hub.$emit('tom-event', 1);
        }
      },
      mounted: function () {
        // 3、接收数据方，通过mounted(){} 钩子中  触发hub.$on()方法名，val是兄弟组件传来的
        hub.$on('jerry-event', (val) => {
          this.num += val;
        });
      }
    });
    //全局实例
    var vm = new Vue({
      el: '#app',
      data: {
        pmsg: '父组件本身的数据',
        fontSize: 10
      },
      methods: {
        handle: function () {
          //4、销毁事件 通过hub.$off()方法名销毁之后无法进行传递数据  
          hub.$off('tom-event');
          hub.$off('jerry-event');
        }
      }

    });
  </script>
</body>
```



### 4.5 组件插槽

组件的最大特性就是**复用性**，而用好插槽能大大提高组件的可复用能力

- slot翻译为插槽:
  - 在生活中很多地方都有插槽,电脑的USB插槽,插板当中的电源插槽。
  - 插槽的目的是让我们原来的设备具备更多的扩展性。
  - 比如电脑的USB我们可以插入U盘、硬盘、手机、音响、键盘、鼠标等等。
- 组件的插槽:
  - 组件的插槽也是为了让我们封装的组件更加具有扩展性。
  - 让使用者可以决定组件内部的一些内容到底展示什么。
- 栗子:移动网站中的导航栏。
  - 移动开发中,几乎每个页面都有导航栏。
  - 导航栏我们必然会封装成一个插件,比如nav-bar组件。
  - 一旦有了这个组件,我们就可以在多个页面中复用了。
- 但是,每个页面的导航是一样的吗? No ,我以京东M站为例。

#### 4.5.1 匿名插槽

```html
<!--组件插槽 基本用法-->
<div id="app">
    <!-- 子组件 -->
    <!-- 提供数据，插槽中的默认内容被覆盖 -->
    <error-box>data1</error-box>
    <error-box>data2</error-box>
    <!-- 不提供内容，插槽中显示默认数据 -->
    <error-box></error-box>
</div>
<!-- props传递数据原则：单项数据流 -->
<script type="text/javascript">
    //兄弟组件2
    Vue.component('error-box', {
        template: `
        <div>
            <strong>
                <slot>默认内容</slot>
            </strong>
        </div>
        `,
    });
    const app = new Vue({
        el: "#app",
        data: {
            msg: "msg"
        }
    });
</script>
```

> 核心标签：<slot></slot>
> 不添加<slot></slot>时，若在子组件标签体里添加内容，是不会有**替换**效果的

#### 4.5.2 具名插槽

- 具有名字的插槽 

- 使用 <slot> 中的 "name" 属性绑定元素

```java
<div id="app">
    <cpn><span slot="left">111</span></cpn>
    <cpn><span slot="center">222</span></cpn>
    <cpn><span slot="right">333</span></cpn>

</div>
<template id="cpn">
    <div>
        <slot name="left"><span>左边</span></slot>
        <slot name="center"><span>中间</span></slot>
        <slot name="right"><span>右边</span></slot>
    </div>
</template>
<script type="text/javascript">
    /*具名插槽*/
    var vm = new Vue({
        el: '#app',
        data: {},
        //局部组件集合
        components:{
            //组件名
            cpn:{
               template:'#cpn' //引用id为cpn的Template
            }
        }
    });
</script>
```

```html
<div id="app">
  <!--写法1-->
  <base-layout>
    <!-- 2、 通过slot属性来指定, 这个slot的值必须和下面slot组件得name值,对应上
如果没有匹配到 则放到匿名的插槽中   --> 
    <p slot='header'>标题信息</p>
    <p>主要内容1</p>
    <p>主要内容2</p>
    <p slot='footer'>底部信息信息</p>
  </base-layout>
	
  <!--写法2：适用于多个标签-->
  <base-layout>
    <!-- 注意点：template临时的包裹标签最终不会渲染到页面上     -->  
    <template slot='header'>
      <p>标题信息1</p>
      <p>标题信息2</p>
    </template>
    <p>主要内容1</p>
    <p>主要内容2</p>
    <template slot='footer'>
      <p>底部信息信息1</p>
      <p>底部信息信息2</p>
    </template>
  </base-layout>
</div>
<script type="text/javascript" src="js/vue.js"></script>
<script type="text/javascript">
  /*
      具名插槽
    */
  Vue.component('base-layout', {
    template: `
      <div>
        <header>
        ###	1、 使用 <slot> 中的 "name" 属性绑定元素 指定当前插槽的名字
        <slot name='header'></slot>
        </header>
      	<main>
     		 <slot></slot>
        </main>
      	<footer>
      		//注意点： 具名插槽的渲染顺序，完全取决于模板，而不是取决于父组件中元素的顺序
      		<slot name='footer'></slot>
       </footer>
  	</div>
	`
  });
  var vm = new Vue({
    el: '#app',
    data: {

    }
  });
</script>  
</div>
```



#### 4.5.3 作用域

##### 编译作用域

```html
<div id="app">
    <!--父组件的isShow为True，使用的是Vue实例的数据-->
    <!--注意父组件isShow = False则父组件和子组件都不显示-->
    <cpn v-show="isShow"></cpn>
</div>
<template id="cpn">
    <div>
        <h2>我是子组件</h2>
        <p>我是内容</p>
        <!--子组件的isShow为False，使用的是子组件的数据-->
        <span v-show="isShow">hhh</span>
    </div>
</template>
<script type="text/javascript">
    /*具名插槽*/
    var vm = new Vue({
        el: '#app',
        data: {
            message: 'hello world',
            isShow: true
        },
        //局部组件集合
        components: {
            //组件名
            cpn: {
                template: '#cpn', //引用id为cpn的Template,
                data() {
                    return {
                        isShow: false
                    }
                }
            }
        }
    });
</script>
```

##### 作用域插槽

- 父组件替换插槽的标签,但是内容由子组件来提供

- 父组件对子组件加工处理

- 既可以复用子组件的slot，又可以使slot内容不一致





```html
<style type="text/css">
  .current {
    color: orange;
  }
</style>
<body>
  <div id="app">
    <fruit-list :list='list'>
      <!-- 插槽填充：slotProps自定义 -->
      <template slot-scope='slotProps'>
        <!-- 逻辑控制实现填充 -->
        <strong v-if='slotProps.info.id==3' class="current">{{slotProps.info.name}}</strong>
        <span v-else>{{slotProps.info.name}}</span>
      </template>
    </fruit-list>
  </div>
  <script type="text/javascript" src="js/vue.js"></script>
  <script type="text/javascript">
    /*
      作用域插槽
    */
    Vue.component('fruit-list', {
      props: ['list'], //接收父组件的值
      template: `
<div>
<li :key='item.id' v-for='item in list'>
<!--这是插槽定义：其中绑定的自定义属性info，提供给父组件-->
<slot :info='item'>{{item.name}}</slot>
    </li>
    </div>
`
    });
    var vm = new Vue({
      el: '#app',
      data: {
        list: [{
          id: 1,
          name: 'apple'
        },{
          id: 2,
          name: 'orange'
        },{
          id: 3,
          name: 'banana'
        }]
      }
    });
  </script>
</body>
```


