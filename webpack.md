### --node-env（webpack命令行接口选项）

webpack命令行接口中，使用--node-env选项可以设置process.env.NODE_ENV

```shell
webpack --node-env=development	# process.env.NODE_ENV = 'development'
```

```shell
// 和上面等价
NODE_ENV=development webpack 	# process.env.NODE_ENV = 'development'
```

```shell
// 由于在一些平台，直接设置NODE_ENV=development会报错，所以使用cross-env
npm install cross-env --save
```

```json
"scripts": {
  "dev": "cross-env NODE_ENV=development webpack-dev-server"
}
```

--node-env 传参后，会把mode也自动覆盖，即使webpack.config.js的config中没有mode。

### --mode（webpack命令行接口选项）

webpack打包时命令中的一个选项：

```shell
webpack --mode=development
```

这和从webpack.config.js中导出mode是一样的效果，都会：

1. mode为development时，会将DefinePlugin中的process.env.NODE_ENV设置为development
2. mode为production时，会将DefinePlugin中的process.env.NODE_ENV设置为production
3. mode为none时，不使用任何默认优化选项

注1：如果要根据webpack.config.js中的mode变量更改打包行为，则必须将配置导出为函数，而不是导出为对象

--mode的值会放在argv中。

```javascript
// webpack.config.js
module.exports = (env, argv) => {
	
}
```

### --env（webpack命令行接口选项）

webpack.config.js导出为函数时，函数的参数有env和argv，env可以使用--env来传递，--env可以使用多次，达到传多个env参数的目的。

### process.env.NODE_ENV

process对象是一个全局变量，process.env是一个包含用户环境信息的对象，控制当前Node.js进程，它对Node.js应用程序始终是可用的，无需require。

node环境中，打印process.env，并没有NODE_ENV属性，因为它是需要在package.json的scripts命令中注入的，NODE_ENV并不是node自带的，它由用户定义。

下面执行npm run dev，只能在webpack的构建脚本中可以访问到process.env.NODE_ENV，而被webpack打包的源码是无法访问到的。

```json
{
  scripts: {
    "dev": "NODE_ENV=development webpack --config webpack.dev.config.js"
  }
}
```

### 如何让被webpack打包的源码访问到process.env.NODE_ENV

借助webpack的DefinePlugin插件，创建全局变量，它在需要根据开发模式或生产模式进行不同操作时，十分有用。

```javascript
const webpack = require('webpack');
module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('development')
    })
  ]
}
```

按照上面的代码，在webpack打包生成的bundle时，源代码中的process.env.NODE_ENV常量会全部被替换成development。

```javascript
// 用DefinePlugin定义全局变量后的打包

// 打包前
if (process.env.NODE_ENV === 'development') {
  console.log('development');
} else {
  console.log('production');
}
// 打包后
console.log('development');
```

### 为什么要用JSON.stringify('development')，而不是'development'

至于为什么要使用JSON.stringify('development')或者'"development"'，是因为如果值为字符串，就会被当成代码片段，如果不是字符串就会被当成字符串。

运行打包后的js时，只有process.env.NODE_ENV被替换成了常量'development'，而process.env原本是什么，就仍是什么，此时代码中process.env中是找不到NODE_ENV属性的。

```javascript
// true
console.log(JSON.stringify('dev') === '"dev"');

// false
console.log(JSON.stringify('dev') === 'dev');
```

### Nodejs的`__dirname` 和`__filename`

`__dirname`会返回当前模块文件解析后的目录绝对路径

`__filename`会返回当前模块文件解析后的绝对路径

### .env文件

环境变量：在scripts中传入、并可以在process.env.[NAME]中访问到的变量

适用场景：当需要配置的环境变量太多，全部放在scripts中既不美观也难以维护，此时将环境变量防砸.env文件中，然后使用dotenv插件来加载.env配置文件。

可以根据scripts中的NODE_ENV，来选择使用哪一个.env文件，如.env.dev、.env.prod等。

下面是一个示例，这样就不用把API_URL写在scripts中了，而是在webpack.config.js来根据.env文件获取，只需要在scripts中给定NODE_ENV。

```ini
// env/.env.development
API_URL = http://abc-dev.com

// env/.env.production
API_URL = http://abc-prod.com
```

