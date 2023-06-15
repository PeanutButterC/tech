### DOM的更新不是同步的

当更新了响应式状态后，并不会立刻更新DOM，Vue将会缓冲到更新周期的下个时机，确保无论进行了多少次状态更改，每个组件都只更新一次。如果要获取状态更新后的DOM，用nextTick

### Vue3中的响应式对象都是Proxy实现的

下面this.obj是一个Proxy，并不是属性值本身

```javascript
ddata() {
  return {
    obj: {}
  }
}
mounted() {
	const newObj = {};
  this.obj = newObj;
  // false
  console.log(this.obj === newObj);
}
```

### Vue中的状态都是默认深层响应式的

深层的对象、数组，都将是响应式的，这句话的意思是，对象内部某个属性如果被更新，那么Vue会觉察到它的变化从而刷新DOM（如果DOM关联了该属性）

### 有状态方法的创建时机

比如防抖函数，可以把处理程序写在methods中，在created钩子里创建防抖函数（created时，methods已经完毕），为什么要在created里创建这个防抖函数呢，因为methods中的方法会被复用，为了防止多个组件共享同一方法而共享计时器状态，导致混乱，因此为每一个组件实例都创建一个属于自己的防抖函数。

像防抖函数这类函数，处理程序是通用的，但每个组件实例又必须有自己的状态。要解决这个问题，可以考虑在created阶段创建属于组件自己的实例方法。

### lodash-es包中的debounce方法是已经包装好的防抖构造器

```javascript
// 引入
import { debounce } from 'lodash-es';

// 使用
created() {
  this.debounceClick = debounce(this.click, 1000);
}
methods: {
  click() {
    console.log('debounce click');
  }
}
```

### 计算属性vs方法

两种方式的结果完全相同，区别在于计算属性会根据其响应式依赖进行缓存，一个计算属性仅在其响应式依赖更新时，才会重新计算，因此无论访问多少次该计算属性，都会返回缓存的结果，不会重复执行getter。

而方法，总会在重渲染时，重新执行。

计算属性会用空间换时间，提升时间性能。

### 可写计算属性示例

我以前有一个疑问，纠结于computed是在mounted前还是后、或者和其他钩子的顺序，可能是纠结于在mounted阶段，计算属性是否已经计算出值。不过搞清计算属性的定义之后，发现它就是一个方法，第一次访问或者其响应式依赖更新时才会执行。那么，就没有必要讨论其和mounted等钩子的先后顺序，因为计算属性根本就不会像钩子一样自发执行，它只有在被访问的时候，才有可能会被执行。在mounted中访问计算属性，当然没问题，当然是有值的，要么执行函数获得值，要么取缓存的值。

计算属性默认只读，尝试修改计算属性会收到警告，除非一些特殊场景：

```javascript
data() {
  return {
    firstName: 'yifan',
    lastName: 'zhuo'
  }
}
computed: {
  fullName: {
    // getter
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
    set(newVal) {
      // 解构写法
      [this.firstname, this.lastName] = newVal.split(' ');
    }
  }
}
```

但我们仍不提倡修改计算属性。

### updated钩子执行时机

我们知道Vue有一个概念叫模板template，我曾一直困惑，Vue状态改变，updated钩子什么时候执行，是在页面刷新之后？还是之前？

触发setter(状态更新) -> 通知watch -> 触发re-render -> 生成new vnode(vdom) -> patch（更新真实DOM）-> 触发updated钩子 -> 触发nextTick回调（微任务）

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-01-02 00.38.06.png" alt="截屏2023-01-02 00.38.06" style="zoom: 33%;" />

显然，updated会在DOM更新后被执行一次。re-render就是刷新模版template（生成新的vdom）。

computed是计算属性，它应该是在re-render的时候才计算新值（或者取缓存），和watch不一样，不过不管是computed还是watch，一定都是在更新DOM前就要完成的。

### watch及附带的async函数知识点、对嵌套状态的监听

我们通常需要在状态变化后，执行一些副作用，比如更改DOM，因为状态变化后，通知watch -> 触发re-render。

又或者需要做一些异步操作，异步操作通常会用async，除非有必要，调用async函数时，前面是不需要await的，只有async函数内部需要await，意思是在外部作用域，调用async函数即可，还是会接着执行下面的代码，async内部有等待，那是微任务，会之后再执行。示例：

```javascript
  watch: {
    'nested.question': {
      handler(newQuestion, oldQuestion) {
        if (newQuestion.includes('?')) {
          // 调用不需要加await，因为本行之后的代码不依赖本行的结果
          this.getAnswer();
        }
      }
    }
  },
  methods: {
    async getAnswer() {
      try {
        this.nested.answer = 'Thinking...';
        const res = await fetch('https://yesno.wtf/api');
        this.nested.answer = (await res.json()).answer;
      } catch (e) {
        this.nested.answer = 'Error! Could not reach the API. ' + e;
      }
    }
  },
```

上面代码的'nested.question'，如果仅仅是想监听深层的某个状态，watch时可直接用 . 连接表示路径即可，不支持表达式。

### watch深层监听

watch默认是浅层监听，如果监听一个对象obj，那么除非该obj被完全替换，否则仅仅是其内部的属性更新将不会触发watcher，想要实现深层监听，即内部属性变化，也会触发watcher，需加deep。

