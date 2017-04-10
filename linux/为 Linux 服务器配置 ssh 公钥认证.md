# 为 Linux 服务器配置 ssh 公钥认证

每次连服务器都要输入一遍密码，非常麻烦。所以可以使用 ssh 公钥来解决总要去输密码的痛苦。

其实一般情况下，购买主机在开通服务前可以添加 ssh 公钥的。



## 01. 创建ssh-key

如果已经创建过了，那么可以忽略此过程。

```bash
$ ssh-keygen -t rsa
```

之后根据提示录入一些信息即可。

创建好后会在 `~/.ssh` 目录里有 2 个文件，分别是 `id_rsa`  `id_rsa.pub` ，其中 `id_rsa.pub` 文件所存放的就是我的公钥了。



## 02. 添加公钥到服务器

### 方法1 自动完成

```bash
$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@ip
```

执行后会自动添加公钥到远端服务器中。



### 方法2 手动添加

首先，先把刚才创建好的公钥内容复制出来

```bash
$ vi ~/.ssh/id_rsa.pub
```

连上远程服务器

```bash
# ssh root@ip
```

然后把刚才复制的公钥重定向到 `~/.ssh` 目录的 `authorized_keys` 文件里。

```bash
# echo '刚才copy的内容' >> ~/.ssh/authorized_keys
```

之后修改一下权限

```bash
# chmod 600 ~/.ssh/authorized_keys
```



## 03. 修改配置文件

添加好公钥后，还要让 ssh 支持公钥认证的方式。

使用 vi 打开配置文件

```bash
# vi /etc/ssh/sshd_config
```

查找 `PubkeyAuthentication` 所在的行，把前面 `#` 去掉，确保 `PubkeyAuthentication` 是 `yes` 。

最后重启 ssh 服务

```bash
# /etc/init.d/sshd restart
```