```javascript
const path = require('path');
const envConfig = require('dotenv').config({
  path: path.resolve(__dirname, './env/.env.' + process.env.NODE_ENV)
});

// { parsed: { API_URL: 'http://abc-dev.com' } }
console.log(envConfig);
```

### .mjs、.cjs、package.json中的type字段

在早期，node仅支持commonjs模块化方案，不过现在已经支持ES Module模块化，既然支持了两套规范，那就要有一个东西来区分，那就是通package.json中的type字段，type字段用于定义package.json文件和该文件所在的目录中.js文件和无扩展名文件的模块化处理方案，默认是commonjs。

```json
{
	"type": "module",
	"scripts": {
  	"start": "node ./src/index.js"
	}
}
```

1. type=commonjs，采用commonjs规范
2. type=module，采用ESM规范，如果源代码有require写法，打包会报错（提示require is not defined in ES module scope, you can use import instead）。npm run start 也会报错，根本上就是type=module时，是看不懂commonjs的。

通常，用node是无法执行es6 js文件的，需要将.js改成.mjs

总结下来，.mjs总是以ES6模块加载，.cjs总是以CommonJS模块加载，而.js文件的加载取决于package.json中的type字段，如果type=module，则按ES6，如果type=commonjs，则按CommonJS加载。

如果type为module，且webpack.config.js是CommonJS规范写的，那么加载webpack.config.js就会报错，因为package.json的type确定了是module，解决办法是改成webpack.config.cjx，并且把打包script里应该加上 --config webpack.config.cjs。

## 对js模块化的理解

#### js没有模块会怎么样？

1. 需要用script标签引入js，但多个script之间有隐式的依赖关系，引入的先后顺序很难把握，当script太多时，很容出现问题。
2. 每个script标签都要发一次请求，过多的请求会拖慢页面加载的速度。
3. 在script声明的顶层变量会污染全局作用域。

#### 有了模块会带来什么好处？

1. 可以清晰地看出来模块之间的依赖关系
2. 可以借助打包工具，将模块**合并成单个js文件**，进而减少网络开销 
3. 模块作用域是隔离的

## webpack等模块打包工具解决了什么问题？

曾经js是没有模块化概念的，后来有了模块化，大都是CommonJS、AMD这种模块化方案，还没有像ES6语言层面的模块化，浏览器没办法看懂这些模块，也没法执行，那就一定需要有一个工具来处理你写的这些模块之间的依赖关系，把不同模块的代码按照特定的规则组织在一起不出问题，并且能让浏览器执行。

#### 既然ES6模块已经得到浏览器的支持，那为什么还需要打包，而不是直接让浏览器执行

1. 无法使用代码分片（首屏必须加载全部代码，）和tree shaking（多余的代码）
2. 目前大多数npm模块还是CommonJS的形式，浏览器并不支持该语法，这些包不能直接拿来用
3. 个别浏览器以及平台的兼容性问题

### 模块化的方案有哪些？

#### AMD、UMD、CommoJs、ESM

1. AMD，异步模块定义，它加载模块的方式是异步的，使用如下：

   ```javascript
   // 定义模块，args: 模块id、依赖、模块内容
   define('getSum', ['calculator'], function () {
     return function (a, b) {
       conosole.log('sum: ' + calculator.add(a, b));
     }
   });
   // 引用模块
   require(['getSum'], function () {
     getSum.add(2, 3);
   });
   ```

2. UMD，通用模块标准（Universal Module Definition），它并不是一种模块标准，而是一组模块形式的集合，它会根据当前全局对象中的值判断目前处于那种模块环境，是AMD就以AMD导出，是CommonJS就以CommonJS导出；

3. 加载CommonJS模块采用require()，存在两种情况：

   1. 该模块未曾被加载过，该模块会首先被执行，然后获取到该模块最终导出的内容；
   2. 该模块已被加载过，直接获取该模块上一次导出的内容

   如果并不是想导入什么东西，而只是想执行某个模块从而达到一定的目的，可以直接require()

### 模块打包工具（module bundler）的任务

我们既想使用模块化，又想让这些模块代码正常运行在浏览器上，那就需要模块打包工具。

module bundler的任务就是解决模块间的依赖，使其打包后能正常运行在浏览器上。