```javascript
data() {
  return {
    obj: {
      name: 'fan',
      age: 1
    }
  }
}
watch: {
  obj: {
	  handler(newObj, oldObj) {  
    	
    },
    // obj新增、删除、更新属性，都会触发handler，但除非obj被完全替换，否则newObj和oldObj是同一个对象
    deep: true
  }
}
```

#### ！！谨慎使用深层监听

留意性能，深层监听会遍历对象的所有属性，大型数据结构时，比较费性能

### 即时回调的watcher执行handler的时机、flush的作用

watch默认是懒执行的，仅当数据变化时，才会执行回调，但有时我们希望创建完watcher就立刻执行handler，可用immediate: true实现。

执行时机就在created钩子之前，因为执行created钩子时，Vue已经处理了data、computed、methods，也就是说watcher也已经创建好了，可在此时执行watcher的handler。

### 默认状态下的watcher执行handler的时机

因为一般情况会有：响应式状态变更 -> 通知watch -> 生成新vnode -> dispatch

所以默认情况下，watcher的handler里访问DOM都是旧的DOM，如果想在handler里访问被Vue更新过的DOM，也就是状态变化后，先更新DOM，再通知watcher执行handler，需要加flush: 'post'。

#### !!问题及猜测

updated钩子是patch之后调用，这个规则肯定不变。不过设置flush: 'post'之后，被监听状态的handler先于updated钩子执行，这是为什么？

猜测是updated钩子不管怎样都仅执行一次。

原来的顺序：响应式状态变化 -> 触发watcher -> re-render -> 生成vnode -> patch -> updated钩子

在加了flush: 'post'之后，会变成：响应式状态变化 -> re-render -> 生成vnode -> patch -> 触发watcher -> updated钩子。

这不过是handler放在patch后执行罢了，依然还是在updated钩子之前。

### 命令式地创建watcher

this.$watch()，应用场景是根据一些条件来确定是否添加watcher。

用选项式watch或命令式创建watcher，宿主组件卸载时会被自动清除，一般情况无需关心何时停止他们，不过也可以用**unwatch()**来主动停止watcher。

### MVVM模型

Model-View-ViewModel（MVVM）

在MVVM架构下，View和Model没有直接联系，而是通过ViewModel进行交互，View数据的变化会同步到Model中，Model的变化也会反映到View上。

ViewModel通过双向数据绑定把View层和Model层连接起来，View和Model的同步工作完全是自动的，无需人为干涉，开发者只需要关注业务逻辑，不需要手动操作DOM，也不需要关注数据状态的同步问题，复杂的数据状态维护完全由MVVM来统一管理。

### 对Vue的理解（Vue的特性）

#### 数据驱动（MVVM）

#### 组件化

1. 降低系统耦合度
2. 方便调试，每个组件职责单一，出现问题可以迅速定位到某一个组件
3. 可维护性强，每个组件职责单一，并且会被复用，对组件的更新可获得整个系统的升级。

#### 指令系统

带有v-前缀的特殊属性，当它表达式的值改变时，将会产生连带影响，响应式作用于DOM，如果没有指令系统，我们要先获取DOM，再做一些操作。

#### Vue和传统开发的区别

1. Vue所有的界面事件，都只会操作数据，而传统开发需要操作DOM
2. Vue所有的界面变动，都是数据自动绑定的，传统开发需要操作DOM

### Vue和React的相同点和区别？

相同点：

1. 都使用虚拟DOM
2. 都使用组件化思想
3. 单向数据流
4. 都支持服务端渲染

区别：

1. React推崇不可变数据，Vue是可变数据
2. 父子组件通信React用回调函数，Vue还可以用事件触发
3. React推崇函数式编程，Vue有很多指令
4. diff算法不同

### 对SPA单页面的理解

SPA（Single-Page-Application）通过动态重写当前页面来与用户交互，避免页面之间的切换，一般就只有一个html文件，通过前端路由实现无刷新跳转。加载页面的时候，会统一加载js等资源（也可能异步加载），通过监听url的hash来实现内容切换。

#### SPA和MPA的区别

多页面应用中，有多个页面，当访问另一个页面时，都要重新加载htm、js、css。

|                 | 单页面应用（SPA）         | 多页面应用（MPA）                   |
| :-------------- | :------------------------ | ----------------------------------- |
| 组成            | 一个主页面和多个页面片段  | 多个主页面                          |
| 刷新方式        | 局部刷新                  | 整页刷新                            |
| url模式         | 哈希模式                  | 历史模式                            |
| SEO搜索引擎优化 | 难实现，可使用SSR方式改善 | 容易实现                            |
| 数据传递        | 容易                      | 通过url、cookie、localStorage等传递 |
| 页面切换        | 速度快，用户体验良好      | 切换加载资源，速度慢，用户体验差    |
| 维护成本        | 相对容易                  | 相对复杂                            |

#### SPA的优缺点

优点：

1. 无刷新就可切换内容，提高用户体验（重新请求页面，是需要发网络请求的，很耗时），代码都放在本地了，能不快吗
2. 符合前后端分离思想

缺点：

1. 不利于SEO（搜索引擎优化），因为数据都是通过请求接口动态渲染的，搜索引擎检索的时候，我只有一个`<div id="app"></div>`，其他啥也没有
2. 首屏速度会受影响。

### vue-router的hash模式和history模式

详细解释在https://zhuanlan.zhihu.com/p/337073166，此处只做概括。

