# 线程问题

## 单线程
单线程主要表现在顺序执行和顺序阻塞上，如下下图所示，此时handleConn在一段时间内只能处理一个请求,请其他请求过来的时候，只有进行等待。

示例1：是一个顺序执行的时钟服务器，它会每隔一秒钟将当前时间写到客户端：
```go
package main

import (
    "io"
    "log"
    "net"
    "time"
)

func main() {
    listener, err := net.Listen("tcp", "localhost:8000")
    if err != nil {
        log.Fatal(err)
    }

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Print(err) // e.g., connection aborted
            continue
        }
        handleConn(conn) // handle one connection at a time
    }
}

func handleConn(c net.Conn) {
    defer c.Close()
    for {
        _, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
        if err != nil {
            return // e.g., client disconnected
        }
        time.Sleep(1 * time.Second)
    }
}
```

如果此时两个客户端同时调用，则返回的结果如下，可以看出此时处于独占式的，仅当客户端1停止后，客户端才能继续进行。第二个客户端必须等待第一个客户端完成工作，这样服务端才能继续向后执行；因为我们这里的服务器程序同一时间只能处理一个客户端连接。
```
$ ./netcat1
13:58:54                               $ ./netcat1
13:58:55
13:58:56
^C
                                       13:58:57
                                       13:58:58
                                       13:58:59
                                       ^C
```

## goroutine 并发
在Go语言中，每一个并发的执行单元叫作一个goroutine。设想这里的一个程序有两个函数，一个函数做计算，另一个输出结果，假设两个函数没有相互之间的调用关系。一个线性的程序会先调用其中的一个函数，然后再调用另一个。如果程序中包含多个goroutine，对两个函数的调用则可能发生在同一时刻。

示例2：一个spinning等待打印和菲波那契并行运行的例子
```go
func main() {
    go spinner(100 * time.Millisecond)
    const n = 45
    fibN := fib(n) // slow
    fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {
    for {
        for _, r := range `-\|/` {
            fmt.Printf("\r%c", r)
            time.Sleep(delay)
        }
    }
}

func fib(x int) int {
    if x < 2 {
        return x
    }
    return fib(x-1) + fib(x-2)
}
```
动画显示了几秒之后，fib(45)的调用成功地返回，并且打印结果：
```
Fibonacci(45) = 1134903170
```
然后主函数返回。主函数返回时，所有的goroutine都会被直接打断，程序退出。除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个goroutine来打断另一个的执行，但是之后可以看到一种方式来实现这个目的，通过goroutine之间的通信来让一个goroutine请求其它的goroutine，并让被请求的goroutine自行结束执行。

留意一下这里的两个独立的单元是如何进行组合的，spinning和菲波那契的计算。分别在独立的函数中，但两个函数会同时执行。

我们对示例1服务端程序做一点小改动，使其支持并发：在handleConn函数调用的地方增加go关键字，让每一次handleConn的调用都进入一个独立的goroutine。

示例3：是一个并发执行的时钟服务器，它会每隔一秒钟将当前时间写到客户端：
```go
for {
    conn, err := listener.Accept()
    if err != nil {
        log.Print(err) // e.g., connection aborted
        continue
    }
    go handleConn(conn) // handle connections concurrently
}
````
此时返回的结果
```
$ ./netcat1
14:02:54                               $ ./netcat1
14:02:55                               14:02:55
14:02:56                               14:02:56
14:02:57                               ^C
14:02:58
14:02:59                               $ ./netcat1
14:03:00                               14:03:00
14:03:01                               14:03:01
```

## 并发通信channels

channels解决了两个goroutine程序之间的调用,个channel是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。每个channel都有一个特殊的类型，也就是channels可发送数据的类型。一个可以发送int类型数据的channel一般写为chan int。
```
ch := make(chan int) // ch has type 'chan int'
```
和map类似，channel也对应一个make创建的底层数据结构的引用。当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。和其它的引用类型一样，channel的零值也是nil。

两个相同类型的channel可以使用==运算符比较。如果两个channel引用的是相同的对象，那么比较的结果为真。一个channel也可以和nil进行比较。

一个channel有发送和接受两个主要操作，都是通信行为。一个发送语句将一个值从一个goroutine通过channel发送到另一个执行接收操作的goroutine。发送和接收两个操作都使用<-运算符。在发送语句中，<-运算符分割channel和要发送的值。在接收语句中，<-运算符写在channel对象之前。一个不使用接收结果的接收操作也是合法的。
```
ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```

注意：使用时注意对channel的关闭和回收

### channels类型
```go
c1:=make(chan int)        无缓冲
c2:=make(chan int,1)      有缓冲
c1<-1  
```
无缓冲的 不仅仅是 向 c1 通道放 1 而是 一直要有别的携程 <-c1 接手了 这个参数，那么c1<-1才会继续下去，不然就一直阻塞着
而 c2<-1 则不会阻塞，因为缓冲大小是1 （其实是缓冲大小为0）只有当 放第二个值的时候 第一个还没被人拿走，这时候才会阻塞。
打个比喻

> 无缓冲的  就是一个送信人去你家门口送信 ，你不在家 他不走，你一定要接下信，他才会走。
无缓冲保证信能到你手上

> 有缓冲的 就是一个送信人去你家仍到你家的信箱 转身就走 ，除非你的信箱满了 他必须等信箱空下来。
有缓冲的 保证 信能进你家的邮箱

将一个带缓存的channel当作同一个goroutine中的队列使用是错误的用法。Channel和goroutine的调度器机制是紧密相连的，如果没有其他goroutine从channel接收，发送者——或许是整个程序——将会面临永远阻塞的风险。

### 单项的channels

示例4：一个单项channels的例子
```go
func counter(out chan<- int) {
    for x := 0; x < 100; x++ {
        out <- x
    }
    close(out)
}