### 模块打包工具的工作方式

1. 将存在依赖关系的模块按照特定规则合并为单个js文件，一次全部加载进页面中
2. 在页面初始时加载一个入口模块，异步加载其它模块（code slitting）

### devServer配置static

如下配置的含义：将dist目录下的文件serve到localhost:8080（webpack服务地址）下。

```javascript
{
  devServer: {
    static: './dist'
  }
}
```

webpack-dev-server启动后，并不会把打包后的bundle真实地写到dist/bundle.js中，而是把bundle放在内存中，并serve到localhost:8080下，也就是说，访问localhost:8080/xxx，都会去dist下面找，不过，localhost:8080/bundle却是从内存中取的，只有bundle才是live-reloading的，也就是说，只有改源码，webpack-dev-server才会重新刷新浏览器，修改dist下的非bundle文件，并不会触发webpack-dev-server的live-reloading。

总之，启动了webpack-dev-server后，如果配置了static: './dist'，那么访问localhost:8080/xxx就会来dist下找，唯一只有被webpack-dev-server打包的那个bundle，和dist目录没关系，它会始终放在内存中，即使dist中有bundle，访问localhost:8080/bundle时，取的也不是dist中的bundle，而是内存中的bundle。

### CommonJS加载

加载CommonJS模块采用require()，存在两种情况：

1. 该模块未曾被加载过，该模块会首先被执行，然后获取到该模块最终导出的内容；
2. 该模块已被加载过，直接获取该模块上一次导出的内容

如果并不是想导入什么东西，而只是想执行某个模块从而达到一定的目的，可以直接require()

### CommonJS和ESM的区别

1. 一个动态，一个静态，CommonJS模块依赖关系的建立发生在代码运行阶段，所以是动态的。ESM模块关系的建立发生在编译阶段，是静态的。

2. CommonJS支持通过if条件判断来require，也支持require()参数是表达式，而ESM都不支持。

3. 由于ESM的的导入导出是静态的，可在编译阶段就分析出模块的依赖关系，因此可以进行**死代码检测和排除**，也可以进行**模块变量类型检查**，以及**编译器优化**（CommonJS导入的都是对象，而ESM支持直接导入变量，减引用层级，程序效率更高）

4. 值复制与动态映射，CommonJS导出的是值的副本，而ESM导出了值的映射。CommonJS可以更改导入的值，因为它本身就是一个副本，不会影响到原模块中的值，而ESM不允许对导出值进行更改。

5. 循环依赖，用例子演示

   ```javascript
   // index.js
   require('foo.js');
   
   // foo.js
   const bar = require('bar.js');
   console.log(bar);
   module.exports = 'foo.js'
   
   // bar.js
   const foo = require('foo.js');
   console.log(foo);
   module.exports = 'bar.js'
   
   // output
   {}
   bar.js
   ```

   foo.js被首次加载后，遇到require('foo.js')，控制权交给bar.js，此时又require('foo.js')，由于foo.js还未执行完，没有完整的导出，所以，在bar.js中只能拿到一个空对象{}，待bar.js执行完，控制权交给foo.js，此时已经拿到bar.js的完整导出。

   **因此，如果在CommonJS中遇到循环依赖，我们将没有办法得到预想中的结果。**

   在ESM中，也会存在这种空对象类似的情况（undefined），但巧妙利用导入导出动态映射，可以让循环依赖也能得到预想的结果，但需要开发者来保证当导入的值被使用时已经设置好了正确的导出值（P32）。

### CommonJS遇到循环依赖，将无法得到预想的结果

因为CommonJS的导入是值的副本，引入的时候是{}，那不论何时，使用的时候它依然是{}，不会和ESM一样，通过动态映射就可拿到准备完毕的值。

### 加载npm模块，引入的是哪个文件？

npm安装好将要用到的包后，直接引入即可

```shell
npm install lodash
```

```javascript
import _ from 'lodash';
```

**webpack在解析到这一行时，会自动去node_modules中寻找名为lodash的模块，而不需要写一条相对路径去找lodash。**

每一个npm模块都有一个入口，在package.json中用main字段记录，当import时，就会引入main字段记录的文件。

当然也可以通过`<package_name>/<path>`的形式单独引模块内部的某个js文件，这样的话webpack就只会打包这个单独的文件，而不会把整个库都打包。