既然是SPA，就假设是一次性加载了全部的js bundle，SPA讲究的需要保证浏览器不要重新加载页面，也就是不要发请求，按照常理如果更换url，刷新浏览器，那浏览器一定会发送请求。

由于前端路由只能依赖url的变化，如果想要路由，就必须更换url，同时又不能让浏览器发请求（重新加载页面）。

vue-router巧妙利用浏览器已有的两种方式，实现了前端路由

1. 第一个方式就是锚点（#），因为在改变锚点后，浏览器是不会刷新页面的，但确确实实是改变了url，但回车刷新浏览器的话，请求是不带锚点的，vue-router实现的核心是监听了hashchange事件，一旦锚点改变，vue-router就会根据新的锚点来加载新的vue组件；

2. 第二个方式是HTML5的History Interface，可以改变url而不刷新页面，其核心是pushState()和replaceSate()，这两个方法可以改变History对象，但却可以不发请求。使用这种方式需要在后端配置一个回退路由，将不存在的路径请求重定向到入口文件（index.html）。

从表象上看，hash带有#，而history模式不带#，比较干净，适合推广，history模式访问二级目录的时候，会404，因为这是前端路由，后端实际上是没有与之匹配的资源的，需要后端配合重定向到首页。

为hash和history做一个对比：

|          |            hash            |  history   |
| :------: | :------------------------: | :--------: |
| url显示  |            有#             |    无#     |
| 回车刷新 | 可以加载到hash值对应的页面 |  一般404   |
| 支持版本 |    支持低版本浏览器和IE    | HTML5新API |

### 实现SPA前端路由

1. hash：思路是hashchange事件
2. history：思路是history.pushState()和history.replaceSate()

### 服务端渲染SSR（Server Side Render）（频率：1）

Vue本是用于构建客户端应用的框架，默认情况下只负责操作浏览器DOM，然而Vue也支持在服务端直接将组件渲染成HTML字符串，作为响应返回给浏览器，最后在浏览器中将静态的HTML“激活”为可以交互的客户端应用，这就是服务端渲染。

这种应用被认为是通用的，通用的意思是说大部分代码既可以运行在服务端，也可以运行在客户端。

#### 服务端渲染优势

1. 更快的首屏加载，这是因为数据获取过程首次访问时在服务端完成，相比客户端获取数据，有更快的数据库连接；
2. 前后端一套框架，通俗讲就是前端代码也可以在后端跑；
3. 更好的SEO，搜索引擎爬虫可以直接看到完全渲染的页面

#### 一段服务端渲染demo

```javascript
// 通用代码app.mjs
import { createSSRApp } from "vue";

export function createApp() {
  return createSSRApp({
    data: () => ({
      count: 0,
    }),
    template: `<button @click="count++">{{ count }}</button>`,
  });
}
```

```javascript
// 服务端渲染代码
import { createApp } from "./app.mjs";
import express from "express";
import { renderToString } from "vue/server-renderer";
import { dirname } from "path";
import { fileURLToPath } from "url";

const __dirname = dirname(fileURLToPath(import.meta.url));

const server = express();

server.get("/", (req, res) => {
  const app = createApp();
  renderToString(app).then((html) => {
    res.send(`
            <!DOCTYPE html>
            <html>
                <head>
                  <title>Vue SSR Example</title>
                  <script type="importmap">
                    {
                      "imports": {
                        "vue": "https://unpkg.com/vue@3/dist/vue.esm-browser.js"
                      }
                    }
                  </script>
                  <script defer type="module" src="./index.js"></script>
                </head>
                <body>
                  <div id="app">${html}</div>
                </body>
            </html>
    `);
  });
});

// 托管
server.use(express.static(__dirname));

server.listen(3000, () => {
  console.log("ready...");
});

```

```javascript
// 前端执行
import { createApp } from "./app.mjs";

const app = createApp();
app.mount("#app");
```

### 预渲染SSG（Static Site Generation）

SSR会在每次请求都在服务端进行一次渲染，而如果一个页面对每个用户来说展示效果都相同，就可以只渲染一次，在构建的过程中就完成渲染，不需要每次请求都渲染一遍，预渲染后的页面作为静态HTML被服务器托管。

注意SSG仅可以用在静态数据的页面，因为一次构建，页面就定了，不会再改变，想更新页面需要重新构建然后部署。

### SEO的方法有哪些（几种常见的）

1. SSR服务端渲染
2. SSG预渲染
3. 使用Phantomjs针对爬虫特殊处理，先判断是否为爬虫，如果是的话，就将请求转发到一个node server，再通过PhantomJS来解析完整的html返回给爬虫

### v-if和v-show

#### 共同点

表达式为false时都不占据页面位置，为true时都会占据。

#### 区别

1. v-show采用css的display: none来决定显示隐藏，v-if是将整个dom元素添加或删除
2. 切换显示隐藏状态时，v-if有局部的编译/卸载的过程，内部子组件、事件都会重建/销毁；v-show仅是css的切换
3. v-if只有到表达式为true时才会渲染，v-show不论是否true，都会渲染
4. v-show状态切换不会触发组件的生命周期，v-if会触发各种钩子

#### 性能

v-if有更高的切换消耗，v-show有更高的初始渲染消耗。

### Vue挂载过程

