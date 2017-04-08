# WebPack 前端工具使用笔记

> [Webpack](https://github.com/webpack/webpack) 是当下最热门的前端资源模块化管理和打包工具。它可以将许多松散的模块按照依赖和规则打包成符合生产环境部署的前端资源。还可以将按需加载的模块进行代码分隔，等到实际需要的时候再异步加载。通过 `loader` 的转换，任何形式的资源都可以视作模块，比如 CommonJs 模块、 AMD 模块、 ES6 模块、CSS、图片、 JSON、Coffeescript、 LESS 等。



### 创建 package.json 文件

首先, 先在项目目录下面创建 package.json 文件, package.json 文件中定义了项目的相关信息, 以及他所依赖于哪些模块.

```bash
npm init
```

执行上面的命令后, 会提示你输入项目的相关信息, 根据所给出的相关提示输入即可.



### 安装 WebPack

```bash
// 安装到全局
npm install -g webpack

// 安装到当前项目
npm install --save-dev webpack
```



执行命令安装时, 若提示没权限, 那么前面加个 `sudo` , 或者切换到权限较高的账户下安装.

还有一点注意的是, 如果没有安装到全局, 那么文章下面所有带 `webpack` 命令的, 都应该换成

```bash
node_modules/.bin/webpack
```

当然也有其他方法, 那就是通过修改 `package.json` 文件, 在 `scripts` 块中添加 ` "start": "webpack"`

![](https://att.sxyz.blog/image/xk4ad.png)

然后下文的 `webpack` 替换为

```bash
npm start
```

`start` 是在 npm 中的一个特殊关键词, 所以可以直接使用 `npm start` , 但是如果不写 `start` 而写其他名字, 比如 `yazi` , 那么替换的命令应该是

```bash
npm run yazi
```



### 简单使用一下

创建一个叫 `test` 的目录用于测试, 之后里面新建两个文件, 文件名分别为 `hello.js` `main.js`

其中 `hello.js` 的目的用于测试, 而 `main.js` 则是作为程序的入口, 引入 `hello.js` 文件.

##### hello.js: 

```javascript
alert("hello yazi!");
```

##### main.js: 

```javascript
require("./hello");
```



然后执行下面的命令: 

```bash
webpack test/main.js ./build.js
```

![](https://att.sxyz.blog/image/c1o69.png)

上面的命令通过我刚才创建的 `test` 目录中的 `main.js` 文件在项目目录下生成了一个 `build.js` 文件.



接着创建一个 html 文件, 引入刚才所生成的那个 `build.js` 文件.

##### test.html

```html
<script src="./build.js"></script>
```



上面的步骤都完成以后, 目录结构应该是如下这样的: 

![](https://att.sxyz.blog/image/kex0v.png)



在浏览器打开 `test.html` 文件后. 可以看到执行了 `test.js` 文件中的内容, 弹出了 **hello yazi** 的提示框.

![](https://att.sxyz.blog/image/iltc4.png)



最后打开 `build.js` 文件看看, 可以找到我在 `test.js` 中所写的 `alert("hello yazi!")` 这行代码

![](https://att.sxyz.blog/image/e8wtn.png)



### 使用 WebPack 配置

每次都要在终端去敲那么长的命令, 很麻烦有木有. 那么现在我可以使用 WebPack 的配置来简化一下流程.



在项目目录创建一个叫做 `webpack.config.js` 的文件, 没错这个就是 WebPack 的配置文件的名字了.

在里面写入下面的内容: 

```json
module.exports = {
	// 入口文件
	entry: __dirname + "/test/main.js",
	output: {
		// 生成的文件所在目录
		path: __dirname,
		// 生成的文件的名字
		filename: "build.js"
	}
}
```

其中 `__dirname` 代表的是项目目录. 他是在 node.js 中所提供的一个全局变量.



然后在终端执行: 

```bash
webpack
```

同样可以达到相同生成的效果, 只不过是输入的命令更简短一些了.



### WebPack 中的 Loader

通过 Loaders 可以使 WebPack 对各种类型的文件进行处理. Loader 需要手动安装并配置在 `webpack.config.js` 文件中的 `module` 块下. 下面我将使用 JSON Loader 处理 JSON 文件.

##### 安装 

```bash
npm install --save-dev json-loader
```

##### 配置

这里有几个可用的配置项参考: 

- `test` : 文件名匹配正则（必选）
- `loader` : loader名（必选）
- `query` : 额外设置选项（可选）
- `include` : 手动指定必须处理的文件或文件夹（可选）
- `exclude` : 手动指定不要处理的文件或文件夹（可选）

```json
module.exports = {
	// 入口文件
	entry: __dirname + "/test/main.js",
	output: {
		// 生成的文件所在目录
		path: __dirname,
		// 生成的文件的名字
		filename: "build.js"
	},
	// 添加module块
	module: {
		loaders: [
			{
				test: /\.json$/,
				loader: "json-loader"
			},
			{
				...
			}
			...
		]
	}
}
```

##### 使用

将 `test` 目录下面的 `hello.js` 文件重命名为 `hello.json` , 并修改内容如下: 

```json
{
	"balbal": "hello yazi!"
}
```

再修改 `test/main.js` 入口文件的内容: 

```javascript
var hello = require("./hello");
alert(hello.balbal);
```

执行 `webpack` , 打开项目目录下的 `test.html` 文件, 可以看到弹出了 `hello yazi!`

---

上面就完成了对JSON文件的打包, 接下来说下对CSS文件的打包.

##### 安装

对CSS文件打包, 依赖 2 个 loader , 第一个 `css-loader` 是对CSS文件中的 `url(..)` 进行处理. 第二个 `style-loader` 是将CSS放到页面中. 这里需要两个配合使用, 先处理掉 `url()` , 然后再把 **处理后** 的内容加入到页面中.

```bash
npm install --save-dev css-loader style-loader
```

##### 配置

```json
module.exports = {
	// 入口文件
	entry: __dirname + "/test/main.js",
	output: {
		// 生成的文件所在目录
		path: __dirname,
		// 生成的文件的名字
		filename: "build.js"
	},
	// 添加module块
	module: {
		loaders: [
			{
				test: /\.css$/,
				loader: "style-loader!css-loader"
			}
		]
	}
}
```

需要注意的地方是: 多个loader中间我使用 `!` 分隔开了. 在 WebPack 读取配置的时候, 会 **依次** 读取到 `css-loader`  `style-loader` (从右往左) , 所以执行时也是先 css 再 style.

##### 使用

配置完成之后, 就剩下测试了, 把 `test/hello.json` 文件改名为 `hello.css` , 并修改内容: 

```css
body {
	background: yellow;
}
```

再把 `test/main.js` 中的内容修改为: 

```javascript
require("./hello.css");    # 注意到没, 前面我写的都是"hello"
```

执行 `webpack` , 然后打开项目目录下的 `test.html` 文件, 然后.. 窝的眼...



### WebPack 中的 Plugin

再介绍一下 WebPack 里面的插件. 其实 Plugin 与 Loader 意义差不多(都是为了帮我搞一些事情) , 但是他们两个最大的区别应该是分工不同. Loader 的作用已经知道了, 是对比如 JS文件 CSS文件 JSON文件这些打包的, 而 Plugin 是干嘛的, 我写个例子来看下.

这里以 `UglifyJsPlugin` 这个插件来介绍, 用它我可以对生成的 `build.js` 文件压缩.

##### 安装

次插件无需安装, 是 WebPack 所自带的.

##### 配置

插件的配置依然是放在 `webpack.config.js` 文件中, 插件的信息放在 `plugins` 块中.

修改 `webpack.config.js` 文件的内容为: 

```json
module.exports = {
	// 入口文件
	entry: __dirname + "/test/main.js",
	output: {
		// 生成的文件所在目录
		path: __dirname,
		// 生成的文件的名字
		filename: "build.js"
	},
	// 添加module块
	module: {
		loaders: [
			{
				test: /\.css$/,
				loader: "style-loader!css-loader"
			}
		]
	},
	// 添加plugin块
	plugins: [
		new (require("webpack").optimize.UglifyJsPlugin)
	]
}
```

所有插件均写在 `plugins` 块中. 以数组的形式来表示. 由于我所使用的 `UglifyJsPlugin` 插件是 WebPack 带的, 所以在使用前需要引入 `webpack` 这个对象, 我使用 `require()` 函数将其引入.

##### 使用

在 `test` 文件夹内, 新建 `word.js` 文件, 内容为: 

```javascript
for(var i=0; i<10; i++)
{
	document.write("我是word.js");
}
```

修改 `test/main.js` 中的内容为: 

```javascript
require("./word");
require("./hello.css");
```



执行 `webpack` 命令, 使用 Sublime 编辑器打开生成后的 `build.js` 文件, 可以看到压缩效果.

![](https://att.sxyz.blog/image/fx7v9.png)