### entry、chunk、bundle的关系

一个工程可能有一个或多个entry，从某个entry出发，将具有依赖关系的模块生成一棵依赖树，这些模块组成一个chunk，由这个chunk打包得到的产物叫做bundle。

既然一个工程有一个或多个entry，那么就会有一个或多个chunk，也就有一个或多个bundle。

特殊情况下，一个entry可能生成多个chunk，最终生成多个bundle。

### entry入口

入口配置由**context**和**entry**共同决定，context是入口路径的前缀，主要目的是让entry编写更加简洁，尤其是多入口情况下，不写context的话，默认是webpack.config.js所在的目录。

entry可以是字符串、数组、对象、函数：

```javascript
// 字符串
module.exports = {
  context: path.join(__dirname, './src'),
  entry: './index.js'
}
```

```javascript
// 数组，作用是将多个资源预先合并，数组最后一个元素作为入口
module.exports = {
  context: path.join(__dirname, './src'),
  entry: ['babel-polyfill', './index.js']
}
// 上面等同于：
// webpack.config.js
module.exports = {
  context: path.join(__dirname, './src'),
  entry: ['babel-polyfill', './index.js']
}
// index.js
import 'babel-polyfill';
```

```javascript
// 对象，想定义多个入口，必须使用对象形式
// 使用对象定义多个入口时，必须为每一个入口定义chunk name
module.exports = {
  entry: {
    // chunk name为index
    index: './index.js',
    // chunk name为lib
   	lib: './lib.js'
  }
}
```

```javascript
// 函数，只需要使匿名函数的返回值为上面任意形式即可
// 使用函数的优势在于可以添加一些代码逻辑来动态地定义入口
// 函数还可以返回Promise来进行异步操作
module.exports = {
  context: path.join(__dirname, './src'),
  entry: () => ({
    index: './index.js',
    lib: './lib.js'
  })
}
```

### 打包时提取vendor的好处（不全）

vender意思为供应商，在webpack里表示工程所使用的库、框架等第三方块集中打包而产生的bundle。

不管是一个还是多个入口，如果能够把它们中的第三方库、框架提取出来单独打成一个bundle，就可以充分利用客户端的缓存，减小请求bundle的体积。第三方库不会经常变动，经常变的是我们的业务bundle，这样客户端再次请求页面时，只需要拿我们的业务bundle，第三方库bundle直接用本地缓存的，就可以提升整个页面的渲染速度。

下面是一个例子，需要用到optimization.splitChunks：

```javascript
// webpack.config.js
module.exports = {
  entry: {
   	foo: './src/foo.js',
    vendor: ['react'],    
  }
  output: {
    path: path.resolve(__dirname, 'dist')
  },
  optimization: {
    splitChunks: {
      // 将选择哪些 chunk 进行优化
      // 表示splitChunks将会对所有的chunk生效，默认情况只对异步chunks生效，并且不需要配置
      chunks: 'all'
    }
  }
}
```

```javascript
// foo.js
import React from "react";
import("./bar.js");
document.write("foo.js", React.version);

// bar.js
import React from "react";
document.write("bar.js", React.version);
```

最终打包会产生四个bundle：foo.js、bar_js.js、vendor.js、vendors-node_modules_react_index_js.js，分别是foo入口生成的bundle、异步bundle、vendor bundle。

### 什么是Tapable？

tap有监听的意思，Tapable是webpack中的一个类，所有由这个类产生的实例都是可监听的；所有webpack中的插件都继承了Tapable（webpack5之后就不再继承了，但API都一样）。

### Loader解决了什么问题，常见的Loader有哪些？

webpack只能识别js和json，其它类型需要Parser用Loader先处理。

常见的Loader有sass-loader，css-loader，style-loader，file-loader，url-loader

### Plugin解决了什么问题，常见的plugin有哪些？

plugin为webpack提供灵活的功能，比如打包优化、资源管理、环境变量注入等。它们会在webpack的不同生命周期运行，解决Loader无法解决的问题。

Plugin本质是具有apply方法的对象，监听Compiler、Compilation的各个hook，在适当的时候调用。