1. `new Vue()`之前，会先创建Vue构造函数，在其prototype上放一些公共方法，具体如下：

   ```javascript
   function Vue(options) {
     if (__DEV__ && !(this instanceof Vue)) {
       warn('Vue is a constructor and should be called with the `new` keyword')
     }
     this._init(options)
   }
   initMixin(Vue)	// 在原型上定义_init方法
   stateMixin(Vue)	// 在原型上定义$data、$props、$set、$delete、$watch等公用方法
   eventsMixin(Vue)	// 在原型上定义$on、$once、$emit等事件方法
   lifecycleMixin(Vue)	// 在原型上定义_update、$forceUpdate、$destroy等生命周期方法
   renderMixin(Vue)	// 定义_render方法，返回虚拟DOM
   ```

2. 到了正式阶段，先创建一个Vue实例，执行_init方法，在其中，主要做的事情按照顺序有：执行beforeCreate钩子、`initState(vm)`（包括data、props、methods、watch等的初始化）、执行created钩子，最后执行`vm.$mount()`。

3. 在$mount中，会根据template生成render函数，render`的作用主要是生成`vnode，$mount有一个需要注意的点，$mount会被执行两次，第一次生成render，第二次执行mountComponent。

4. 调用mountComponent渲染组件，也就是执行render，生成vnode。

5. _update主要功能是patch，将vnode转换为真实的DOM，并更新到页面中。

### Vue生命周期

#### 是什么

Vue实例从创建到销毁的过程就是生命周期。生命周期中有很多钩子，当任务流转到某个地方，就会执行对应的钩子。钩子会自动绑定this（可以在钩子函数中使用this访问到Vue实例的各个属性），不能用箭头函数定义生命周期方法。

#### 有哪些

beforeCreate、created、beforeMount、mounted、beforeUpdate、updated、beforeUnmount、unmounted、errorCaptured等

#### 整体流程

<img src="/Users/erfan/Desktop/111.png" alt="111" style="zoom: 50%;" />

### 把数据请求放在created和mounted中有什么区别

created在组件实例创建完成立刻执行，此时DOM节点为生成，mounted是在DOM节点渲染完毕后立刻执行，created比mounted早，但都能拿到实例的属性和方法，最好放在created中。

### KeepAlive

当在多个组件之间切换时，缓存被移除组件的状态。当一个组件被KeepAlive缓存时，就多了actived和deactived钩子，并不会被卸载。

注：actived钩子在组件挂载时也会被调用，并且deactivated钩子在组件卸载时也会被调用。两个钩子不仅适用`<KeepAlive>`的根组件，也适用于缓存树中的后代组件。

用法：

```html
<Keep-alive :include="['Home']">
	<component :is="activeComp"/>
</Keep-alive>
```

### 为什么Vue的data需要是一个函数

这个问题直接看源码，因为创建Vue用到的构造函数是一样的，初始化时，在`Vue.prototype._init()`方法中，需要`initState(vm)`，这个方法中有一步`initData(vm)`，专门用来初始化data，再细看`initData`，会先判断传进来的data是对象还是函数，如果是函数，就直接执行，拿到返回值赋给`vm._data`以及data，然后监听data。

```typescript
function initData(vm: Component) {
  let data: any = vm.$options.data;
  // 
  data = vm._data = isFunction(data) ? getData(data, vm) : data || {};
  // ...
  const ob = observe(data)；
  ob && ob.vmCount++
}

function getData(data: Function, vm: Component): any {
  //...
  try {
    return data.call(vm, vm);
  }
  // ...
}
```

从源码可以看出，如果data传的是对象，那么多次创建的Vue实例拿到的data就都是这个对象，组件被多次复用，只要该对象被修改，将会影响到该组件的其它实例。为了不冲突，于是传入函数，这样每个Vue实例拿到的对象都是不同的。

### Vue为对象添加一个新属性为什么不能刷新页面

Vue2中，对象的每个属性都是用`Object.defineProperty()`定义过get和set的，直接为对象添加新属性，根本没有触发任何get、set，响应式系统就不会感知到数据的变化，页面也就不会刷新。

解决办法就是用`Vue.set()`，找个公用方法可以让新加的属性变成响应式的；或者用`Object.assign()`，直接改变整个对象；也可以用`Vue.forceUpdate()`强制刷新，但一般不要这么做。

### v-model有什么用

它是一个双向绑定指令，通常需要将输入框的内容同步给js变量，同时也要让js变量的变化反映到输入框中。手动来操作比较麻烦。

同时也可以把v-model用于自定义组件，达到父子组件通信的目的。

对于`<input>`和`<textarea>`，v-mode会绑定value，监听input事件，对于其它元素，再查

```html
// 普通的写法
<input :value="text" @input="event => text = event.target.value"/>
```

```html
// v-model写法
<input v-model="text">
```

由于html内置的表单不总能满足需求，我们通常会把input封装成新的表单组件，于是`v-model`就会用在组件上了。

```html
// 组件上的v-model
<CustomInput v-model="text"/>

// 会被转换成
<CustomInput :modelValue="text" @update:modelValue="newVal => text = newVal" />
```

```html
// 组件内部的写法
<input :value="modelValue" @input="$emit('update:modelValue', $event.target.value)">
```

```javascript
// 组件内部的另一种写法（用v-model="value"）
computed: {
	value: {
    get() {
      return this.modelValue;
    },
    set(newVal) {
      this.$emit('update:modelValue', value);
    }
  }
}
```

```html
// 不想用modelValue，可以自己定义，那么在组件内部就可以把modelValue都换成title
<CustomInput v-model:title="text">
```

