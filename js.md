### js模块（）module

语言级的模块系统出现在ES6中，现已得到了所有主流浏览器和Node.js的支持，也就是说主流浏览器或Node.js是能够解析import、export这些语句的。

export标记了可以在该模块外部访问的变量、函数。

想在html文件中直接使用模块，需用：

```html
<script type="module">
	import ...
</script>
```

import/export只在http(s)协议能起作用，file://无效。

模块的核心功能：

1. 始终使用 'use strict'
2. 模块级作用域

**模块代码仅在第一次被导入时执行一次**

有一条很重要的规则：顶层模块代码应该用于初始化，创建模块特定的内部数据结构，如果需要多次调用某些东西，就应该以函数的形式导出。

理解上面的规则：

```javascript
// admin.js
export const config = {};

export function sayHi() {
  console.log(config.user);
}
```

```javascript
// init.js
import { config } from './admin.js';

config.user = 'Fan';
```

```javascript
// another.js
import { config, sayHi } from './admin';

// output: 'Fan'
sayHi();
```

这是经典的使用方式：

1. 模块导出一些配置、方法
2. 在第一次导入时，就对它初始化，写入属性，可以在应用的顶级脚本中（上面的init.js）进行此操作
3. 进一步，在其他模块中导入使用。

### 模块中的this

对比一下就知道了：

```html
<script>
  // output: window
	console.log(this);
</script>

<script type="module">
  // output: undefined
  console.log(this);
</script>
```

### 在浏览器中使用模块，可以看[这里](https://zh.javascript.info/modules-intro#liu-lan-qi-te-ding-gong-neng)

只需记住，模块脚本是延迟执行的，即模块脚本不会阻塞html的处理，它们会与其它资源并行加载，并且在html文档完全准备就绪后再执行，也就是说模块脚本能看到全部的html，包括在它下方的html。

具有type=“module”的外部脚本，即有src的脚本，只会运行一次

```html
<!-- 脚本 my.js 被加载完成（fetched）并只被运行一次 -->
<script type="module" src="my.js"></script>
<script type="module" src="my.js"></script>
```

### 不允许裸模块

在浏览器 中，import必须给出相对或绝对url。

```javascript
import {sayHi} from 'sayHi'; // Error，“裸”模块
// 模块必须有一个路径，例如 './sayHi.js' 或者其他任何路径
```

### 模块导出export

以下导出方式效果都一样

```javascript
// 方式1：在function前加export，依然是函数声明，而不会成为函数表达式
// 导出变量
export const name = 'fan';

// 导出类
export class Person {
  constructor(name) {
    this.name = name
  }
}

// 导出函数
export function sayName() {
  console.log(name);
}
```

```javascript
// 方式2：
// 导出变量
const name = 'fan';

// 导出类
class Person {
  constructor(name) {
    this.name = name
  }
}

// 导出函数
function sayName() {
  console.log(name);
}

export { name, Person, sayName };	// 导出变量列表
```

### 模块导入import

把需要导入的东西放在 import { } 中：

```javascript
import { name, Person, sayNam } from '...';
```

若导入的东西太多，可以：

```javascript
import * as Obj from '...';

console.log(Obj.name);
const person = new Obj.Person();
Obj.sayName();
```

### 推荐按需导入

一次性全部导入写起来爽，但现代构建工具会将模块打包到一起并对其进行优化，以加快加载速度并删除未使用的代码。

1. 按需导入后，模块中未被使用的功能就会从打好的包中被删掉，可减小构件的体积，这就是所谓的摇树（tree shaking）
2. 按需导入，使用的时候，更加简洁，sayHello() 而不是Obj.sayHello()
3. 便于重构

### 导入别名

```javascript
import { name as n, sayHello as s, Person as p  } from '...';
```

### 导出别名

```javascript
// 外部使用就变成n、s、p了
export {
	name as n,
  sayHello as s,
  Person as p
};
```

### 静态import和动态import区别

直接举例：

