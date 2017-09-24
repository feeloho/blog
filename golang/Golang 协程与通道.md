# Golang 协程与通道

这篇文章记录了我学习 Go 语言中的协程与通道的一些笔记，是片断性的。



## 01. 无缓冲通道

```go
ch := make(chan int)
```

无缓冲通道只能包含 1 个元素，读和写都是阻塞的。



## 02. 缓冲通道

```go
ch := make(chan int, 5)
```

有缓冲的通道可以包含指定个数的元素，读和写都是异步的，不会造成阻塞。



## 03. 死锁

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)
    ch <- 10
    go test(ch)
    time.Sleep(1e9)
}

func test(ch chan int) {
    fmt.Println(<-ch)
}
```

以上代码会出现死锁的问题，因为所有协程都处于休眠状态，向 ch 通道中写入元素后没有机会被读出。修改代码如下：

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)
    go test(ch)
    ch <- 10
    time.Sleep(1e9)
}

func test(ch chan int) {
    fmt.Println(<-ch)
}
```



## 04. 带缓冲的与不带缓冲的效能测试

可能由于异步的原因，带缓冲的平均要比不带缓冲的快一些。但是当把时间调大，缓冲数量调大后，发现带缓冲的反而更慢一些，可能是在资源调度上面浪费了一些时间。

### 无缓冲

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)
    go read(ch)
    go write(ch)
    time.Sleep(3 * 1e9)
}

func read(ch chan int) {
    for {
        fmt.Println(<-ch)
    }
}

func write(ch chan int) {
    for i := 0; ; i++ {
        ch <- i
    }
}

// ....
// 330374
// 330375
// 330376
```

### 带缓冲

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int, 1000000)
    go read(ch)
    go write(ch)
    time.Sleep(1e9)
}

func read(ch chan int) {
    for {
        fmt.Println(<-ch)
    }
}

func write(ch chan int) {
    for i := 0; ; i++ {
        ch <- i
    }
}

// ....
// 434095
// 434096
// 434097
```



## 05. 同步实现

```go
package main

import (
    "fmt"
    "time"
)

func main() {

    ch := make(chan bool)

    go func() {
        time.Sleep(2 * 1e9)
        ch <- true
    }()

    fmt.Println("2 秒后程序结束")
    <-ch

}
```

或者也可以这样：

```go
package main

import (
    "fmt"
    "time"
)

func main() {

    ch := make(chan bool)

    go func() {
        time.Sleep(1e9)
        ch <- true
    }()
    go func() {
        time.Sleep(1e9)
        ch <- true
    }()

    fmt.Println("2 秒后程序结束")
    <-ch
    <-ch

}
```



## 06. 迭代器实现

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for x := range xrange(10) {
        fmt.Println(x)
    }

    time.Sleep(1e9)
}

func xrange(len int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < len; i++ {
            ch <- i
        }
        close(ch)
    }()
    return ch
}
```



## 07. 迭代器死锁实验

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for x := range xrange(10) {
        fmt.Println(x)
    }

    time.Sleep(1e9)
}

func xrange(len int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < len; i++ {
            ch <- i
        }
    }()
    return ch
}
```

上面的代码最后会出现一个 deadlock panic，下面将试着分析一下出现这个问题的原因。

我认为，出现这个 panic 的原因是因为没有 close ，for range 循环没有收到 close 通知，认为后面应该还有其他元素。 于是进行了下一轮的迭代。(而这时已经没有其他元素了) 所以就发生了 panic。

```go
package main

import (
    "fmt"
    "time"
)

func main() {

    x := xrange(5)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)

    // Result:
    // 0
    // 1
    // 2
    // 3
    // 4

    time.Sleep(1e9)
}

func xrange(len int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < len; i++ {
            ch <- i
        }
    }()
    return ch
}
```

上面的代码模拟了只取出 5 次的情况，发现并没有发生 panic。而再多取出一个时则发生了 panic：

```go
package main

import (
    "fmt"
    "time"
)

func main() {

    x := xrange(5)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x)
    fmt.Println(<-x) // 多取出了 1 个

    // Result:
    // 0
    // 1
    // 2
    // 3
    // 4
    // fatal error: all goroutines are asleep - deadlock!

    time.Sleep(1e9)
}

func xrange(len int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < len; i++ {
            ch <- i
        }
    }()
    return ch
}
```

之后我又试着把取出的代码放到 goroutine 中，发现并没有出现 panic：

```go
package main

import (
    "fmt"
    "time"
)

func main() {

    go func() {
        x := xrange(5)
        fmt.Println(<-x)
        fmt.Println(<-x)
        fmt.Println(<-x)
        fmt.Println(<-x)
        fmt.Println(<-x)
        fmt.Println(<-x) // 多取出了 1 个
    }()

    // Result:
    // 0
    // 1
    // 2
    // 3
    // 4

    time.Sleep(1e9)
}

func xrange(len int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < len; i++ {
            ch <- i
        }
    }()
    return ch
}
```

此时我认为，是因为 goroutine 的存在(这时 xrange 函数的 goroutine 已经处于休眠状态了)，使得最后一个 `<-x` 在一直等待新元素加入。

针对以上总结：

1. 当不再 send 时，记得 close channel；
2. channel 如果在 goroutine 里用，即使代码逻辑有问题，发生 panic 的几率也小一些；
3. channel 就是给不同 goroutine 做消息共享使用的，所以还是尽量在 goroutine 里用吧。