在自定义组件上使用v-model也可以用修饰符，用modelModifiers就可以拿到修饰符。

```html
<CustomInput v-model:title.capitalize="text"/>
```

```javascript
export default {
  props: {
    modelValue: String,
    modelModifiers: {
      default: () => ({})
    }
  },
  emits: ['update:modelValue'],
  created() {
    console.log(this.modelModifiers) // { capitalize: true }，
  }
}
```

### 为什么要用Proxy替代Object.defineProperty

1. 如果Vue的一个属性是数组，通过数组下标添加元素，无法响应；
2. 如果Vue的一个属性是对象，也无法响应该对象的属性，除非监听其每一个属性

可以看到`Object.defineProperty`无法感知到对象中某个属性的变化，这就是Vue2的缺陷，修改对象中的深层属性或者改变数组元素或长度时，无法响应，但Vue3用了Proxy，可以实现深层监听。

```javascript
// 验证Object.defineProperty的缺陷
const o = {
  _birth: {
    year: 90,
    month: 2,
    day: 3,
  },
};
Object.defineProperty(o, "birth", {
  get() {
    return this._birth;
  },
  set(newVal) {
    console.log("setter");
    this._birth = newVal;
  },
});

console.log(o);
o.birth.year++;

console.log(o);
```

### this.$set()的用处

由于`Object.defineProperty`的缺陷，想要为Vue实例动态增加新属性时，新增的属性将不会是响应式的，因为Vue2无法探测普通新增属性，不过确实可以成功新增，但无法是响应式的。

```javascript
// Vue无法感知到newProperty，该属性不是响应式的
this.myObject.newProperty = 'hi';
```

`this.$set`就专门为了解决这个问题，当想新增响应式属性时，就用`this.$set`(target, key, value)。

注：Vue3的文档里没有发现$set方法，应该是Vue3不需要了，因为有了Proxy，新增属性就是响应式的。

### 渲染机制

主要讨论Vue如何将一份模版转换为真实的DOM节点，又如何高效更新这些节点。

#### 虚拟DOM

即vnode是纯js对象，对应真实的DOM，放在内存中，便于开发者操作，把具体的DOM操作交给渲染器去处理。

#### 渲染pipeline

<img src="/Users/erfan/Downloads/render-pipeline.03805016.png" alt="render-pipeline.03805016" style="zoom:50%;" />

1. 编译，Vue模板被编译为渲染函数，该函数返回虚拟DOM树，该步骤可在构建时完成，也可以使用运行时编译器完成；
2. 挂载，运行时渲染器调用渲染函数，返回虚拟DOM树，并创建实际的DOM节点；
3. 更新，当依赖变化后，会创建一个更新后的虚拟DOM树，运行时渲染器将其与旧树对比，将必要的更新应用到真实DOM。

### 为什么需要DOM？

真实的DOM操作开销较大，而用虚拟DOM，对真实DOM进行抽象，可以减少操作的开销。

### 实现虚拟DOM的思路？

每个vnode都有像tag这样的属性，同时也有children属性，保存着子vnode，这样就形成一棵虚拟树结构。

### Vue中的数据频繁变化为什么只会更新一次

频繁变化是这个意思：

```javascript
data() {
  return {
    isOpen: true;
  }
},
mounted() {
	// 第一次变化
  this.isOpen = false;
  // 第二次变化
  this.isOpen = true;
}
```

```html
<template>
	<div>
    {{ isOpen ? 'open' : 'close' }}
  </div>
</template>
```

根据Vue双向绑定的原理，按发布订阅模式，isOpen将被设置数据拦截，模板template中依赖了isOpen，就会创建watcher（更新视图），订阅isOpen的变化，当isOpen变化时，就会将对应的这个watcher放入队列中，等到下一个事件循环再执行（更新视图）。由于watcher都是有唯一id的，如果是频繁变化，队列中不允许有重复的watcher id，所以本事件循环无论isOpen变化多少次，队列中只会有最新的watcher.update方法。

到了下一个事件循环，这些watcher.update将会执行，创建一个新的虚拟DOM树。

### this.$nextTick的作用及原理

#### 作用

由于Vue渲染DOM是异步的，所以状态改变后，并不会立刻更新DOM，如果想要访问更新后的DOM，则需要使用nextTick。

#### 原理

只要Vue状态变化（就会执行该状态的副作用函数，副作用函数和watcher是两回事），Vue就将开启一个队列，缓冲本事件循环中所有变化数据的watcher（用来更新DOM的函数，watcher会取最新的状态来更新DOM），并保证队列中同样的watcher只有一个；nextTick 方法会确保回调函数在下一个事件循环开始的时候，执行完队列中的全部watcher（这会更新视图），最后再执行nextTick的回调函数。

nextTick是微任务，根据执行环境分别尝试采用 Promise、MutationObserver、setImmediate，如果以上都不行则采用 setTimeout。

为什么是微任务呢？我估计是和then一样，当使用Promise实现的时候，当所有watcher执行完毕了，才会resolve，才会把nextTick的回调函数加入微任务队列，而这些微任务都会在本轮事件循环内执行完，这就实现了next tick，回调是在下一个事件循环执行的，而在这所谓的下一个事件循环内，刚好把上一个事件循环留下的patch工作都做了。

### 如何理解mixin，应用场景是什么？

mixin混入，在vue组件中使用mixin选项，可以把一些公用的方法或者属性抽离出来，然后混入到有需要的组件中，可以简化代码。

