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





