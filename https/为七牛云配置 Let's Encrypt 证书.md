# 为七牛云配置 Let's Encrypt 证书

在上一篇文章《Let's Encrypt 免费 HTTPS 证书申请》中，介绍了通过 `文件验证` 的方式申请 Let's Encrypt 证书，并为 Nginx 配置证书的整个流程。

那么这篇文章来介绍下如何通过 `DNS 验证` 的方式申请并为 七牛云 配置 SSL 证书。

那么可能会有人说，七牛云不是提供了一个 1 年的免费证书申请服务么，为什么你还要自己申请 Let's Encrypt 的证书再配置，那不是很麻烦么? 其实我也不想啊！但是那东西一直申请失败，提示 `CA验证失败` ，提交工单说是黑盒申请七牛也无能为力。我猜想可能是我域名后缀不支持吧。



## 01. 安装 acme.sh

全程是通过 acme.sh 这个工具来申请的，所以第一步要先安装一下。

```bash
curl https://get.acme.sh | sh
```



## 02. 申请证书

先说下为什么这里使用 `DNS 验证` 的方式申请验证域名: 

> 其实原因很简单，因为我域名绑定到了七牛，如果使用 `文件验证` 的方法申请，那么就要保证在七牛空间里面有这个验证文件，并且还要能够访问到。所以这时候就不太方便了，然后我就选择了 `DNS 验证` 。



DNS 验证方式介绍: 

> DNS 方式，在域名上添加一条 txt 解析记录，验证域名所有权。
>
> DNS 方式的真正强大之处在于可以使用域名解析商提供的 api 自动添加 txt 记录完成验证。
>
> acme.sh 目前支持 cloudflare，dnspod，cloudxns，godaddy 以及 ovh 等数十种解析商的自动集成。



我使用的是 cloudflare，所以执行下面的命令: 

```bash
export CF_Key="你的API秘钥"
export CF_Email="cloudflare 登录邮箱"

~/.acme.sh/acme.sh --issue --dns dns_cf -d ibllk.pw
```

其他的用法可以参考 https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md



执行上面命令后，会自动使用 api 添加 txt 记录到 cloudflare，并等待 120 秒。

等待结束后，会继续进行下一步操作，如果验证通过，那么证书就申请成功了。



## 03. 配置证书

上面的步骤都执行完后，那么就代表证书已经申请成功了。默认证书是存放在 `~/.acme.sh/` 目录中的。

执行下面命令把证书复制到 `/root/ibllk/` 目录里面: 

```bash
mkdir /root/ibllk
~/.acme.sh/acme.sh  --installcert -d ibllk.pw   \
        --key-file        /root/ibllk/ibllk.pw.key \
        --fullchain-file  /root/ibllk/ibllk.pw.cer
```

其中 `ibllk.pw.cer` 是证书文件，`ibllk.pw.key` 是证书的私钥文件。



## 04. 证书续期

acme.sh 能够自动检查证书有效期，并自动续期。安装 acme.sh 后他会添加一条定时任务: 

```bash
crontab -l
8 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

如果发现证书失效，我们刚才执行的命令都会自动重复一遍以完成证书续期。



## 05. 七牛自动同步更新证书

这个等有时间我拿 Python 写个脚本更新。