1. 在export为default导出时如下，由此可见，静态导入时，拿到的那个东西就是default对应的东西。动态导入时，拿到的是一个Module对象，该对象有一个default属性。

   ```javascript
   // hi.js
   export default function () {
     console.log('hi');
   }
   
   // bye.js
   export default function () {
     console.log('bye');
   }
   
   // 静态导入
   import hiModule from './hi.js'
   
   // ƒ () {
   // 	 console.log("hi");
   // }
   console.log(hiModule);	
   
   // 动态导入
   async function main() {
     const byeModule = await import('./bye.js');
     // Module {
     //   default: f () 
     // }
     console.log(byeModule);
   }
   ```

2. 在export为普通导出时，

   ```javascript
   // hi.js
   export function hi() {
     console.log('hi');
   }
   
   // bye.js
   export function bye() {
     console.log('bye');
   }
   
   // 静态导入
   // 只能这么获取，{}不能少，否则会提示不是default导出
   import { hi } from './hi.js'
   // hi方法
   console.log(hi);
   
   // 动态导入
   async function main() {
     const byeModule = await import('./bye.js');
     // Module {
     //   bye: f ()
     // }
     // 是一个Module对象，其中bye属性就是bye方法
     console.log(byeModule);
   }
   
   
   // 下面这段代码非常之秀
   async function main() {
     // 因为动态导入，返回的是Module对象，从中取default并重命名为bye，这个bye就是bye模块导出的默认方法
     const { default: bye } = await import("./bye.js");
     bye();
   }
   ```

### 动态导入不是函数调用

import()并不是一个函数调用，它只是一种特殊语法，只是恰好使用了括号，import并不是一个变量，不能对其使用apply/call。

### 带async和defer的script标签

#### defer

1. defer属性仅对外部script生效，内联script将被忽略
2. 带defer的script标签不阻塞DOM构建，会先在后台下载，在DOMContentLoaded事件触发之前执行
3. 多个带defer的script标签包保持顺序

#### async

1. async属性仅对外部script生效，内联将被忽略
2. 带async的script标签不阻塞DOM构建，它是独立的，不会等待其他任何东西，其他东西也不会等待它，下载完就执行，可能在DOMContentLoaded之前，也可能在之后。
3. 多个带async的script标签不保证顺序，谁先下载完谁就执行

#### 动态脚本

动态脚本默认是async的，但可以将async属性赋为false，多个动态脚本就会保持执行顺序，又由于是放在body元素的最后，所以和defer的效果一样，例：

```javascript
function loadScript(src) {
  let script = document.createElement("script");
  script.src = src;
  script.async = false;
  document.body.append(script);
}
loadScript("./main.js");
loadScript("./lib.js");
```

由于async=false，main.js一定先于lib.js执行。

#### 总结

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-03-20 12.12.51.png" alt="截屏2023-03-20 12.12.51" style="zoom: 50%;" />

### Object.defineProperty和Proxy的区别（小红书）

一般都是在讨论在Vue中的区别，也就是它们在实现数据劫持时的区别。

1. `Object.defineProperty`代理对象的某个属性，想代理多个属性需要全部遍历，`Proxy`一次代理整个对象；
2. `Object.defineProperty`无法感知新增属性，`Proxy`可以感知新增属性；
3. `Object.defineProperty`无法感知数组元素的新增和删除，以及`length的修改`，Proxy可以感知数组元素的新增、删除、`length`修改；
4. `Object.defineProperty`中用到的this指向目标对象，Proxy指向代理对象；
5. `Object.defineProperty`可兼容IE，Proxy对低版本浏览器不支持；
6. Proxy的劫持方法多，`Object.defineProperty`只有getter和setter。

造成这些区别的最核心原因就是`Object.defineProperty`方法本身就仅仅是定义对象的属性，并为其赋予描述符Descriptor（有setter、getter）

贴上`Object.defineProperty`实现对象和数组劫持的代码：