func squarer(out chan<- int, in <-chan int) {
    for v := range in {
        out <- v * v
    }
    close(out)
}

func printer(in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
}

func main() {
    naturals := make(chan int)
    squares := make(chan int)
    go counter(naturals)
    go squarer(squares, naturals)
    printer(squares)
}
````

调用counter（naturals）时，naturals的类型将隐式地从chan int转换成chan<- int。调用printer(squares)也会导致相似的隐式转换，这一次是转换为<-chan int类型只接收型的channel。任何双向channel向单向channel变量的赋值操作都将导致该隐式转换。

### 串联的Channels（Pipeline）

Channels也可以用于将多个goroutine连接在一起，一个Channel的输出作为下一个Channel的输入。这种串联的Channels就是所谓的管道（pipeline）。第一个goroutine是一个计数器，用于生成0、1、2、……形式的整数序列，然后通过channel将该整数序列发送给第二个goroutine；第二个goroutine是一个求平方的程序，对收到的每个整数求平方，然后将平方后的结果通过第二个channel发送给第三个goroutine；第三个goroutine是一个打印程序，打印收到的每个整数。
![avatar](http://go.wuhaolin.cn/gopl/images/ch8-01.png)

```go
func main() {
    naturals := make(chan int)
    squares := make(chan int)

    // Counter
    go func() {
        for x := 0; x < 100; x++ {
            naturals <- x
        }
        close(naturals)
    }()

    // Squarer
    go func() {
        for x := range naturals {
            squares <- x * x
        }
        close(squares)
    }()

    // Printer (in main goroutine)
    for x := range squares {
        fmt.Println(x)
    }
}
```
## select的多路复用

当多个channel我们无法做到从每一个channel中接收信息，如果我们这么做的话，如果第一个channel中没有事件发过来那么程序就会立刻被阻塞，这样我们就无法收到第二个channel中发过来的事件。这时候我们需要多路复用(multiplex)这些操作了，为了便于理解，我们首先给出一个代码片段：
```go
select {
case v1 := <-c1:
    fmt.Printf("received %v from c1\n", v1)
case v2 := <-c2:
    fmt.Printf("received %v from c2\n", v1)
case c3 <- 23:
    fmt.Printf("sent %v to c3\n", 23)
default:
    fmt.Printf("no one was ready to communicate\n")
}
```
上面这段代码中，select 语句有四个 case 子语句，前两个是 receive 操作，第三个是 send 操作，最后一个是默认操作。代码执行到 select 时，case 语句会按照源代码的顺序被评估，且只评估一次，评估的结果会出现下面这几种情况：  
除 default 外，如果只有一个 case 语句评估通过，那么就执行这个case里的语句；  
除 default 外，如果有多个 case 语句评估通过，那么通过伪随机的方式随机选一个；  
如果 default 外的 case 语句都没有通过评估，那么执行 default 里的语句；  
如果没有 default，那么 代码块会被阻塞，指导有一个 case 通过评估；否则一直阻塞
