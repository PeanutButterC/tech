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

深层的对象、数组，都将是响应式的。

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