```javascript
const list = [1, 2, 3, 4, 5];

const o = {
  name: "fan",
  age: 20,
  year: 96,
  month: 2,
  day: 3,
};

// 对象
for (let key in o) {
  let val = o[key];
  Object.defineProperty(o, key, {
    get() {
      console.log("getter object");
      return val;
    },
    set(newVal) {
      console.log("setter object");
      val = newVal;
    },
  });
}

// 数组
list.forEach((ele, idx) => {
  Object.defineProperty(list, idx, {
    get() {
      console.log("getter list");
      return ele;
    },
    set(newVal) {
      console.log("setter list");
      ele = newVal;
    },
  });
});
```

### Reflect是什么，和Proxy搭配使用

Reflect提供了与内置方法`[[get]]、[[set]]`（当定义了访问器属性时，`[[get]][[set]]`就会被访问器属性覆盖）一样的能力，当你取一个对象的属性时`obj.name`，它实际是调用了内置方法`[[get]]`，但这个get是无法被我们显市调用的。Reflect就可以让我们显式调用这个`[[get]]`：`Reflect.get(obj, name)`，这和`obj.name`是一样的效果。

和Proxy搭配使用，有什么优势呢？

```javascript
// 直接启用内置[[get]]
let user = {
  _name: 'fan',
	get name() {
    return this._name;
  }
};

user = new Proxy(user, {
  get(targe, prop, receiver) {
    return target[prop];
  }
});

// 显示调用[[get]]
user = new Proxy(user, {
  get(targe, prop, receiver) {
    // prop是访问器属性，this将会是receiver，而不是target
    // 反正都是要调get name() {}，可能会想到用apply或call，但get name() {}是getter，是不能被显示调用的
    return Reflect.get(targe, prop, receiver);
  }
});
```

上面其实都相当于是调了`[[get]]`，但Reflect.get的优势是，当reciver是其它对象时，可以取到其它对象的属性，相当于在`this=receiver`的上下文下执行，否则是在this=target这个上下文执行访问器属性`name`。

下面是一个透明包装器，proxy将取值转发给user目标对象

```javascript
let user = {
  _name: 'fan'
};
user = new Proxy(user, {
  get(target, prop) {
    return target[prop];
  }
});

// 'fan'
console.log(user._name);
```

Reflect旨在补充Proxy，任何一个Proxy捕捉器都用带有相同参数的Reflect调用，我们应该使用它们将调用转发给目标对象。

让一个对象变成observable的，实现观察者模式

```javascript
function makeObservable(target) {
  /* 你的代码 */
  const handlers = Symbol("handlers");
  target[handlers] = [];

  target.observe = function (fn) {
    target[handlers].push(fn);
  };

  return new Proxy(target, {
    set(target, prop, value) {
      const isSuccess = Reflect.set(...arguments);
      if (isSuccess) {
        target[handlers].forEach((fn) => {
          fn(prop, value);
        });
      }
      return isSuccess;
    },
  });
}

let user = {};
user = makeObservable(user);

// 没有拦截observe，直接转发给原目标对象
user.observe((key, value) => {
  console.log(`SET ${key}=${value}`);
});

user.name = "John"; // alerts: SET name=John
```



### let、const、var的区别

1. let、const是块级作用域，var是函数作用域；
2. let、const存在暂时性死区，var不存在，因为它会变量提升；
3. let、const不可重复声明，var可以；
4. let、var可以不需要初始值，可变，而const必须有初始值，且该值不可变（引用）；
5. 在全局，var声明的变量会放在window中（全局执行上下文的变量环境的环境记录器应该就是window），let和const声明的变量放在全局执行上下文的词法环境的环境记录器中；

### 谈谈作用域

作用域就是一个变量所作用的范围，也就是程序能够访问到这个变量的范围，实际上是用词法环境实现的：

每个运行的函数、代码块、整个script脚本，都对应一个词法环境的内部关联对象，词法环境由环境记录器和外部词法环境引用两部分组成，环境记录器中存放着声明的本地变量，外部词法环境引用指向外层的词法环境。

当代码要访问一个变量的时候，首先搜索内部词法环境，然后依次向外层词法环境搜索。

当一个函数被创建的时候，就会在这个函数对象里保存一个`[[Environment]]`属性，它指向该函数创建时所在的词法环境，到该函数被执行时，就会创建自己的词法环境，该词法环境指向`[[Environment]]`词法环境。

