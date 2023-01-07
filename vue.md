### DOM的更新不是同步的

当更新了响应式状态后，并不会立刻更新DOM，Vue将会缓冲到更新周期的下个时机，确保无论进行了多少次状态更改，每个组件都只更新一次。如果要获取状态更新后的DOM，用nextTick

### Vue3中的响应式对象都是Proxy实现的

下面this.obj是一个Proxy，并不是属性值本身

```javascript
data() {
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

我们知道Vue有一个概念叫模版template，我曾一直困惑，Vue状态改变，updated钩子什么时候执行，是在页面刷新之后？还是之前？

触发setter(状态更新) -> 通知watcher -> 触发re-render -> 生成new vnode(vdom) -> patch（更新真实DOM）-> 触发updated钩子 -> 触发nexTick回调（微任务）

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-01-02 00.38.06.png" alt="截屏2023-01-02 00.38.06" style="zoom: 33%;" />

显然，updated会在DOM更新后被执行一次。我的理解re-render就是刷新模版template（生成新的vdom）。

### watch及附带的async函数知识点、对嵌套状态的监听

我们通常需要在状态变化后，执行一些副作用，比如更改DOM，因为状态变化后，通知watcher -> 触发re-render。

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

因为一般情况会有：响应式状态变更 -> 通知watcher -> 生成新vnode -> dispatch

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

#### 