下面是两个plugin的基本结构，该plugin监听了Compiler的afterResolvers hook，在插件被创建时，就会执行apply方法，这里使用了tap方法，具体使用tap、tapAsync还是tapPromise，取决于hook在webpack中是如何定义的。

```javascript
class MySyncWebpackPlugin {
  apply(compiler) {
    compiler.hooks.afterResolvers.tap('MySyncWebpackPlugin', (compilation) => {
      console.log('[compilation] > ', compilation);
    });
  }
}
```

```javascript
class MyAsyncWebpackPlugin {
  apply(compiler) {
    compiler.hooks.done.tapAsync('MyAsyncWebpackPlugin', (stats, cb) => {
      setTimeout(() => {
        console.log('[stats] > ', stats);
        cb();
      }, 1000);
    });
  }
}
```

常见的Plugin有：

1. HtmlWebpackPlugin，打包结束后，会生成一个html模板，并把打包好的bundle插入到模板中。
2. mini-css-extract-plugin，提取css到一个单独的文件中。
3. DefinePlugin，创建一个全局变量，在整个编译阶段都能获取到这个变量。

### tap、tapAsync、tapPromise的区别

tap的作用是监听hook，给tap传的回调函数需要是同步的，而tapAsync中的回调可以是异步的，tapPromise只是tapAsync的另一种写法，只是它返回的是promise。

### loader和plugin的区别

1. webpack只能识别js，对于其它类型的资源，比如图片、css，需要用loader进行转译为webpack能接收的形式，再进行打包；
2. plugin赋予webpack丰富的功能，比如环境变量注入（definePlugin）;
3. loader运行在打包之前，由Parser实例在解析Module Factor传来的模块对象时启用，plugin在整个编译周期都起作用；

### 编写Loader的思路

1. Loader的本质是函数，函数中的this是由webpack提供的对象，可以获取当前loader所需要的所有信息，所以Loader不能是一个箭头函数。
2. 保证Loader的功能单一，比如less转换成css需要less-loader、css-loader、style-loader这个链式调用才能拿到最后的结果。

### 编写Plugin的思路

1. webpack基于发布订阅模式，plugin需要监听Compiler和Compilation中的hook，在特定的时机执行自己的任务
2. 插件必须是一个包含apply方法的对象， 或者是一个函数；
3. 异步的事件需要在插件处理完任务时调用回调函数通知 `Webpack` 进入下一个流程

### 什么是HMR？实现的原理是什么？

HMR全程是Hot Module Replacement，更改某个模块后，webpack会重新打包，并将新的模块发送给浏览器，浏览器用新的模块替换旧的模块，这样就不必刷新整个应用，依然维持当前状态。

### 首屏优化有哪些方案

1. 尽量减小打包的体积
   - 将大部分用于调试的日志放在开发模式下，而生产模式不输出这些日志
   - 按需引入代码
   - 组件懒加载（prefetch，webpack路由组件添加特殊注释）
2. 使用服务端渲染
   - 在服务端就执行Vue代码，生成html字符串，响应给客户端，激活后即可让用户交互，由于在服务端和数据库交互速度更快，因此会降低首屏时间。
3. 使用预渲染
   - 如果有一些内容，对任何用户都做相同的展示，就可以在打包期间进行渲染，生成html字符串，这样只需渲染一次，直接响应，不需要在每一次请求都渲染一次。
4. 按照页面或者组件分块懒加载
5. 使用gzip减小网络传输的流量大小

### prefetch和preload的区别

它们解决的问题是你在浏览一个网页的时候，各种交互可能会需要发送网络请求，但这些资源的请求并不一定非要到用户确实需要使用的时候才请求，而是可以提前发送请求，缓存在内存中，但不执行，等到要使用时，再取用执行。

prefetch是利用浏览器的**空闲时间**来下载用户即将使用的资源，不会立刻执行，只是缓存在内存。如果用户并没有用到prefetch的资源，那这段网络开销就算浪费了，所以使用较为频繁的资源，就可以用prefetch。

preload是页面的渲染必定会用到某个资源，在最开始就请求它，不需要等到解析html到确实该资源时才去请求。

主要区别是：

1. preload是告诉浏览器必定会使用的资源，浏览器一定会加载，而prefetch是告诉浏览器可能会使用的资源，浏览器不一定会加载这些资源；
2. 一般首屏的资源都是preload，路由对应的资源都是prefetch