如果一个词法环境不再被任何变量引用，就会被回收。

### 闭包

一个函数及其可访问到的周边环境组成闭包，函数被创建时，闭包也同时被创建，闭包可以在函数内部访问外部变量。为什么js中的函数都是闭包呢？是因为每个函数在诞生的时候都会用一个`[[Environment]]`属性指向其创建时的词法环境，在函数被执行的时候，会创建函数自己的词法环境，并指向外部词法环境。

注：用`new Function()`创建的函数，其`[[Environment]]`指向的是全局词法环境。

### 闭包有什么用

一个函数，在它执行完之后，就会从执行栈中退出来，函数内的变量无法使用，如果想在该函数外部还能使用到内部的变量，就可以利用闭包。

### 什么是变量提升

用var和function生明的变量，可以在声明它们之前访问。js代码被执行时，会创建全局执行上下文，随后执行，在创建阶段，js引擎将var和function的声明移到顶层，这就是js的变量提升。

注：var声明的变量会给一个undefined，function一开始就是完整的函数，let、const、函数表达式、箭头函数都不会变量提升。var和function对于重复的声明都会后者覆盖前者（非严格模式下）。

### js数据类型

原始类型7 + 引用类型1：string、number、boolean、undefined、null、bigint（以n结尾的数）、symbol、object

### 原始数据类型和引用数据类型的区别

1. js引擎会为变量分配两种内存，原始值变量会被分配固定大小的内存并存入栈中，而引用类型变量会将引用值存入栈中，而将引用的对象存入堆中；
2. 复制原始类型的变量，会创建一个独立的值，而复制引用类型，会将引用复制给新变量，但它们指向的对象是同一个；
3. 可对引用类型的属性进行操作，原始类型不行。

### 为什么0.1 + 0.2 === 0.3为false

因为js存储数字的方式，导致0.1实际的二进制表示并不精准，计算后的结果也就不是精确的0.3。

解决方式，`+sum.toFixed(2)`、乘以10或100，计算后再除10或100、利用Number.EPSILON，当计算值与预期值之差小于Number.EPSILON时，就返回true。

### undefined和null的区别

从定义上讲，undefined表示没有任何值	，null表示没有任何对象，即空引用。当某些东西没有值时，js通常默认为 `undefined`。

### 为何不能直接给变量赋值undefined

undefined是标识符，它是window下的一个属性，也就是一个全局变量，而null是关键字。

在低版本浏览器上，全局变量undefined可以直接被赋值，undefined也可以在函数中被用作变量，被赋值为其它值，造成一些错误，因此想要让一个变量的值为undefined，就用void 0，void + ‘表达式’这种结构会返回undefined。

### 运算符?? 、??= 和 || 的区别

?? 是一个逻辑运算符，左边的值如果是undefined或者null，就返回右侧的值，否则返回左侧的值。|| 不局限于undefined和null，而是所有的空值（0, '', NaN等）。

a ??= b，表示如果a为undefined或null，就将b赋值给a，否则a还是原值。

### typeof NaN 和typeof null

typeof NaN为number

typeof null为object

### js的隐式转换

#### [==规则](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Equality)

js的隐式转换主要是+运算符以及==运算符。

+作一元运算符时会转换为数字，+作二元运算符时会将操作数转换为字符串（若其中有引用类型，则先用valueOf，再用toString），再拼接。

== 引用类型和基本类型比较时，引用类型会转换为基本类型，先valueOf，后toString，注意数组调用toString的结果会不一样。

这里简单记住一点：引用类型和基本类型比较时，引用类型的值会被转换为基本类型，此时先调用valueOf，如果其结果不是基本类型，就调用toString。

如果是number和其他类型比较，则另一个会被转换成number，比如空字符串变为0。

注意NaN和NaN不相等，不管是==还是===。

### ==、===、Object.is()的区别

== 会有类型隐式转换，转换后再比较；

=== 是严格相等，无类型转换，若类型不相同直接false；

`Object.is()`除了`Object.is(NaN, NaN)`为true，`Object.is(-0, +0)`为false外，其它和===相同。

