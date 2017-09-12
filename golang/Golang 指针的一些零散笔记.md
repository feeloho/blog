# Golang 指针的一些零散笔记

最近学习了一下 Go 语言，发现里面很多东西都和其他语言都不太一样。但是我却很喜欢这门语言，可能是他比较简单吧？比较符合我胃口。下面是学习过程中对 Go 中的指针部分做的一些片段性的笔记。



## 01. 数组指针

```go
package main

import (
    "fmt"
)

func main() {
    // 一个 [3]int 型数组
    arr := [...]int{1, 2, 3}

    p := &arr // 一个指向 [3]int 型数组的指针
    test(p)   // 传递过去的是 arr 的引用

    fmt.Println(arr) // [1 233 3]
}

func test(p *[3]int) {
    fmt.Println(p[0]) // 1
    p[1] = 233
}
```



## 02. 数组与切片的传递

Go 里面的切片是长度可变的数组，但更准确来说是指向数组这一原始类型的指针变量，所以切片是个引用值。当用户数据大于其所指向的区块范围时，这时会开辟并指向到一个新的原先的两倍的内存空间。

```go
package main

import (
    "fmt"
)

func main() {
    // 值传递
    arr := [...]int{1, 2, 3}
    fmt.Printf("%p ---> ", &arr)
    test1(arr)

    // 引用传递
    sli := arr[:]
    fmt.Printf("%p ---> ", sli)
    test2(sli)

    // Result:
    // 0xc4200161c0 ---> 0xc420016200
    // 0xc4200161c0 ---> 0xc4200161c0
}

func test1(a [3]int) {
    fmt.Printf("%p\n", &a)
}

func test2(s []int) {
    fmt.Printf("%p\n", s)
}
```



## 03. 使用 new() 创建数组

`arr1 := [3]int{1, 2, 3}` 是建立了个叫 arr1 的 [3]int 型数组,

`arr2 := new([3]int)` 是分配了个 [3]int 型的内存空间并将 arr2 指向了它，等同于 `arr2 := &[3]int{}`

```go
package main

import (
    "fmt"
)

func main() {
    arr1 := [3]int{1, 2, 3}
    arr2 := new([3]int)

    fmt.Println(arr1, arr2) // [1 2 3] &[0 0 0]

    test1(&arr1)
    test2(arr2) // 使用 new 创建的变量本身就是一个指针，不需要加 & 符号

    fmt.Println(arr1, arr2) // [233 2 3] &[233 0 0]
}

func test1(a *[3]int) {
    a[0] = 233
}

func test2(a *[3]int) {
    a[0] = 233
}
```



## 04. new 与 make 的区别

1. new 是用来分配内存的，而 make 是用来初始化的；
2. new(T) 返回一个指向 T 类型的零值指针地址，make(T) 返回一个类型为 T 的初始值；
3. make 只适用于 slice map channel 类型。

```go
package main

import "fmt"

func main() {
    test2()
}

func test1() {
    arr := [...]int{1, 2, 3}
    sli := arr[:]
    fmt.Printf("%p ---> %p\n", &arr, sli)

    // 0xc4200161c0 ---> 0xc4200161c0
    // 证实: 切片是指向数组的指针
}

func test2() {
    arr := [...]int{1, 2, 3}

    sli1 := arr[:]
    fmt.Println(sli1)

    sli2 := new([]int)
    *sli2 = arr[:]
    fmt.Println(*sli2)

    fmt.Printf("%p ---> %p ---> %p\n", &arr, sli1, sli2)

    // 0xc4200161c0 ---> 0xc4200161c0 ---> 0xc42000a080
    // 证实: sli2 得到了新分配的内存空间
}
```

