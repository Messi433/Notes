# Vue.js 进阶1

## 一、前端模块化

webpack

Vue CLI

前端复杂化：命名冲突、代码不能复用

前端模块化：https://www.bilibili.com/video/BV15741177Eh?p=73

### 1.1 前端模块化的引出

#### JS原始功能

- 在网页开发的早期, js制作作为一种脚本语言，做一些简单的表单验证或动画
  实现等,那个时候代码还是很少的
  - 那个时候的代码是怎么写的呢?直接将代码写在<script>标签中即可
- 随着ajax异步请求的出现,慢慢形成了前后端的分离
  - 客户端需要完成的事情越来越多,代码量也是与日俱增
  - 为了应对代码量的剧增,我们通常会将代码组织在多个js文件中,进行维护
  - 但是这种维护方式,依然不能避免一些灾难性的问题
- 比如全局变量同名问题：看右边的例子
- 另外,这种代码的编写方式对js文件的依赖顺序几乎是强制性的
  - 但是当js文件过多,比如有几十个的时候,弄清楚它们的顺序是一件比较头痛的事情
  - 而且即使你弄清楚顺序了,也不能避免上面出现的这种尴尬问题的发生

#### 模块化雏形

- **匿名函数(闭包) =>解决命名冲突问题**
  - aaa.js文件中，我们使用匿名函数

```js
(function () {
    //导出的对象
    var obj = {};
    //对象
    var name = 'xiaoming';
    var age = 18;

    //自定义函数
    function sum(num1, num2) {
        return num1 + num2;
    }

    var flag = true;
    if (flag) {
        console.log(sum(1, 2))
    }
    //空的obj对象动态添加flag、sum属性
    obj.flag = flag;
    obj.sum = sum;

    return obj
})()
```

```js
/*只解决了闭包，没解决代码复用 => 无法引用aaa.js*/
;(function () {
    //1.想使用aaa.js中的flag
    if (flag){
        console.log("小明哈哈哈")
    }
    //2.想使用aaa.js中的sum()
    console.log(sum(3,4));
})()
```

- 但是如果我们希望在main.js文件中,用到flag ,应该如何处理呢?
  - 显然,另外一个文件中不容易使用,因为flag是一个局部变量。
  - 解决方案：使用模块作为出口

<!--aaa.js-->

```js
var moduleA = (function () {
    //导出的对象
    var obj = {};
    //对象
    var name = 'xiaoming';
    var age = 18;

    //自定义函数
    function sum(num1, num2) {
        return num1 + num2;
    }

    var flag = true;
    if (flag) {
        console.log(sum(10, 20))
    }
    //空的obj对象动态添加flag、sum属性
    obj.flag = flag;
    obj.sum = sum;

    return obj //导出
})()
```

<!--main.js-->

```js
(function () {
    //1.想使用aaa.js中的flag
    if (moduleA.flag){
        console.log("小明哈哈哈")
    }
    //2.想使用aaa.js中的sum()
    console.log(moduleA.sum(2,1));

    //3.想使用bbb.js中的flag
    if (moduleB.flag){
        console.log("小红哈哈哈")
    }
    //4.想使用bbb.js中的sum()
    console.log(moduleB.sum(3,4));
})()
```

- 我们做了什么事情呢?
  - 非常简单,在匿名函数内部,定义一个对象。
  - 给对象添加各种需要暴露到外面的属性和方法(不需要暴露的直接定义即可)。
  - 最后将这个对象返回,并且在外面使用了一个MoudleA接受。
- 接下来,我们在man.js中怎么使用呢?
  - 我们只需要使用属于自己模块的属性和方法即可
- 这就是模块最基础的封装,事实上模块的封装还有很多高级的话题:
  - 但是我们这里就是要认识一下为什么需要模块 ,以及模块的原始雏形。
  - 幸运的是,前端模块化开发已经有了很多既有的规范,以及对应的实现方案。

- 常见的模块化规范:
  - CommonJS、AMD、CMD ,也有ES6的Modules



#### CommonJS

模块化的核心：导入、导出

##### CommonsJS 导入导出(了解)

Commons导出

```js
/...代码/
//commonsJS导出
moduleA.exports = {
    flagA: flag,
    sumA: sum
}
```

Commons导入

```js
//commonsJS导入
//写法1
var aaa = require('./aaa.js')
var flag = aaa.flagA
var sum = aaa.sumA
//写法2
var {flag, sum} = require('./aaa.js')
```



### 1.2 ES6导入导出

- 我们使用export指令导出了模块对外提供的接口,下面我们就可以通过import命令来加载对应的这个模块了
- 我们需要在HTML代码中引入两个js文件,并且类型需要设置为module
- `type="module"`，意味着每个.js都是独立的模块，不可以直接引用，需要导入导出

```html
<script src="js/aaa.js" type="module"></script>
<script src="js/bbb.js" type="module"></script>
<script src="js/main.js" type="module"></script>
```

- aaa.js用来对外export

```js
//导出的对象
var obj = {};
//对象
var name = 'xiaoming';
var age = 18;
var flag = true;

//自定义函数
function sum(num1, num2) {
    return num1 + num2;
}

if (flag) {
    console.log(sum(10, 20))
}

//ES6导出方式1
export {
    flag, age, name,sum
}
//ES6导出方式2
export var money = 1000.0
export var height = 185
```

- import指令用于导入模块中的内容,比如main.js的代码

```js
/* 导入 */
import f, {name,age,flag,sum} from "./aaa";

sum(20,30);
console.log(name)
console.log(age)
console.log(flag)

/* 导入 */
import {height,money} from "./aaa";

console.log(height);
console.log(money);
```

- export default

某些情况下, 一个模块中包含某个的功能,我们并不希望给这个功能命名,而且让导入者可以自己来命名
这个时候就可以使用`export default`

```js
//ES6导出方式3:export default
const address = 'beijing';
export default address //export default导出的属性唯一

/*function f() {
    console.log("hello world");
}
export default f*/ //export default导出的函数唯一
```

> 另外，需要注意:
> export default在同一个模块中,不允许同时存在多个。



- 如果我们希望某个模块中所有的信息都导入，一个个导入显然有些麻烦: 
  - 通过可以导入模块中所有的export变量
  - 但是通常情况下我们需要给起一个别名,方便后续的使用

```js
import {所有属性、方法、类} from './aaa.js'

import * as aaa from './aaa'
aaa.money
aaa.height
aaa.mul()
aaa.Person
```



### 1.3 WebPack

#### 学习目标

- [ ] **认识webpack**
- [ ] **webpack的安装**
- [ ] **webpack的起步**
- [ ] **webpack的配置**
- [ ] **loader的使用**
- [ ] **webpack中配置Vue**
- [ ] **plugin的使用**
- [ ] **搭建本地服务器**