### 判断数据类型的方式

1. typeof
2. instanceof，a instanceof b 用于判断a指向的原型是否在原型链上（截止到b的原型）
3. Object.prototype.toString.call(arg)

### ||=、&&=、??= 分别是什么

a ||= b，a为假，就把b赋值给a；

a &&= b，a为真，就把b赋值给a；

a ??= b，a为undefined或null，就把b赋值给a；

### 用new创建对象的时候发生了什么

1. 在内存中创建一个新的对象；
2. 将该对象的`[[prototype]]`特性指向构造函数的原型；
3. 将构造函数中的this指向刚创建的新对象；
4. 执行构造函数；
5. 如果构造函数返回对象（不为null），则结果是返回的对象，如果不返回对象，则结果就是刚创建的对象

### apply、call、bind的区别

apply、call、bind的区别就是：apply和call是用来改变上下文对象的（即相关函数的this指向的），而bind会返回一个新函数，并且已经制定了上下文对象。apply和call的区别在于apply需要传入的参数是一个数组或类数组对象，call传入的是参数列表（多个参数）。

call比apply的性能要好，call传入参数的格式正是内部所需要的格式，不需要如apply一样还要再解构参数，即apply源码中多了CreateListFromArrayLike的调用，其他基本一样。

一个函数，就是一个函数，不依赖谁，它是独立的，它的this和外层函数的this无关，只要调用它时没有用apply、call、bind或者上下文对象，那么这个函数内的this就指向全局对象window或者global。

### 创建对象的方法

1. Object构造函数；
2. 对象字面量；
3. Object.create()；
4. 类
5. 工厂模式
6. 构造函数模式
7. 原型模式
8. 组合模式

### 有哪些继承方式

所谓继承，就是子拥有父的属性和方法

1. 原型链继承，子构造函数的原型指向父构造函数的原型

2. 借用构造函数，在子构造函数中用call调用父构造函数，于是子构造函数就拥有了父构造函数中的属性、方法

3. 组合继承，融合原型链和借用构造函数，把公共方法放在原型上，其它借用构造函数

4. 寄生式继承，对被寄生对象做增强

   ```javascript
   function createObj (o) {
     let clone = Object.create(o);
     clone.sayName = function () {
       console.log("hi");
     }
     return clone;
   }
   ```

5. 寄生组合式继承。

### Map和WeakMap的区别

1. Map的键可以是任意值，WeakMap的键只能是对象；
2. Map的键值对保存插入顺序，WeakMap只是一个集合，无序；
3. Map的键可枚举，WeakMap的键不可枚举，因此WeakMap就获取不到有哪些key以及key的长度；
4. WeakMap的键是对象的弱引用，如果该对象没有被任何一个其它的变量引用，GC就会回收它

### Map和Object的区别

1. Object的属性只能是string，Map可以的键可以是任意类型；
2. Map可以很快获取长度，Object要手动计算；
3. Map的键值对是保留插入顺序；
4. 因为Object有原型，所以会有一些默认的属性存在

### 浅拷贝方法

1. 赋值运算符 =；
2. 扩展运算符 ...；
3. `.slice()`；
4. `Object.assign()`；
5. `Array.from()`

### 深拷贝方法

1. `JSON.stringify`和`JSON.parse`序列化，注意function、undefined、symbol会被忽略；
2. 广度优先；
3. 深度优先；
4. lodash库的cloneDeep；
5. rfdc库

### 深拷贝时出现循环引用怎么办

举例什么是循环引用：

```javascript
let fan = {
    name: 'fan',
    friend: gen
};
let gen = {
    name: 'gen',
    friend: fan,
}
```

以上，在拷贝gen的时候，friend是fan，然后去拷贝fan，发现里面的friend是gen，它们互相引用了，会没完没了地拷贝下去。

为了处理这种情况，采用**WeakMap**，比如当前拷贝gen，就把gen加入WeakMap中，下次再遇到有人引用gen就直接取这个WeakMap中的gen，而不用重新复制。

