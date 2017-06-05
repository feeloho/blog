# Composer 配合 SS 实现依赖加速下载

Composer 在 PHP 开发中绝对算得上是一个利器，现在几乎整天都离不开它了。但是下载依赖安装的过程却非常慢，因为是国外的嘛。这时候可能就需要使用某些手段来加速一下了。这里我使用的是拿 SS 作为一个代理服务器来实现的。（话说不是有个什么 [中文镜像](https://pkg.phpcomposer.com/) 的么，不过可能由于强迫症原因不太喜欢用。）

其实拿 SS 来实现优势还是蛮大的，因为 SS 相当于一个 服务 的概念，不光可以为 Composer 提供服务，其他的比如 npm pip brew 什么的也都可以去那么用了。这里主要写 Composer 的原因是在使用 Composer 途中遇到了一个坑，记录一下。



## 01. 配置终端使用代理

打开终端输入

```bash
export http_proxy=socks5://127.0.0.1:1086
export https_proxy=$http_proxy
```

1086 是 SS 在本地所监听的端口，根据你自己的去配置。

但是这样有个问题，每次打开终端都要去输那么长的命令，非常费劲。所以我的做法是加了个 alias 。

```bash
alias proxy="export http_proxy=socks5://127.0.0.1:1086; export https_proxy=socks5://127.0.0.1:1086"
```

这样所有在终端执行的程序应该就都能够使用代理服务器来得到加速的效果了。可以看下IP地址：

```bash
☁  ~  curl ip.gs
Current IP / 当前 IP: xx.xx.xx.xx
ISP / 运营商:  sakura.ad.jp
City / 城市: Tokyo Tokyo
Country / 国家: Japan

  /\_/\
=( °w° )=
  )   (  //
 (__ __)//
```



## 02. Composer 中踩到的一个坑 

以为配置好代理就可以用 Composer 了。但是在使用时却报错了。

```bash
☁  test  composer install
The "https://packagist.org/packages.json" file could not be downloaded: failed to open stream: Unable to find the socket transport "socks5" - did you forget to enable it when you configured PHP?
```

为了解决这个问题，我尝试着重装了 PHP 的各种扩展（curl sockets openssl 等），发现并没有什么卵用。

之后我在 GitHub 发现了 [这个Issues](https://github.com/composer/composer/issues/2900) ，我猜测是 PHP 没有正确识别 socks5 协议。而 Composer 也对此没有做什么处理。其实我感觉加入这一行代码到 Composer 中情况可能会有所改变：

```php
curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_SOCKS5_HOSTNAME);
```

但是最佳解决问题的方式是尽量不要改动 Composer 的代码。所以我找到了 proxychains。



## 03. proxychains 工具配置

安装 proxychains ：

```bash
brew install proxychains-ng
```

创建配置文件 `~/.proxychains/proxychains.conf` ，其内容为：

```ini
strict_chain
proxy_dns 
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
quiet_mode

[ProxyList]
socks5  127.0.0.1 1086
```

同样的，1086 修改成你的 SS 端口号。此时在执行 Composer 的时候就需要在前面添加上 `proxychains4` ：

```bash
proxychains4 composer install
```

不过名字有点长，可以也加个 alias

```bash
alias pc="proxychains4"
alias composer="proxychains4 composer"
```

有个需要注意的问题是，proxychains 不能代理 /bin /sbin /usr/bin 这些目录中的程序。原因是 MacOS 系统加了保护，可以使用下面的方式关掉，不过不太建议。

```bash
csrutil disable
reboot  # 系统会重启
```

