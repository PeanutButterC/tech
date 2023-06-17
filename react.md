## 如何解决React的重渲染（频率：1）

React默认规则是父组件重渲染，则其所有子组件都会重新渲染， 不论子组件的props等传值是否改变，因此就会造成大量的重复渲染。

为了让无关的子组件不要重新渲染，可以用memo把子组件包起来，这样的话只有当子组件收到的props改变后，才会重新渲染子组件。

但仅仅用memo还不一定生效，需要用useCallback和useMemo在父组件中把值和函数都包起来，进行缓存。

## useMemo和useCallback的区别

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-04-21 16.11.14.png" alt="截屏2023-04-21 16.11.14" style="zoom: 50%;" />

useCallback示例，点击button，App会重新执行，但Child就不会重新执行，因为myCount和changeCount都没有变。

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

useMemo示例：

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

## useCallback和useMemo的第二个参数是什么

第二个参数是一个数组，里面是依赖项，也就是说，只有当这些依赖项变化之后，才会返新的函数或新的值，否则是会取缓存的。

## React Native容器和浏览器的区别

RN在执行js方面和浏览器区别不大，都是单线程运行js，不同的是RN容器可以直接写ES6，不用考虑兼容问题。

## React Hooks的优势

- 淡化生命周期，以前相同功能会被分拆到不同的生命周期中，难以维护，现在通过hooks被聚合在一起了，比如`useEffect`利用setup function和cleanup function把某个状态的副作用都聚在一起；

  完全不相关的代码可能被混在同一个生命周期中，使用useEffect可以将它们拆开，比如下面代码，断开连接就不需要放在`componentWillUnmount`中了，放在cleanup function就可；

  ```javascript
  import { useEffect } from 'react';
  import { createConnection } from './chat.js';
  
  function ChatRoom({ roomId }) {
    const [serverUrl, setServerUrl] = useState('https://localhost:1234');
  
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

- 不用再考虑this的指向问题

## useEffect的工作方法

`useEffect(setup, dependencies?)`，setup可选地返回一个cleanup函数，挂载的时候会执行一次`useEffect`，此后依赖的状态变化会用旧状态执行一次cleanup，再用新状态执行setup，最后在组件卸载的时候，执行一次cleanup

## React class组件处理错误的API

1. ErrorBoundary组件，可以捕获渲染期间、生命周期、构造函数中的错误，并以兜底UI替换
2. 事件处理过程一般不会被ErrorBoundary捕获到，可以改用try...catch
3. window.onerror处理函数

## React类组件和函数组件有什么区别？

1. 核心不同：如果props变化，类组件的this.props会受到影响，而函数组件由于是把函数重新执行了一遍，是独立的，不存在这个问题，具体例子看[这里](https://overreacted.io/zh-hans/how-are-function-components-different-from-classes/)。这就是因为类组件它实际上只创建了一个实例，this指向这个实例，如果父组件传的props变了，那么用到this.props的地方拿到的就是新值。
2. 类组件有生命周期；
3. 类组件创建时需要继承React.Component
4. 类组件有自己的状态state，函数组件可以用useState钩子

