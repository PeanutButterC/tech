### 如何解决 React 的重渲染（频率：1）

React 默认规则是父组件重渲染，则其所有子组件都会重新渲染， 不论子组件的 props 等传值是否改变，因此就会造成大量的重复渲染。

为了让无关的子组件不要重新渲染，可以用 memo 把子组件包起来，这样的话只有当子组件收到的 props 改变后，才会重新渲染子组件。

但仅仅用 memo 还不一定生效，需要用 useCallback 和 useMemo 在父组件中把值和函数都包起来，进行缓存。

### useMemo 和 useCallback 的区别

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-04-21 16.11.14.png" alt="截屏2023-04-21 16.11.14" style="zoom: 50%;" />

useCallback 示例，点击 button，App 会重新执行，但 Child 就不会重新执行，因为 myCount 和 changeCount 都没有变。

```jsx
const App = () => {
  console.log("re-render-App");
  const [count, setCount] = useState(0);

  const changeCount = useCallback(function () {
    console.log("hello");
  }, []);

  const myCount = useMemo(function () {
    return count * 2;
  }, []);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <Child changeCount={changeCount} myCount={myCount} />
    </div>
  );
};

export default memo(function Child({ myCount, changeCount }) {
  console.log("re-render-Child");

  return <div onClick={changeCount}>{myCount}</div>;
});
```

useMemo 示例：

```jsx
// 使用了 useMemo, 只有 total 改变，才会重新计算
const totalToStringByMemo = useMemo(() => {
  return total + "";
}, [total]);

return (
  <div className="App">
    <h3>countToString: {countToString}</h3>
    <h3>countToString: {totalToStringByMemo}</h3>
  </div>
);
```

### useCallback 和 useMemo 的第二个参数是什么

第二个参数是一个数组，里面是依赖项，也就是说，只有当这些依赖项变化之后，才会返新的函数或新的值，否则是会取缓存的。

### React Native 容器和浏览器的区别

RN 在执行 js 方面和浏览器区别不大，都是单线程运行 js，不同的是 RN 容器可以直接写 ES6，不用考虑兼容问题。

### React Hooks 的优势

- 淡化生命周期，以前相同功能会被分拆到不同的生命周期中，难以维护，现在通过 hooks 被聚合在一起了，比如`useEffect`利用 setup function 和 cleanup function 把某个状态的副作用都聚在一起；

  完全不相关的代码可能被混在同一个生命周期中，使用 useEffect 可以将它们拆开，比如下面代码，断开连接就不需要放在`componentWillUnmount`中了，放在 cleanup function 就可；

  ```javascript
  import { useEffect } from "react";
  import { createConnection } from "./chat.js";

  function ChatRoom({ roomId }) {
    const [serverUrl, setServerUrl] = useState("https://localhost:1234");

    useEffect(() => {
      const connection = createConnection(serverUrl, roomId);
      connection.connect();
      return () => {
        connection.disconnect();
      };
    }, [serverUrl, roomId]);
    // ...
  }
  ```

- 状态管理更加清晰，将状态与状态变更的逻辑陪对，拥有比以前更好的代码结构；

- 不用再考虑 this 的指向问题

### useEffect 的工作方法

`useEffect(setup, dependencies?)`，setup 可选地返回一个 cleanup 函数，挂载的时候会执行一次`useEffect`，此后依赖的状态变化会用旧状态执行一次 cleanup，再用新状态执行 setup，最后在组件卸载的时候，执行一次 cleanup

### React class 组件处理错误的 API

1. ErrorBoundary 组件，可以捕获渲染期间、生命周期、构造函数中的错误，并以兜底 UI 替换
2. 事件处理过程一般不会被 ErrorBoundary 捕获到，可以改用 try...catch
3. window.onerror 处理函数

### React 类组件和函数组件有什么区别？

