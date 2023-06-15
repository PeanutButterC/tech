### 1. 讲一讲BFC

BFC是块级格式化上下文，是页面上一块独立的渲染区，是块级盒子的布局区域，该区域不受外界影响。

在一些条件下html元素会创建格式化上下文，写几个会创建BFC的例子：

1. `<html>`标签就会创建一个格式化上下文；
2. Flex items (direct children of the element with display flex` or `inline-flex) if they are neither flex nor grid nor table containers themselves.
3. 添加了overflow且值不为clip或visible
4. ...

BFC会有一些特性：

1. BFC里的元素会有margin合并的现象；
2. BFC容器会把里面的float元素包裹；
3. BFC不会被浮动元素遮盖；

还有IFC（行内元素或一段文本会创建）、GFC（网格布局时）、FFC（flex布局时）

### 2. 你认为flex元素是BFC吗？

不是，按照定义，flex元素是FFC，只有flex元素的直接子元素并且子元素也不是flex，才是BFC。

### 3. 如何判断鼠标点击位置处于一个二维区域内（不用坐标）？

给这个二维区域加上点击事件监听器，在冒泡阶段触发处理函数（捕获阶段无法实现），addEventListener的第三个参数useCapture为false即可。即使二维区域html元素是relative或absolute定位也不影响，因为DOM实体就是在偏移后的位置。

### 4. transform、transition和animation的区别

1. transform是将元素位移、旋转、放大缩小等；
2. transition是定义动画，比如`transition: all 0.5s ease-in 1s;`，就是说被元素任意的属性改变都要用这个动画效果，需要一个手动触发的方式。
3. animation定义好后可直接播放，`animation: move-to-right 2s ease-in 2s;`额外需要定义关键帧`@keyframes {}`

### 5. window.requestAnimationFrame

向浏览器请求一帧动画，浏览器会保证你传入的回调函数在浏览器下一次重刷新之前调用。这样的话就能保证浏览器每刷一屏，动画都会有最新的进展，浏览器每秒刷60次，回调如果也能在1s内执行60次，那么就相当丝滑，**不会掉帧。**

### 6. 水平垂直居中

```html
<div class="parent">
  <div class="child"></div>
</div>
```

方案一：flex

```css
.parent {
    height: 500px;
    border: 2px solid red;
    display: flex;
    justify-content: center;
    align-items: center;
}
.child {
    width: 100px;
    height: 100px;
    background: lightblue;
    margin: 20px;
}
```

效果：

<img src="/Users/erfan/Documents/fan/刷题/前端面试题/css/img/1.png" height="300px">

方案二：position

```css
.parent {
  position: relative;
}
.child {
  position: absolute;
  left: 50%;
  top: 50%;
  // 1  
  margin-left: -50px;
  margin-top:-50px;
  // 2 *****
  transform: translate(-50%, -50%)
  // 3
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  margin: auto;
}
```

方案三：grid布局

```css
.parent {
    display: grid;
}
.child {
    justify-self: center;
    align-self: center;
}
```

方案四：【注】在普通的盒子里，即没有flex的盒子里，margin: auto只能将div水平方向上居中

```css
.parent {
    display: flex;
}
.child {
    margin: auto;
}
```

方案五：利用伪元素+vertical-align

```css
.parent {
    text-align: center;		// 将inline-block元素居中
}
.parent::before {
    display: inline-block;
    content: "";
    width: 0;
    height: 100%;
    vertical-align: middle;		// 由于该伪元素的高度是100%，并且vertical-align是middle，所以父元素的基线就是该元素的middle（中线），同一行的元素都按该线为准。
}
.child {
	display: inline-block;
    vertical-align: middle;		// 子元素的中线对准父元素的“基线”
}
```

### css module的原理



### 头部底部固定中间两栏布局
