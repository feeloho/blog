# Nginx 配置

这篇文章写了 Nginx 常见的一些配置选项，整理并记录下来方便以后使用。



## 01. 目录结构

安装好后的目录结构如下：

```
nginx
├── conf    # 配置文件
├── html    # 网站程序
├── logs    # 日志文件
└── sbin    # 二进制程序文件
```

配置文件默认是放在 `conf` 目录中的 `nginx.conf` 文件。



## 02. 区段介绍 

```nginx
# 全局区段

events {      # 事件区段
}

http {        # HTTP区段

    # HTTP全局区段

	server {  # Server区段

		# Server全局区段

		location 匹配模式 {    # Location区段
		}

	}
}
```

### 全局区段

```nginx
# nginx进程pid号存放文件
pid /var/run/nginx.pid;

# 运行的用户与用户组
user 用户 用户组;  # 默认为 nobody nobody

# 工作进程数
worker_processes CPU数量*CPU核数;  # 默认为 1

# 日志文件
# 可选级别: debug|info|notice|warn|error|crit|alert|emerg
error_log /var/log/nginx.error.log 级别;
```

### 事件区段

```nginx
# 事件驱动模型
# 可选模式: select|poll|kqueue|epoll|resig|/dev/poll|eventport
use epoll;

# 设置工作进程轮流接受新连接
accept_mutex on;  # 开启可防止惊群现象

# 工作进程是否可同时接受多个连接
multi_accept on;  # 默认为 off

# 最大连接数
worker_connections 1024;  # 默认为 512
```

### HTTP 区段

```nginx
# 错误页面
error_page 500 502 503 504 /50x.html;

# 定义服务器组
upstream 服务器组名 {   
    server ip:port;
    server ip:port backup;                          # 热备
    server ip:port weight=1;                        # 权重
    server ip:port max_fails=2 fail_timeout=30s;    # 最大重试 超时
}
```

### HTTP 全局区段

```nginx
# 引入文件类型映射表
include mime.types;

# 默认类型
default_type application/octet-stream;

# 自定义日志格式
log_format 格式名称 '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for';

# 用户请求日志
access_log /var/log/nginx.access.log 格式名称;  # off则关闭

# 减少上下文切换 数据拷贝次数
sendfile on;  # 默认为off

# 限制sendfile() 减少阻塞
sendfile_max_chunk 512KB;  # 默认为0

# 连接超时
keepalive_timeout 65;

# 负载均衡
proxy_pass http://服务器组名;
```

### Server 全局区段

```nginx
# 监听端口
listen      80 443;

# 服务器名
server_name sxyz.blog www.sxyz.blog;

# keepalive 最大请求数
keepalive_requests 100;
```

### Location 区段

```nginx
# 根目录
root path;

# 默认主页
index index.htm index.html;

# 禁止IP
deny 127.0.0.1;
deny all;  # 禁止所有

# 允许IP
allow 120.121.122.123;
allow all;  # 允许所有

# 反向代理
proxy_pass http://ip:port;

# 添加 X-Forwarded-For 字段
proxy_set_header X-Forwarded-For $remote_addr;
```



参考链接：

http://nginx.org/en/docs/ngx_core_module.html

