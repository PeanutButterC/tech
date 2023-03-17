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