如果混入的属性或方法与组件中的冲突了，则data会覆盖，钩子函数会组成数组，都调用。

有全局混入和局部混入

### slot的应用场景

当一个组件中有一部分内容是需要差异化，就可以用插槽，定制它，而不用再写一个组件。

有默认插槽、具名插槽、作用域插槽

### `Vue.Observable`是什么？应用场景？

调用这个方法可以让对象编程响应式的

有时候业务很简单，使用vuex又比较复杂，可以使用这个方法，定义响应式的属性，然后直接在vue组件中引入，并在computed中使用

### 什么是透传attributes

给一个组件传属性或事件监听器，如果组件内部没有消费掉，它们就会被绑在组件模版的根元素上，除非用props和emits消费掉，就不会绑在根元素上。

多根节点无自动透传能力，需要使用`v-bind="$attrs"`，显示指定透传给哪一个根节点。

在子组件中，可以使用`this.$attrs`来访问所有透传的attribute。

```html
<MyButton @click="onClick" :title="hello"></MyButton>

// 渲染之后的MyButton模板
<div @click="onClick" title="hello"></div>
```

### 子组件可以直接改变父组件的值吗

不可以，Vue是单向数据流，不能直接改变传下来的props，只能用自定义事件通知父组件去更新。

注：如果有自定义事件，推荐是要使用emits将它消费掉，不然可能因为有隐藏的透传，造成两次触发该事件

### 平行组件的传值

可以利用公共的父组件，或者事件总线Event Bus（Vue2），Vue3不再支持event bus，需要使用外部库vue3-eventbus。

发的时候用emit，收的时候用on

### 命令式与声明式的差异

命令式框架关注过程，声明式框架关注结果。

**声明式代码的性能不优于命令式代码**，因为声明式代码只关注结果，更新时，修改操作交给Vue去做了，声明式代码会多出查找差异的性能损耗。

```javascript
// jQuery（命令式）
$('#app')
	.text('hello world')
	.on('click', () => { alert('ok') })

// Vue（声明式）
<div @click="() => alert('ok')">hello world</div>
```

最理想的情况是找出差异的性能损耗为0，因此说声明式代码的性能不优于命令式代码。

不过，声明式代码的可维护性比命令式代码更好，所以Vue框架要做的就是在保证可维护性的同时，让性能损耗更小。

声明式代码的更新性能消耗 = 找出差异的性能消耗 + 直接修改的性能消耗，而虚拟DOM的出现就是为了无限降低找出差异的性能消耗

### innerHTML、虚拟DOM、原生JS实现的框架性能

三者的性能不能简单地下定义，这要根据情况来判断。

1. 原生js操作DOM性能最高，但心智负担大，通俗说就是代码太繁琐不易维护；
2. 虚拟DOM比原生js多出查找差异的性能损耗以及虚拟DOM内存消耗，但心智负担低，易维护；
3. innerHTML只要有更新，就会删掉旧DOM，重建新DOM，如果页面很大，将造成非常大的性能开销。

因此，Vue采用了虚拟DOM，这是权衡的结果。

### 编译时、运行时、编译运行时框架

Vue是编译时+运行时框架。也就是程序员既可以手写render函数，也可以写template让编译器先去编译，从而分析后进一步提升更新性能。

1. 编译时框架就是程序员写的代码都需要先编译，才能运行，即只写一套html模板字符串，编译器将其编译成原生hs代码；
2. 运行时即让程序员去写render函数和虚拟DOM结构，可直接运行，无需编译；
3. 编译运行时，既支持程序员写虚拟DOM数据对象，直接运行，也支持写html模板字符串，编译后直接运行。

### Vue的设计思路

#### 声明式地描述UI

使用与html标签一致的方式来描述DOM元素，使用与html一样的方式描述属性，使用v-bind描述动态绑定的属性，使用v-on描述事件，使用与html一致的层级结构。哪怕是事件，也不需要写命令式的代码，而是使用声明的方式实现，这就是声明式地描述UI。

除了上面这种模板的方式来声明式地描述UI以外，还可以使用js对象来描述，其实就是虚拟DOM，它大概长这样：

```javascript
const title = {
  tag: 'h1',
  props: {
    onClick: () => console.log('hello');
  },
  children: 'hello world'
};
```

渲染函数长这样：

```javascript
export default {
  render() {
    // return h('h1', { onClick: '' }, 'hello world');
    return {
      tag: 'h1',
      props: {
        onClick: () => console.log('hello');
      },
      children: 'hello world'
    }
  }
}
```

#### 渲染器

渲染器可以将虚拟DOM转换成真实DOM，下面是一个简易的渲染器

```javascript
const vnode = {
  tag: "div",
  props: {
    onClick: () => console.log("hello world"),
  },
  children: "Hello World",
};

function renderer(vnode, container) {
  const el = document.createElement(vnode.tag);
  for (const key in vnode) {
    if (/^on/.test(key)) {
      el.addEventListener(
        key.substring(2).toLowerCase(),
        vnode.props[key]
      );
    }
  }
  if (typeof vnode.children === "string") {
    el.appendChild(document.createTextNode(vnode.children));
  } else if (Array.isArray(vnode.children)) {
    vnode.children.forEach((child) => {
      renderer(child, el);
    });
  }
  container.appendChild(el);
}

renderer(vnode, document.querySelector("#app"));
```

#### 组件

