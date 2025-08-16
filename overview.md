# 概述

## 1. go语言的诞生与演进

2009年10月30日，Rob Pike在Google Techtalk上做了一次有关Go语言的演讲“The Go Programming Language”[6]，首次将Go语言公之于众。

Go语言项目在2009年11月10日正式开源，这一天也被Go官方确定为Go语言诞生日。

三位作者：

[1] Rob Pike，贝尔实验室早期成员，参与了Plan 9操作系统、C编译器以及多种语言编译器的设计和实现，UTF-8编码的发明人之一。

[2] Robert Griesemer，Java的HotSpot虚拟机和Chrome浏览器的JavaScript V8引擎的设计者之一。

[3] Ken Thompson，图灵奖得主，Unix之父，C语言的发明人之一。

## 2. go的演进

1. Go的基本语法参考了C语言，Go是“C家族语言”的一个分支；
2. Go的声明语法、包概念则受到了Pascal、Modula、Oberon的启发；
3. 并发的思想则来自受到Tony Hoare教授CSP理论[1]影响的编程语言，比如Newsqueak和Limbo。

`<img src="images\go的演进.png" width="80% height=80%>`

## 3. 设计哲学

### 3.1 追求简单，少即是多

复杂性留给了语言自身的设计和实现，简单，易用和清晰留给了开发者

1. 25个关键字
2. 内置垃圾回收，内存总是初始化为零值
3. 显式依赖，没有循环依赖
4. 首字母大小写决定可见性
5. 没有类，任何类型都可以拥有方法，没有子类型集成
6. 接口是隐式的，无须implements声明
7. 方法就是函数
8. 没有指针算术
9. 内置数组边界检查，内置并发支持

### 3.2 偏好组合，正交解耦

go语言通过组合的方式将程序的各个部分耦合在一起，而不是像OO语言一样使用类型体系，继承，显式接口实现等机制

在语言设计层面，Go提供了正交的语法元素供后续组合使用，包括：

1. Go语言无类型体系（type hierarchy），类型之间是独立的，没有子类型的概念；
2. 每个类型都可以有自己的方法集合，类型定义与方法实现是正交独立的；
3. 接口（interface）与其实现之间隐式关联；
4. 包（package）之间是相对独立的，没有子包的概念。

Go语言提供的最为直观的组合的语法元素是类型嵌入（type embedding）。通过类型嵌入，我们可以将已经实现的功能嵌入新类型中，以快速满足新类型的功能需求.

这种方式有些类似经典OO语言中的继承机制，但在原理上与其完全不同，这是一种Go设计者们精心设计的语法糖。被嵌入的类型和新类型之间没有任何关系，甚至相互完全不知道对方的存在，更没有经典OO语言中的那种父类、子类的关系以及向上、向下转型（type casting）。在通过新类型实例调用方法时，方法的匹配取决于方法名字，而不是类型。

举例1:

垂直组合：类型嵌入（struct）

在poolLocal这个结构体类型中嵌入了类型Mutex，被嵌入的Mutex类型的方法集合会被提升到外面的类型（poolLocal）中。比如，这里的poolLocal将拥有Mutex类型的Lock和Unlock方法。但在实际调用时，方法调用会被传给poolLocal中的Mutex实例。

```go
// $GOROOT/src/sync/pool.go
type poolLocal struct {
    private interface{}
    shared  []interface{}
    Mutex
    pad     [128]byte
}
```

举例2：

水平组合：通过interface将程序各个部分组合在一起

水平组合的模式有很多，一种常见的方法是通过接受interface类型参数的普通函数进行组合，例如下面的代码。

```go
// $GOROOT/src/io/ioutil/ioutil.go
func ReadAll(r io.Reader)([]byte, error)

// $GOROOT/src/io/io.go
func Copy(dst Writer, src Reader)(written int64, err error)
```

类似的水平组合模式还有wrapper，middleware等

通过在interface类型的定义中嵌入interface类型来实现接口行为的聚合，组成大接口

```go
// $GOROOT/src/io/io.go
type ReadWriter interface {
    Reader
    Writer
}
```

### 3.3 原生并发，轻量高效

硬件能力的发展方向：提高主频变成了多核，反过来影响了程序语言的设计开始面向多核，原生内置并发支持

