# Let's Encrypt 免费 HTTPS 证书申请

我博客使用的是 Let's Encrypt 的证书，为什么要用他呢？免费啊！而且还很方便，官网上写了这样一句话：

> Let’s Encrypt is a **free**, **automated**, and **open** Certificate Authority。

之前想整理下申请过程的，可是一直没时间，现在来写一下叭。其实 Let's Encrypt 申请使用过程很简单的。



## 01. 生成身份密钥

```bash
mkdir /opt/cert/ && cd /opt/cert    # 后面的操作都在此目录进行
openssl genrsa 4096 > account.key
```



## 02. 生成域名密钥

```bash
openssl genrsa 4096 > domain.key
```

创建 csr 文件：

```bash
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:sxyz.blog,DNS:www.sxyz.blog,DNS:att.sxyz.blog")) > domain.csr
```

如果提示 `cat: /etc/ssl/openssl.cnf: No such file or directory` ，那么换成下面的试下：

```bash
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/pki/tls/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:sxyz.blog,DNS:www.sxyz.blog,DNS:att.sxyz.blog")) > domain.csr
```



## 03. 验证配置

Let's Encrypt 的域名所有权验证方式是请求你的域名下面的某个文件，所以这里需要配置一下 Nginx 让其能够请求的到。

创建个目录作为验证时的请求目录：

```bash
mkdir -p /var/www/challenges
```

配置 Nginx：

```nginx
server {
    listen 80;
    server_name sxyz.blog www.sxyz.blog att.sxyz.blog;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }
}
```



## 04. 获取证书

```bash
wget https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/www/challenges/ > ./signed.crt
```



## 05. 使用证书

```bash
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
```

Nginx https 配置：

```nginx
server {
    listen 443;
    server_name sxyz.blog www.sxyz.blog att.sxyz.blog;

    ssl on;
    ssl_certificate /opt/cert/chained.pem;
    ssl_certificate_key /opt/cert/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    #ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;
}
```



## 06. 配置自动更新

证书申请后有效期仅有 90 天，所以要配置自动更新证书。

```bash
# 创建脚本并添加执行权限
touch acme_tiny.sh
chmod a+x acme_tiny.sh
```

脚本内容为：

```bash
#!/bin/bash

python /opt/cert/acme_tiny.py --account-key /opt/cert/account.key --csr /opt/cert/domain.csr --acme-dir /var/www/challenges/ > /tmp/signed.crt || exit
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat /tmp/signed.crt intermediate.pem > /opt/cert/pchained.pem

# 重新读入配置
/usr/local/nginx/sbin/nginx -s reload
```

添加到 crontab 中：

```bash
crontab -e
0 0 1 * * cd /opt/cert/ && ./renew_cert.sh 2>> /var/log/acme_tiny.log
```