为什么不用Map而使用WeakMap呢？因为WeakMap会清掉没有被引用的键值对，也就是说如果我拷贝gen的时候，把它加入WeakMap，但是并没有遇到有人引用gen，那gen就可以被垃圾回收机制周期性清除了，节省空间。

### this指向

1. 全局上下文，this指向全局对象，浏览器中就是window；

2. 函数上下文
   1. 作为对象的方法调用，this指向该对象；
   2. 作为普通函数调用，this指向全局对象或undefined（非严格模式）；
   3. call、apply、bind，this指向绑定的对象；
   4. 作为构造函数调用，this指向创建的对象；

3. 在类中，this指向新创建的对象；

4. 箭头函数中，this和封闭词法环境的this保持一致，词法环境就三种，全局、函数、块级，且this只取决于函数和全局；

5. 原型链中的this，如果函数在原型上，那么this依然是调用这个方法的对象；

6. this在事件处理函数中，指向的是触发事件的元素；

7. getter和setter中的this指向获取或设置属性所在的对象；

   ```javascript
   var length = 10;
   function fn() {
     return this.length + 1;
   }
   var obj = {
     length: 5,
     test1: function () {
       return fn();
     },
   };
   obj.test2 = fn;
   
   console.log(obj.test1.call()); // 11
   console.log(obj.test1()); // 11
   console.log(obj.test2.call()); // 11
   console.log(obj.test2()); // 6
   ```

### 将类数组转换为数组的方法

函数的arguments就是一个类数组，它还提供了lengh属性

1. 扩展运算符 `[...arguments]`；
2. `Array.from(arguments)`；
3. `Array.prototype.slice.call(arguments)`；
4. `[].slice.call(arguments)`

### 箭头函数和普通函数的区别

箭头函数内的`this`和外层的`this`保持一致，一旦确定，就不会再被更改。

无法使用arguments对象，该参数在函数体内不存在。

不可以使用yield命令（就是不能用来当作Generator函数）。

不可以使用new命令，会报错，因为：

1. 没有自己的this
2. 没有prototype属性，而new命令会将构造函数的prototype赋值给新实例的`__proto__`

### 简单介绍ES6的Iterator和Iterable

可枚举（enumerable）和可迭代（Iterable）是不一样的。迭代用for of

ES6规定了可迭代协议（Iterable）和迭代器协议（Iterator），可迭代协议就是允许对象定义自己的迭代行为，要成为可迭代对象，必须实现迭代器方法；于是就有了迭代器协议，它规定了迭代器方法应该是什么样子，方法名是`[Symbol.iterator]`，该方法需返回一个对象，且对象里必须实现next方法，next返回{value, done}的格式，done的值取决于是否还能返回下一个值。

内置也有一些可迭代的对象，比如String、Array、Map、Set等，还有Generator函数返回的迭代器对象，普通的Object对象是不可迭代的，但可以使其变成可迭代的，添加[Symbol.iterator]，但最好是用`Object.entries()`。

### for in 和for of的区别

1. for in是列出可枚举的属性名（key），也会列出该对象原型上的可枚举属性名，如果是数组，则列出的是索引（index）；
2. for of，只有实现了Iterator接口的数据类型才能够被迭代，ES6的对象无法使用for of，另外，for of列出的是值（value）；

### 对Generator函数的理解

1. 普通函数只能返回一个值，或不返回值，而Generator可以按需一个接一个返回多个值；

2. 普通函数不可暂停，Generator函数可暂停；

3. Generator靠next()来控制数据流，每执行一次next()，yield将返回值添加到{ value, done }的value上；

4. yield* 可以返回数组的每个元素；

5. 可以用for...of迭代，return后面的值不会被迭代到

   ```javascript
   const arr = ['a', 'b', 'c'];
   
   function* generator() {
       yield 1;
       yield* arr;
       yield 2;
     	return 3;
   }
   
   for (let value of generator()) {
       console.log(value)
   }
   
   // 1 a b c 2
   ```


### filter、find、some

filter返回数组中满足条件的值组成的新数组

find返回数组中满足条件的第一个值

some判断数组中是否有某个值