go语言原生支持并发的设计哲学体现：

1. 轻量级协程并发模型:

   1. 传统编程语言的并发实现
      1. 程序负责创建线程,操作系统负责调度,缺点在于复杂和难于扩展
      2. 复杂1:创建容易,退出难. 创建一个线程时（比如利用pthread库）虽然参数也不少，但还可以接受。而一旦涉及线程的退出，就要考虑线程是不是分离的（detached）？是否需要父线程去通知并等待子线程退出（join）？是否需要在线程中设置取消点（cancel point）以保证进行join操作时能顺利退出？
      3. 复杂2: 并发单元间通信困难，易错：多个线程之间的通信虽然有多种机制可选，但用起来相当复杂；并且一旦涉及共享内存（shared memory），就会用到各种锁（lock），死锁便成为家常便饭
      4. 复杂3: 线程栈大小（thread stack size）的设定：是直接使用默认的，还是设置得大一些或小一些呢？
      5. 难以扩展: 虽然线程的代价比进程小了很多，但我们依然不能大量创建线程，因为不仅每个线程占用的资源不小，操作系统调度切换线程的代价也不小。对于很多网络服务程序，由于不能大量创建线程，就要在少量线程里做网络的多路复用，即使用epoll/kqueue/IoCompletionPort这套机制。即便有了libevent、libev这样的第三方库的帮忙，写起这样的程序也是很不容易的，存在大量回调（callback），会给程序员带来不小的心智负担。
   2. go放弃了传统的基于OS线程的并发模型,采用了用户轻量级线程, 也叫corountine
   3. go中叫gorouine, 一个goroutine默认分配的栈空间为2kb, 一个go程序中创建了10000个协程, 总的占用也就20MB
   4. go routine调度器:传统的基于OS的线程调度是OS完成的, go中不依赖OS, 需要go自己完成调度, 实现go程序内goroutine的公平竞争cpu资源的任务就落到了go运行时头上. goroutine调度器就是将这些goroutine按照一定算法放到cpu上执行的程序
2. 支持并发的语法元素和机制

   1. 单核年代的编程语言设计: c++, java
      1. 执行单元:线程
      2. 创建和销毁的方式: 调用os库函数或调用对象方法
      3. 并发线程间的通信: os提供的ipc(进程间通信))机制, 比如共享内存,socket,pipe, 有并发保护的全局变量
   2. go:
      1. 执行单元: goroutine
      2. 创建和销毁: go+函数调用, 函数退出就是goroutine退出
      3. 并发goroutine通信: 通过语言内置的channel传递消息或实现同步, 并通过select实现多路channel的并发控制
3. 并发 vs 并行

   1. 并发: 并发是有关结构的，它是一种将一个程序分解成多个小片段并且每个小片段都可以独立执行的程序设计方法；并发程序的小片段之间一般存在通信联系并且通过通信相互协作。
   2. 并行: 并行是有关执行的，它表示同时进行一些计算任务。
   3. 采用并发方案设计的程序在单核处理器上也是可以正常运行的（在单核上的处理性能可能不如非并发方案），并且随着处理器核数的增多，并发方案可以自然地提高处理性能，提升吞吐量。而非并发方案在处理器核数提升后，也仅能使用其中的一个核，无法自然扩展，这一切都是程序的结构所决定的。这告诉我们：并发程序的结构设计不要局限于在单核情况下处理能力的高低，而要以在多核情况下充分提升多核利用率、获得性能的自然提升为最终目的。

### 3.4 面向工程, 自带电池

1. 标准库丰富: Go在标准库中提供了各类高质量且性能优良的功能包，其中的net/http、crypto/xx、encoding/xx等包充分迎合了云原生时代关于API/RPC Web服务的构建需求
2. 工具链:
   1. 构建和运行：go build/go run
   2. 依赖包查看与获取：go list/go get/go mod xx
   3. 编辑辅助格式化：go fmt/gofmt
   4. 文档查看：go doc/godoc
   5. 单元测试/基准测试/测试覆盖率：go test
   6. 代码静态分析：go vet
   7. 性能剖析与跟踪结果查看：go tool pprof/go tool trace
   8. 升级到新Go版本API的辅助工具：go tool fix
   9. 报告Go语言bug：go bug