### Babel的工作流程

1. 解析，通过解析器进行词法分析和语法分析，把源码转换成AST抽象语法树；
2. 转换，操作AST，可以增删，转换成想要的样子
3. 代码生成，深度遍历第2步的抽象语法树生成新的代码

### 如何使用webpack来打包的

确定打包入口、输出文件，引入各种loader、plugin

### webpack插件（taptable）

## webpack利用文件系统缓存

webpack.config.js文件中的cache字段可以配置缓存策略，在build模式下，因为打包完webpack进程就会关掉，所以只能使用文件系统缓存，配置为：config.type: 'filesystem'；在watch模式下，缓存放在内存中，由webpack自身来管理，不用人工介入。

按照缓存规则，在打包一次之后，webpack会将module、chunk、moduleGraph缓存下来，下一次打包如果发现某个文件源代码变了，就要重新编译该文件，如果没有变，则直接使用缓存。

管理缓存的方法：

1. 打包依赖。有一些特殊的依赖包，如果它的源码变了，就要让全部的缓存失效，整个重来，所以可以在cache.buildDependencies中配置好这些特殊的依赖。
2. 缓存名称。webpack默认使用`${config.name}-${config-mode}`的目录形式存放缓存，即使源代码不变，在dev模式下的打包缓存会放在`${config.name}-development`文件夹下，而在prod模式下打包的缓存会放在`${config-name}-production`目录下。
3. 缓存版本。当赋予cache.version一个新值时，打包就不会再使用旧的缓存了。

## Compiler是什么？

Compiler是webpack中的一个核心类，它控制着总任务流，把任务分给其他模块（大多都是插件，这些插件和通过npm安装引入到webpack配置中的茶碱没有本质区别）来做。Compiler的工作方式是暴露出整个工作流程中的hook，插件监听这些hook，在相应的时机完成工作，所以，Compiler只负责做调度，掌控整体节奏而不负责具体工作。

## Compilation是什么？

它也是webpack中处于核心地位的一个类，可以把Compiler看成是总指挥官，而Compilation是总管，负责更细致的工作。它也有很多hook，让其他模块监听更细致的事物。

## Resolver是什么？

找到入口文件、构建依赖关系图时寻找每个依赖文件都是Resolver的工作。但Resolver并不会返回源代码，而是返回找到的这个模块的相关信息。

## Module Factor是什么

Module Factor类似于函数，接收的是Resolver返回的信息，返回的是模块对象（包含源码）。

## Parser是什么？

从Module Factor得到的模块源代码是各式各样的，要将它们变成webpack能够理解的形式，因此，就由Parser来根据webpack的配置，用不同的loader将它们都变成js。

将源代码都转译成js后，Parser还需要将这些js转化为抽象语法树，并进一步寻找依赖，如果又找到了新的依赖，就继续使用Resolver、Module Factor。直到所有的模块都处理完毕。

## webpack的构建流程（打包机制）

1. 准备工作，通过命令行和配置文件获取本次打包所需的配置项，对配置项进行一次检查；
2. 缓存加载，根据配置文件中的cache字段的配置来加载已有的缓存，利用缓存可提高构建效率。
3.  模块打包
   - 首先，初始化Compiler和Compilation，这只是打包流程的开始；
   - 下一步是构建依赖关系图，其步骤是：从入口开始，由Resolver找到模块的具体位置，返回模块信息（不包含源码）；Module Factor拿到模块信息后返回模块对象（包含源码）；Parser按照配置规则，使用对应的loader将模块转化为js，随后将js解析为AST，并进一步寻找依赖；循环此步骤直到所有的依赖都被处理完成。webpack把这些最终的产物组合在一起，生成chunk（此处会有特殊处理，比如异步module会被单独打成一个chunk）
   - 最后，用模板渲染将chunk转化为最后的代码（bundle）。模板渲染实际上就是根据依赖关系图以及前面各步骤得到的模块信息，让webpack组织和拼装模板（对于不同类型的模块，webpack都有对应的模板），把实际模块内容填进去，最终“渲染”得到我们看到的目标代码。可以理解成webpack已经搭好了架子，就等着把模块内容填进去了。

