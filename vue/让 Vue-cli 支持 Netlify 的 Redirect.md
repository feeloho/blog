# 让 Vue-cli 支持 Netlify 的 Redirect

最近有个 Vue 项目，是使用 Vue-cli 构建起来的。Vue-Router 是使用的 [HTML5 History](https://router.vuejs.org/zh-cn/essentials/history-mode.html) 模式。这个模式是使用浏览器提供的 History 接口实现的，所以如果是在页面上通过点击链接往里面 push 一个新地址是没问题的，但是如果直接请求这个地址则会 404 （因为这个地址并不是真实存在的，只是我们通过代码模拟出来的）。

所以 Vue-Router 的手册中也给出了解决方式：通过配置重写规则把 404 统一重定向到 index.html 文件。但是我是一个很懒的人，懒得自己配环境，所以直接用的 Netlify 提供的免费服务。

当然 Netlify 很强大，提供了 redirect 的功能，是通过添加 `_redirects` 规则文件到**根目录**下来实现的。具体可以参照：https://www.netlify.com/docs/redirects/

问题就是如何在 `npm run build` 的时候自动把 `_redirects` 文件添加到根目录呢？



## 01. 尝试

```
.
├── build
├── config
├── node_modules
├── src
└── static

5 directories
```

上面的是由 vue-cli 所构建出来的项目的目录结构。起初我尝试把 `_redirects` 放到 `static` 目录中，但是失败了。是因为 `npm run build` 后 `static` 目录中的文件并不是放到了根目录（这里是 `dist`）中，而是在根目录中的 `static` 目录内。

```
dist
├── index.html
└── static
    ├── _redirects  <-- 在这里
    ├── css
    ├── fonts
    └── js

4 directories, 10 files
```



## 02. 查资料

于是我 Google 了一圈，找到了这个 issues ：https://github.com/vuejs/vue-cli/issues/285，但是这貌似也不是我想要的。所以没办法了，只能自己动手了。



## 03. 瞎™改

看了眼 `package.json` 文件里的 build 是执行了 `node build/build.js` ，打开这个文件一直跟踪到了 `build/webpack.prod.conf.js` 文件里，在 90 行左右有这样的代码：

```js
// copy custom static assets
new CopyWebpackPlugin([
  {
    from: path.resolve(__dirname, '../static'),
    to: config.build.assetsSubDirectory,
    ignore: ['.*']
  }
])
```

使用了 [CopyWebpackPlugin](https://github.com/webpack-contrib/copy-webpack-plugin) 这个插件来把 static 中的文件拷贝到了 `config.build.assetsSubDirectory` 下。

assetsSubDirectory 的定义在 `config/index.js` 文件中，应该就是指向了 `dist/static` 这个目录。

```js
module.exports = {
  build: {
    env: require('./prod.env'),
    index: path.resolve(__dirname, '../dist/index.html'),
    assetsRoot: path.resolve(__dirname, '../dist'),
    assetsSubDirectory: 'static',  // <-- 在这里
    assetsPublicPath: '/',
    productionSourceMap: true,
    // ....
```

所以鸡汁的我就修改了一下 `build/webpack.prod.conf.js` 的代码：

```bash
+    // copy _redirects file
+    new CopyWebpackPlugin([
+      {
+        from: path.resolve(__dirname, '../static/_redirects'),
+        to: config.build.assetsSubDirectory + '/..'
+      }
+    ]),
     // copy custom static assets
     new CopyWebpackPlugin([
       {
         from: path.resolve(__dirname, '../static'),
         to: config.build.assetsSubDirectory,
-        ignore: ['.*']
+        ignore: ['.*', '_redirects']
       }
     ])
```

把 ```_redirects``` 文件复制到 dist/static 的上级目录 `/..` 中。完成啦~



## 04. 总结

这种修改文件的方法可能是最简便的了，但是随便修改这些文件显得不是很专业嘛。所以大家如果有别的方法请告诉我。