虚拟DOM不仅能描述真实DOM，还能描述组件，组件其实就是一组DOM元素的封装，大概长这样：

```javascript
const MyComponent = function () {
  return {
    tag: 'div',
    props: {
      onClick: () => alert('hello');
    },
    children: 'hello world'
  }
}

// 怎么用虚拟DOM描述组件
const vnode = {
  tag: MyComponent
};
```

只需要把前面的渲染器改造一下，让它可以处理tag是函数的情况，并且递归地去生成真实的DOM。组件当然不仅可以是函数，还可以是对象，只需要在对象里定义render函数，render返回虚拟DOM即可。

#### 模板的工作原理

这就需要编译器的参与了，对编译器来说，模板就是一个普通的字符串，编译器会分析template里的内容，生成一个渲染函数，添加到script的组件对象上，因

### 为什么mutations必须是同步的

mutation中不能有异步操作，就像下面这样，因为devtool会把每一条mutation都记录下来，devtool会捕捉前一状态和后一状态的快照，如果mutation中存在异步回调，就无法判断回调何时执行，devtool也就无法追踪到mutation执行前和执行后的状态了，因为这两个状态可能一样，造成调试困难。

```javascript
mutations: {
  increment(state) {
    api.getName(() => {
      state.count++;
    })
  }
}
```

总结起来就是，若某个mutation执行完毕，那该mutation中引起的状态变更也要在这一时刻变更完成。

### 怎么解决刷新页面时，vuex中的数据丢失的问题

1. 可以使用localStorage和sessionStorage，为window注册beforunload事件，在关闭页面或离开页面之前，会触发该事件，此时可以把数据都放在本地保存起来，再次打开页面的时候从本地取出即可。

```javascript
window.onbeforeunload = function () {
  // 把vuex中的数据存储到本地
}
```

2. 使用vuex-persistedstate插件

### 已经有了cookie，为什么还要使用localStorage和sessionStorage

1. 与cookie不同，web存储对象不会随着每个请求发送到服务器，因此可以保存更多数据，至少5MB；
2. 服务器无法通过http header操作存储对象，web存储对象只能由js来操作；
3. web存储对象绑定到源，即不同源的页面（协议/域/端口相同）无法访问彼此的web存储对象







## Vue-Interview

### v-if和v-for的优先级

v-if比v-for的优先级更高，v-if会被先执行，如果在同一个元素上同时使用了它俩，v-if将不能使用v-for中的变量，导致出错。

不推荐在一个元素上同时使用v-if和v-for

```html
// 需要这样改写
<template v-for="todo in todos">
    <li v-if="todo.isComplete">
      {{ todo.title }}
    </li>
</template>
```

### key的作用

当我们讨论diff的时候，讨论的对象有三，旧vnode、新vnode、真实渲染的DOM。

patch是指对真实DOM进行操作。

Vue渲染列表时，更新DOM的大概过程：列表改变 -> 生成新的vnode（re-render） -> 对比新旧vnode的key和type，如果新vnode的key在旧vnode中能找到且type相同，则patch该节点（对真实DOM进行补丁），如果找不到key，那就要创建新的真实DOM -> 按最新的顺序移动真实DOM。

这个过程就是要利用DOM复用，但DOM复用并不代表DOM元素就不需要更新，patch就是在更新DOM元素，因为key值相同，也不一定保证DOM元素就不会变了。

回答：key的作用就是为了最大程度地复用DOM元素，减少DOM操作，在渲染列表时，如果设置了key，Vue就会用新vnode的key在旧vnode中比对，看是否有可复用的DOM，如果找到了相同的key，就直接patch去更新真实DOM，不需要删掉旧DOM元素，创建新DOM元素了（主要就是这个删除和新建很费性能）。如果不设置key，那么Vue会就地更新，也就是逐一比对，此时key为undefined，找不到能复用的DOM，会认为两个vnode相同，从而直接patch，每一个元素都patch一次，造成浪费。

### 如果将数组索引用作key，会有什么问题

1. 违背虚拟DOM的初衷，该复用的没复用，实际还是在原地更新；

   ```javascript
   // 旧虚拟dom，用索引作为key
   [
   	{
       key: 0
       text: 'A',
     },
     {
       key: 1,
       text: 'B'
     },
     {
       key: 2,
       text: 'C'
     }
   ]
   
   // 新虚拟DOM
   [
     {
       key: 0,
       text: 'C'
     },
     {
       key: 1,
       text: 'B'
     },
     {
       key: 2,
       text: 'A'
     }
   ]
   ```

2. 当删除列表第一个元素时，实际会删除最后一个DOM。由于key为0和1的虚拟dom都还在，所以只会删除key为2的真实DOM，更新key为0和1的真实DOM。

   ```javascript
   // 旧虚拟dom，用索引作为key
   [
   	{
       key: 0
       text: 'A'
     },
     {
       key: 1,
       text: 'B'
     },
     {
       key: 2,
       text: 'C'
     }
   ]
   
   // 新虚拟DOM
   [
     {
       key: 0,
       text: 'B'
     },
     {
       key: 1,
       text: 'C'
     }
   ]
   ```

### 修饰符是什么，有哪些？

修饰符处理了许多DOM的细节，让开发者集中于业务逻辑。

常见的修饰符有：

表单修饰符

1. lazy（光标离开后，才把表单的值赋给value）
2. trim（去掉表单内容两边的空格）
3. number（将用户输入的值转换成number类型）