## live reload 和 HMR的区别

live reload是修改代码后自动触发浏览器刷新，HMR不会刷新浏览器，会替换修改的部分，而保证原来的页面状态不变，从而节省开发者大量的时间成本。

## webpack热更新（HMR）的原理？

如果是本地开发，浏览器将作为客户端，webpack-dev-server作为服务端，HMR的核心是客户端从服务端拉取更新后的资源（实际上是chunk diff）。

1. 浏览器何时拉取这些更新？WDS和浏览器之间维护一个websocket，并且WDS对本地文件进行监听，当本地文件发生改变时，WDS向浏览器推送更新事件，并带上这次构建的hash，让客户端和上一次资源的hash进行对比，一样就不用变；
2. 拉取什么？当对比hash后确定有改动，浏览器就向WDS发请求获取更改文件的列表，即哪些模块有了改动，请求的名字通常为[hash].hot-update.json；
3. WDS告知需要更新的chunk为某一个模块后（比如是main），浏览器继续向WDS获取该chunk的增量更新，拿到结果。
4. 要如何处理获取到的增量更新？哪些地方要更新？这不属于webpack的工作了，react-hot-loader和vue-loader借助webpack的API来实现HMR。

## webpack proxy工作原理？为什么能够解决跨域问题？

webpack proxy仅用于本地**开发模式**下解决跨域问题，因为浏览器有同源策略，调试的时候webpack-dev-server开的服务地址一般都是localhost，后端资源又在其他服务器上，肯定会被浏览器的同源策略限制导致跨域问题。

但是服务器与服务器之间请求数据并不存在跨域问题，跨域行为只是浏览器的安全策略限制，因此，在浏览器请求跨域资源时，利用webpack的proxy代理一下，然后转发给浏览器，而代理服务器传递数据给本地时，因为两者同源，就不存在跨域问题了。

proxy工作原理：利用http-proxy-middleware实现把请求转发给其他服务器。

```javascript
const express = require('express');
const proxy = require('http-proxy-middleware');

const app = express();

app.use('/api', proxy({target: 'http://www.example.org', changeOrigin: true}));
app.listen(3000);
```

webpack像下面这样配置，proxy的属性名就是匹配规则

```javascript
module.exports = {
    // ...
    devServer: {
        contentBase: path.join(__dirname, 'dist'),
        compress: true,
        port: 9000,
        proxy: {
            '/api': {
                target: 'https://api.github.com'
            }
        }
        // ...
    }
}
```

## 利用webpack来优化前端性能的方法

从两个方面入手：

1. 压缩文件体积，比如Tree Shaking、js代码压缩（terser-webpack-plugin，默认的）、css压缩（css-minimizer-webpack-plugin）、html压缩（HtmlWebpackPlugin）；
2. 代码分包（split chunk），让首屏的bundle小一点，按需加载；

## 如何提高webpack的构建速度

从三个方面入手：

1. 优化搜索时间
   - 对于引入时没有尾缀的文件，采用 resolve.extensions，把频率最高的尾缀放在前面
   - 用resolve.alias，直接确定文件所在的绝对路径，相对路径会一层一层寻找
2. 缩小文件搜索范围
   - 使用loader时，用include、exclude，把不必要使用loader的文件排除掉
   - 用resolve.modules配置webpack去哪些目录下寻找第三方模块
3. 减少不必要的编译
   - 合理使用sourceMap，越详细打包越慢
   - 对开销较大的loader使用cache-loader，显著减少第二次打包的时间

## 还有哪些模块打包工具？

1. Rollup是ES module打包器，代码简洁，效率高，默认Tree-Shaking，但不支持HMR，并且对于CommonJS模块的打包需要使用插件，而当前大部分的第三方库都是CommonJS，因此Rollup不太适合开发应用使用，但对于js库很不错，打包快、体积小。
2. Parcel不需要配置，只需了解简单的命令，即可打包。
3. Snowpack
4. Vite，`vite`会直接启动开发服务器，不需要进行打包操作，也就意味着不需要分析模块的依赖、不需要编译，因此启动速度非常快。它由一个开发服务器和一套构建指令组成，用Rollup打包代码。在HMR方面，当修改一个模块时，仅仅需要让浏览器请求该模块即可。
