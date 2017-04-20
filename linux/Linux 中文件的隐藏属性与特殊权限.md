# Linux 中文件的隐藏属性与特殊权限

## 01. 隐藏属性

可以使用 `chattr` 命令设置文件的隐藏属性，使用命令 `lsattr` 列出文件的隐藏属性。

### chattr 

```bash
chattr [+-=][ASacdistu] 文件或目录名称
选项与参数：
+   ：添加某一个特殊参数，其他原本存在参数则不动。
-   ：移除某一个特殊参数，其他原本存在参数则不动。
=   ：配置一定，且仅有后面接的参数

A   ：当配置了 A 这个属性时，若你有存取此文件(或目录)时，他的存取时间 atime
      将不会被修改，可避免I/O较慢的机器过度的存取磁碟。这对速度较慢的计算机有帮助
S   ：一般文件是非同步写入磁碟的(原理请参考第五章sync的说明)，如果加上 S 这个
      属性时，当你进行任何文件的修改，该更动会『同步』写入磁碟中。
a   ：当配置 a 之后，这个文件将只能添加数据，而不能删除也不能修改数据，只有root 
      才能配置这个属性。 
c   ：这个属性配置之后，将会自动的将此文件『压缩』，在读取的时候将会自动解压缩，
      但是在储存的时候，将会先进行压缩后再储存(看来对於大文件似乎蛮有用的！)
d   ：当 dump 程序被运行的时候，配置 d 属性将可使该文件(或目录)不会被 dump 备份
i   ：这个 i 可就很厉害了！他可以让一个文件『不能被删除、改名、配置连结也无法
      写入或新增数据！』对於系统安全性有相当大的助益！只有 root 能配置此属性
s   ：当文件配置了 s 属性时，如果这个文件被删除，他将会被完全的移除出这个硬盘
      空间，所以如果误删了，完全无法救回来了喔！
u   ：与 s 相反的，当使用 u 来配置文件时，如果该文件被删除了，则数据内容其实还
      存在磁碟中，可以使用来救援该文件喔！

注意 ：属性配置常见的是 a 与 i 的配置值，而且很多配置值必须要身为 root 才能配置
```

给文件添加隐藏属性：

```bash
$ touch test

# 禁止修改文件
$ sudo chattr +i test

# 尝试改变内容，提示没有权限
$ echo '123' > test
-bash: test: Permission denied
```

### lsattr   

```bash
lsattr [-adR] 文件或目录
选项与参数：
-a ：将隐藏档的属性也秀出来；
-d ：如果接的是目录，仅列出目录本身的属性而非目录内的档名；
-R ：连同子目录的数据也一并列出来！
```

查看刚才添加隐藏权限的文件：

```bash
$ lsattr test
----i--------e-- test
```



## 02. 特殊权限

然后说下文件的 3 个特殊权限：**SUID**  **SGID** **SBIT**。



### SUID  

这个是给 可执行的二进制文件 加的。是加在 ugo 权限里 `o` 上面的。加上之后，进程在执行的过程中使用的不是 运行者 的权限，而是 文件拥有者 的权限。

例如 `passwd` 这个程序，他执行的时候就是使用的 文件拥有者 的权限，因为他要改变 `/etc/shadow` 文件的内容，普通用户根本无法改变，除非是超级用户，而 `/usr/bin/passwd` 的拥有者就是 `root` 。

```bash
$ ls -l /usr/bin/passwd
# 可以发现 拥有者权限的最后一位是s
-rwsr-xr-x 1 root root 40756 Feb 25 06:49 /usr/bin/passwd
```

然后做个试验尝试下

```bash
$ cp /bin/ls .
$ mkdir test
$ chmod 000 test/
$ sudo chown root:root ls
$ sudo chmod u+xs ls
$ ls -l
total 2
-rwsr--r-- 1 root root 96364 Apr 20 15:18 ls
d--------- 2 pi   pi    4096 Apr 20 15:18 test

# 使用 /bin/ls 提示没权限
$ ls test/
ls: cannot open directory test/: Permission denied

# 使用刚才copy过来的ls可以正常列出文件
$ ./ls test/
```

上面是做了一个权限为 `d---------` 的目录，之后复制 `/bin/ls` 出来给他加上 `s` 的权限之后，可以正常的使用 `root` 用户的权限来列出目录的文件列表了。



### SGID  

SGID 是加在 ugo 权限中的 `g` 里头的。支持给 文件 和 目录 添加。

**加给文件：**

与 SUID 相似，SGID 加上之后使用的是 `群组` 的权限去执行的。

**加给目录：**

如果在这个目录里面创建新文件，则默认群组为该目录的群组。

```bash
$ mkdir test
$ sudo chmod g+s test/
$ sudo chgrp root test/

$ ls -ld test/
drwxr-sr-x 2 pi root 4096 Apr 20 15:53 test/ # 群组是 root

$ cd test/
~/test $ touch 1
~/test $ ls -l

total 0
-rw-r--r-- 1 pi root 0 Apr 20 15:55 1 # 群组变成了 root
```



### SBIT 

SBIT 只能加给 目录，当这个目录加入此属性后在里面创建的文件只有文件 拥有者 与 root 具有 移动/删除/更名 的权限。

```bash
~ $ su - root
Password:

~# cd ~yazi/

/home/yazi# mkdir test
/home/yazi# chmod +wt test/
/home/yazi# ls -ld test/
drwxr-xr-t 2 root root 4096 Apr 20 19:00 test/

/home/yazi# touch test/1
/home/yazi# chmod 777 test/1

/home/yazi# exit
logout

~ $ cd test/
~/test $ ls -l
total 0
-rwxrwxrwx 1 root root 0 Apr 20 19:01 1

# 权限是777而提示无法删除
~/test $ rm 1
rm: cannot remove ‘1’: Permission denied
```



参考链接：

http://cn.linux.vbird.org/linux_basic/0220filemanager_4.php

