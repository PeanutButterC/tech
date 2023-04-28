## useMemo和useCallback的区别

<img src="/Users/erfan/Library/Application Support/typora-user-images/截屏2023-04-21 16.11.14.png" alt="截屏2023-04-21 16.11.14" style="zoom: 50%;" />

useCallback示例：

```jsx
// 使用 useCallBack 缓存
const handleCountAddByCallBack = useCallback(() => {
  setCount((count) => count + 1);
}, []);

return (
  <div className="App">
	  <h3>CountAddByChild1: {count}</h3>
		<Child1 addByCallBack={handleCountAddByCallBack} add={handleCountAdd} />
  </div>
);
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