1. 核心不同：如果 props 变化，类组件的 this.props 会受到影响，而函数组件由于是把函数重新执行了一遍，是独立的，不存在这个问题，具体例子看[这里](https://overreacted.io/zh-hans/how-are-function-components-different-from-classes/)。这就是因为类组件它实际上只创建了一个实例，this 指向这个实例，如果父组件传的 props 变了，那么用到 this.props 的地方拿到的就是新值。
2. 类组件有生命周期；
3. 类组件创建时需要继承 React.Component
4. 类组件有自己的状态 state，函数组件可以用 useState 钩子

### React 组件 state 批处理是什么？

一般更新 state 的操作是放在事件处理函数中的，只有等到事件处理函数的全部代码都执行后，React 才会对组件进行 re-render。批处理就是即使调用了`setState`，也不要马上更新 state 并且 re-render，而是将更新操作放到该状态对应的队列，等到事件处理函数全部执行后，再 re-render，在 rerender 期间会调 useState，此时 React 内部会根据当前状态和队列计算出本次渲染该状态的值。

就像这样：

```react
function App() {
  const [count, setCount] = useState(0);
  const [number, setNumber] = useState(0);

  return (
    <span>{count}</span>
    <span>{number}</span>
    <button onClick={() => {
        setCount(count + 1);	// 将“count值更新为1”排入count的队列
        setCount((c) => count + 2)	// 将“将上一个count值+2”这个操作放入count的队列
        setNumber(20);				// 将“number值更新为20”排入number的队列
        setNumber((number) => number + 1);	// 将“上一个number值+2”排入number队列
        // 事件处理函数执行完毕，开始进行re-render，这里是批处理，因为是等到上面的操作都完成后，才re-
        // render的
        // re-render时，执行到useState，获取count和number，都是执行刚刚的队列里的更新操作，拿到最后的值
        // 并返回
      }}>btn</button>
  )

}
```

### 如果事件处理函数是异步的，什么时候触发重渲染？

事件处理函数会被拆成两部分，同步部分会批处理一次，触发一次 rerender，异步回调部分是另外一次批处理，再触发一次 rerender。

### 练习：修复请求计数器

关键点就是不能用`setPending(pending + 1)`，`setPending(pending - 1)`，这个 pending 值实际是一样的。

```javascript
import { useState } from "react";

export default function RequestTracker() {
  const [pending, setPending] = useState(0);
  const [completed, setCompleted] = useState(0);

  async function handleClick() {
    setPending(pending + 1);
    await delay(3000);
    setPending(pending - 1);
    setCompleted(completed + 1);
  }

  return (
    <>
      <h3>等待：{pending}</h3>
      <h3>完成：{completed}</h3>
      <button onClick={handleClick}>购买</button>
    </>
  );
}

function delay(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}
```

### 练习：实现状态队列

在一个事件处理函数中，对同一个 state 的 set 被放在一个队列中，下一次 rerender 时，根据这个队列计算出最终的 state 值。在 React 的 useState 中，实际就是像下面这样，根据初始状态和一个队列，计算出最终的值。

```javascript
function getFinalState(baseState, queue) {
  // TODO: 对队列做些什么...
  return queue.reduce((pre, cur) => {
    if (typeof cur === "number") {
      return cur;
    } else if (typeof cur === "function") {
      return cur(pre);
    }
  }, baseState);
}
```

### 如果想要在下次 render 前多次更新同一个 state，要怎么做？

这是一种不常见 case，如果使用`setCount(count + 1)`，这是在告诉 React 下一个值是 count + 1，而如果多次使用`setCount(count + 1)`，并不会让 count 多次累加，想要实现多次累加，需要为`setCount`传一个函数，其中函数的参数是上一个 state 的值：

### 如何在本次事件处理程序中访问到更新后的 state（类似 vue 的 nextTick）

因为每一个渲染都有自己的事件处理函数，且每一次渲染都有自己的快照，那么一般就没有办法在本事件处理函数中访问到更新之后的 state，但可以利用给 setState 传的更新函数，在更新函数中拿到更新后的值。

```javascript
const handleClick = () => {
  setCount((count) => {
    const res = count + 1;
    console.log(res); // 拿到最新的值，有点像vue中的nextTick
    return res;
  });
};
```

### 用 useRef 保存列表的 DOM 引用

ulRef 是数组，用于保存多个`<li>`的 dom 引用，最重要的问题是如何保证组件多次 rerender 或者`<li>`数量变化后，ulRef 能准确保存着多个`<li>`的最新 dom 引用。

```javascript
import { useState, useRef, forwardRef } from "react";

export default function App() {
  const [count, setCount] = useState(0);
  const [todos, setTodos] = useState([
    { id: 1, text: "java" },
    { id: 2, text: "c++" },
    { id: 3, text: "python" },
  ]);
  const ulRef = useRef(null);
  if (ulRef.current === null) {
    ulRef.current = [];
  }

  const items = todos.map((e, idx) => {
    return (
      <Item
        idx={idx}
        text={e.text}
        key={e.id}
        ref={ulRef}
        onClick={() => handleClick(e.id)}
      />
    );
  });

  const handleClick = (id) => {
    console.log(ulRef.current[id - 1].innerHTML);
  };

  console.log(ulRef.current);

  return (
    <>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>increment</button>
      <ul>{items}</ul>
    </>
  );
}

const Item = forwardRef(function ({ text, onClick, idx }, refs) {
  return (
    <>
      <li
        className="item"
        ref={(r) => {
          refs.current[idx] = r; // 关键点
        }}
      >
        {text}
      </li>
      <button onClick={onClick}>btn</button>
    </>
  );
});
```
