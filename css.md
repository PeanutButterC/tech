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

### 4. transform、transition和animation的区别（待详细化）

transform是将元素位移、旋转等，transition是定义动画

transition一般需要一个手动触发的方式，animation定义好后可直接播放

### 5. window.requestAnimationFrame

向浏览器请求一帧动画，浏览器会保证你传入的回调函数在浏览器下一次重刷新之前调用。这样的话就能保证浏览器每刷一屏，动画都会有最新的进展，浏览器每秒刷60次，回调也执行60次。**不会掉帧。**