事件修饰符

1. stop（禁止事件冒泡）
2. prevent（禁掉事件的默认行为）

还有鼠标、键盘修饰符以及v-bind修饰符（就是在绑定的值后面加修饰符 :text.sync="text"，这在vue2能实现一个双向绑定，和v-model一样）

### 自定义指令是什么，有用过吗？

v-show、v-model都是Vue已有的指令，也可以自己写指令，叫作自定义指令。

可全局或局部声明自定义指令，全局：

```javascript
Vue.directive('focus', {
  // 当被绑定的元素插入到 DOM 中时……
  inserted: function (el) {
    // 聚焦元素
    el.focus()
  }
})
```

局部声明：

```javascript
directives: {
  focus: {
    // 指令的定义
    inserted: function (el) {
      el.focus()
    }
  }
}
```

我应用过的场景是图片加载失败后的兜底图：

```javascript
Vue.directive('err-img', {
  inserted(el) {
    el.onerror = function () {
      el.src = '兜底图地址'
    }
  }
})
```

inserted表示这个函数在该元素插入到DOM中时调用，Vue2还有一些其他的钩子比如bind（指令第一次绑定到元素上时）、updated（vnode更新时调用），Vue3就改成created、beforeMount、mounted、beforeUpdate、updated等和Vue声明周期一样，用这些钩子更清晰些。

钩子函数的参数有el（代表绑定了自定义指令的元素）、binding、vnode、preNode，

### 双向绑定及其原理

双向绑定：更新数据，视图也会随之更新，而视图变化，数据也会更新。

实现方式是[数据绑定](./前端算法.md#观察-订阅模式，数据绑定)+事件回调，对应的语法糖v-model，这样可以省掉大量的处理代码，可读性更好。

通常v-model会用在表单上，也可以用在组件上，当用在组件上的时候，需要在组件内部处理modelValue和update:modelValue。v-model实际上会被转换成:value和@input（表单上）或:modelValue和@update:modelValue（组件上）。

#### 相关问题1：想要改变v-model的属性名和事件名

使用v-model:varName和@update:varName即可。

#### 相关问题2：v-model和.sync修饰符的区别（Vue2）

Vue3中已经没有.sync修饰符了，Vue3中v-model也可以绑定多个值，可以用于组件上，而不需要加model选项，并且用event:value的方式定义事件。

### diff算法

定义：diff算法就是找出新旧虚拟DOM的差异，然后根据差异精准更新真实DOM（patch）；

必要性：使用虚拟DOM+diff算法，可以减少DOM操作，提升更新性能；

执行时机：当Vue实例的状态改变后，re-render得到新的虚拟DOM，此时即可执行patch函数，传入新旧虚拟DOM，找到差异，并更新到真实DOM；

执行过程：patch函数遵循深度优先，同层比较的策略

1. 判断新旧虚拟DOM节点是否是同类型节点（type），不同则删除旧的真实DOM并重建；
2. 如果是文本节点，则更新文本内容（patch补丁）；
3. 如果新旧虚拟DOM是同类型节点，则递归更新其子节点（patch补丁）；

vue3引入的patch优化：patchFlags（在compile阶段，将使用了动态绑定的地方进行标记，这样在runtime阶段使用patch的时候，可快速找出要patch的地方，靶向更新，性能更高）、block。

### Vue3的8种组件方式

1. props，父子组件传值
2. emit，子组件触发父组件的方法
3. provide/inject，父组件和其所有子组件传值或传函数
4. expose/ref，父组件访问子组件内的值或方法
5. v-model，父子组件传值，双向绑定
6. attrs，未声明props和emits时，实现父组件透传属性到子组件
7. vuex
8. mitt

### 对vuex的理解

1. vuex定义：vuex是vue专用的状态管理库，它集中管理应用的状态，并且可以保证状态变更的可预测性；
2. vuex解决的问题（核心理念）：vuex解决了多组件之间的状态共享问题，用各种组件通信方式来保证组件之间状态统一会导致应用难维护，所以vuex将所有组件的状态都抽取出来，用统一的方式获取状态或修改状态，vuex也保证了数据单向流动；
3. 何时使用vuex：当全局状态非常复杂时，可以使用vuex；
4. 使用：状态都放在state中，对state的修改都只能使用mutation方法，在组件中用commit提交mutation即可，如果状态修改过程涉及到异步操作或复杂逻辑，需要使用action，在组件中使用dispatch派发action。还可以利用vuex的模块化modules选项，以及命名空间选项namespace，将状态分类，使模块具有更高的封装性。
5. 响应性：vuex借用vue将state作响应性处理，使得状态变化时，触发组件重渲染。

### 封装过axios吗？主要是封装哪些方面？

axios的API很友好，完全可以在项目中直接使用，但重复地写axios请求代码不仅麻烦，而且还难以维护。

封装主要是针对以下一些方面：

1. 区分开发、测试和生产环境，利用process.env.NODE_ENV，不同的环境用不同的域名，还可进一步利用webpack proxy解决开发模式下的跨域问题；
2. 规范请求头和超时时间，或者用请求拦截统一把token放到请求中，如果确实有特殊要求，可以留个接口来覆盖默认行为；
3. 把axios各种请求方法写好之后，再用一个函数封装起来，放在api文件里，要发请求就从该文件里引，便于统一管理api接口；
4. 请求和响应的统一拦截，可以做一些统一的错误处理等。





