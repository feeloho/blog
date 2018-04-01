# 使用 PHP 的 Phar 包实现程序自动更新

前两天有 PHP 的道友问我如何拿 PHP 实现程序自动更新。但是我对自动更新这个功能，说实话是并没有太大兴趣去搞的。我认为我把我写的程序源码放到 GitHub 这种代码托管平台上，然后如果你需要用的话就直接 clone 下来丢到服务器上跑就行了，没必要再单独搞个 安装/升级/更新程序。可能这就是典型的程序员思维方式吧。

但是换个角度从产品的角度，产品的易用性、完整性来讲，确实加上这个功能对部分用户来讲会方便的多，也便于产品的推广。

总之，来探索一下如何使用 PHP 简单的实现程序自动更新。这里我想到的是用 Phar 包的方式实现。



## 01. 什么是 Phar

维基百科是这样介绍的：

> PHAR（PHP 归档）文件是一种打包格式，通过将许多 PHP 代码文件和其他资源（例如图像，样式表等）捆绑到一个归档文件中来实现应用程序和库的分发。
>
> PHAR 文件可以是三种格式之一：tar 和 ZIP（它们与各自的工具相兼容），以及自定义的 PHAR 格式。无论使用何种格式，所有 PHAR 文件都使用 .phar 作为文件扩展名。使用标准的 tar 和 zip 应用程序可以创建和解压缩 Tar 和 Zip 格式的归档，而 PHAR 格式的归档需要使用自己写的 PHP 代码（利用 PHP 的 PHAR 扩展）或者使用 PEAR 的 PHP 归档包来创建和提取。

简单来说就是把一堆文件打包成一个 phar 文件。和前端的 Webpack 等打包有点像，只不过 Webpack 是把一堆文件打包成一个 js 文件。

 

## 02. 目录结构

```
server
├── build.php     // 构建 phar 文件的脚本
├── bundle.phar   // 构建好的包
├── index.php     // 用于提供更新服务(最新版本、更新代码)
└── src           // 实际程序代码的目录
    └── index.php
client
├── bootstrap.php  // 从 server 拉下来的更新代码，也就是 server 中 index.php 里的「更新代码」
├── bundle.phar    // 最新的包，在 bootstrap.php 文件从 server 中拖下来的
├── index.php      // 用来下载 bootstrap.php 文件
└── version        // 最新版本，也就是 server 中 index.php 里的「最新版本」
```

其中 server 目录中的代码是需要架设到服务器提供更新服务的。client 则是用户使用的代码。server 目录中的 `bundle.phar` 文件是 build.php 执行后产生的。client 目录中的 `bootstrap.php`、`bundle.phar`、`version` 文件是 index.php 被访问时所产生的。



## 03. 构建脚本 [server/bundle.php]

要想打包源码为 phar 文件需要先修改一下 php.ini 文件，将 `phar.readonly` 的值修改为 `Off` 。

```ini
[phar]
phar.readonly = Off
```

然后将下面代码保存到 `server/bundle.php` 文件中：

```php
<?php

$p = new Phar('bundle.phar');
$p->startBuffering();
$p->buildFromDirectory('src');
$p->compressFiles(Phar::BZ2);
$p->stopBuffering();

echo 'done';
```

这些代码会将 `src` 目录下的文件打包成 `bundle.phar` 文件，使用的压缩格式是 bz2。然后在终端执行此脚本即可开始构建（注意确保 src 目录存在）。



## 04. 更新服务 [server/index.php] 

第二步执行完成后会产生一个 `server/bundle.phar` 文件，下面的代码依赖于此文件，所以确保第二步已经成功执行。

```php
<?php

// 最新版本
if (isset($_GET['version'])) {
    exit(sha1_file('bundle.phar'));
}


// 更新代码
$url = "http" . (isset($_SERVER['HTTPS']) ? 's' : '') . "://{$_SERVER['HTTP_HOST']}/";

echo <<<EOF
<?php

\$version = file_get_contents('{$url}?version');

if (file_get_contents('version') !== \$version) {
    file_put_contents('version', \$version);
    file_put_contents('bundle.phar', file_get_contents('$url'));
}

require('bundle.phar');

EOF;

```

上面的代码主要是提供给客户端作 api 服务调用，只有 2 个功能：获取最新版本号、获取更新代码。



## 05. 客户端 [client/index.php]

```php
<?php

if (!is_file('bootstrap.php')) {
    copy('http://127.0.0.1/phar', 'bootstrap.php');
}

require 'bootstrap.php';
```

上面代码的用途是从更新服务端下载更新代码，然后保存为 bootstrap.php 文件并且引入。注意代码中的 `http://127.0.0.1` 是已架设更新服务的 ip 或域名地址。



## 06. 总结

仅仅那么十几行代码就完成了自动更新的功能，可以说是相当简单粗暴了。不过在这里边也有许多可以扩展、可以优化的地方，这里就先做省略，主要是提供一种思路。而 Phar 最大的优势就是下载下来后可以直接 require 使用，从而避免了很多麻烦。