### 谈谈事件循环（小红书）

#### 把握两点：

1. js是单线程语言
2. Event Loop是js的执行机制

之所以有事件循环，就因为js是单线程的，会有很多同步任务和异步任务，如何规划这些任务的执行顺序，就成了事件循环要做的事。

所有的js代码可以分为宏任务和微任务

1. 宏任务（macro-task）：整体代码`script`、`setTimeout`、`setInterval`
2. 微任务（micro-task）：`Promise`相关、`process.nextTick`

进入整体代码（整体代码被看作是一个宏任务）后，开始第一次事件循环，执行宏任务中的所有代码，完了之后接着执行微任务队列里的微任务。然后再从宏任务队列中取一个宏任务开始执行，在这个宏任务执行完毕后，又接着执行依附于它的微任务（在执行该宏任务的过程中送进微任务队列的微任务），此为第二次事件循环，如下图。

```javascript
setTimeout(function() {
    console.log('setTimeout');
}, 3000);

new Promise(function(resolve) {
    console.log('promise');
    resolve();
}).then(function() {
    console.log('then');
});

console.log('console');
```

以上代码的执行步骤：

1. 整体代码作为一个宏任务开始执行。
2. 遇到setTimeout，它是宏任务，3秒后将其回调函数加入宏任务队列（主线程不参与这个过程，等待的这3秒主线程已经去干其它事情）。
3. 遇到Promise，立刻执行回调函数，输出‘promise’。
4. 遇到then，它是微任务，主线程继续往下执行，**Promise对象状态改变后，会自动将then的回调函数加入微任务队列。**
5. 执行最后一句，输出'console'。
6. 执行当前微任务队列里的所有任务，发现队列中有then微任务的回调函数，输出‘then’。
7. 第一轮事件循环结束，查看宏任务队列，开始第二轮事件循环。
8. 执行setTimeout宏任务的回调函数，输出‘setTimeout’。
9. 结束。

### ES6的新特性

1. let、const、块级作用域
2. 数组解构、对象解构
3. 箭头函数
4. 扩展运算符（迭代）
5. Proxy
6. Reflect
7. class类、Set、Map、Symbol
8. for...of
9. 可迭代接口Iterable
10. Generator函数
11. Async函数
12. 字符串扩展方法（includes、startWith、endWith）
13. 对象字面量增强（key和value相同可省略value）
14. `Object.is`、`Object.assign`

### 谈谈对TS的理解

1. 是什么？TS是JS的超集，扩展了JS的语法，它是一种静态类型检查的语言，在编译阶段就可以检查出类型的错误；
2. 特性？类型检查、类型推断、类型擦除、接口、枚举等
3. 区别？文件名后缀的区别、ts会被编译成js

### 说几个Promise的静态方法

1. `Promise.all()`
2. `Promise.race()`
3. `Promise.any()`
4. `Promise.resolve()`
5. `Promise.reject()`

### 说几个Promise的实例方法

1. `Promise.prototype.then()`
2. `Promise.prototype.catch()`
3. `Promise.prototype.finally()`

### 知道哪些前端性能指标

1. FP（First Paint，首次绘制），指屏幕上首次发生视觉变化的时间；
2. FCP（First Contentful paint，首次内容绘制），表示浏览器第一次向屏幕绘制“内容”（文本、图片等）
3. LCP（Largest Contentful Paint），绘制最大文本或图片的时间；
4. FMP（First Meaningful Paint，首次有效绘制），页面主要内容开始出现在屏幕上的时间点
5. TTI（Time to Interactive，可交互时间），表示网页第一次达到可交互状态的时间点，且主线程此时已达到流畅
6. TTFB（Time to First Byte），浏览器接收第一个字节的时间

![img](https://pic1.zhimg.com/80/v2-a19a1f23efe0c4d83564e3951cd1fad4_1440w.webp)

### js的本地存储方式

Cookie、localStorage、sessionStorage、indexedDB

### Symbol的用处

1. 避免命名重复
2. 在call、apply中有用到
3. 在迭代器迭代函数名用的就是`Symbol.iterator`
