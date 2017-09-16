# Golang 项目目录结构

项目的目录结构会影响整个项目的文件排布甚至是整个项目的文件架构。所以在开始编写代码之前要先明确项目的目录如何去安排。其它语言可能对项目结构没有什么要求，相对更加自由一些，而 Go 语言做了这方面的规定。这样虽然失去了自由，但是却得到了目录结构的一致性。



## 01. 环境变量

在安装 Go 语言时，会让你配置几个环境变量：

* **GOROOT：**go 的安装位置
* **GOPATH：**go 的工作目录，即项目目录
* **GOBIN：**`go install` 编译后的二进制文件存储路径，默认为 `$GOPATH/bin` 目录

可以通过 `go env` 查看到它们的值。



## 02. 目录结构

假设 GOPATH 配置路径为 `~/test` ，GOBIN 配置为空，那么 test 的初始目录结构应该是这样的：

```
test
├── bin  # 存放编译后的二进制文件
├── pkg  # 存放编译后的包文件
└── src  # 存放源代码
```

接下来进到 `src` 目录创建一个 `main.go` 的主程序文件：

```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("hello yazi!")
}
```

执行 `go run main.go` 可以看到 "hello yazi!" 的打印结果。现在的目录结构是这样的：

```
test
├── bin
├── pkg
└── src
    └── main.go
```



## 03. 自定义 package

我们自定义的包文件同样也放到 `src` 目录里面。首先先创建个名为 `yazi` 的目录，并在里面创建一个 `yazi.go` 的文件，其内容为：

```go
package yazi

func ReturnVal() string {
    return "hello yazi!!"
}
```

现在的目录结构是这样的：

```
test
├── bin
├── pkg
└── src
    ├── main.go
    └── yazi
        └── yazi.go
```

修改 main.go 的文件内容：

```go
package main

import (
    "fmt"
    "yazi"
)

func main() {
    fmt.Println(yazi.ReturnVal())
}
```

这时执行 `go run main.go` 会打印出 "hello yazi!!" 证明刚才编写的包成功引入进来了。

可以使用 `go install yazi` 来编译并安装 yazi 这个包，执行后的目录结构如下：

```
test
├── bin
├── pkg  # 里面多了些东西
│   └── darwin_amd64
│       └── yazi.a
└── src
    ├── main.go
    └── yazi
        └── yazi.go
```



## 04. 包的命名

从上面可以看出，定义了一个叫 `yazi` 的包，并且创建了 `src/yazi` 这个目录，在此目录中又有 `yazi.go` 这个文件。下面将做一些实验性的行为。

**1. 修改目录名与文件名：**

> 将 yazi 目录与 yazi.go 文件重新命名为 yazilalala 与 yazilalala.go ，之后执行 `go run main.go` 发现抛出错误。
>
> 将 main.go 中的 `import "yazi"` 修改为 `import "yazilalala"` ，再次执行发现问题解除。

**2. 修改 package 名：**

> 将 yazi.go 文件中的 `package yazi` 修改为 `package yazilalala` ，之后执行 `go run main.go` 发现抛出错误。
>
> 将 main.go 中的 `yazi.ReturnVal()` 修改为 `yazilalala.ReturnVal()` ，再次执行发现问题解除。

**3. 修改文件名：**

> 将 yazi.go 文件重新命名为 `abc.go` ，之后执行 `go run main.go` 成功打印出结果。



**结论：** 

* 包的目录名会影响 import 操作
* 包的 package 名实际上是包的名字
* 修改目录中的文件名不会影响到包的正常工作


