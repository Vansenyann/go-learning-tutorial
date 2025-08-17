# 并发

并行 vs 并发 

并行方案就是在处理器核数充足的情况下启动多个单线程应用的实例，这样每个实例“运行”在一个核上（如图中的CPU核1～CPU核N），尽可能多地利用多核计算资源

并发就是重新做应用结构设计，即将应用分解成多个在基本执行单元

## 1. goroutine的调度原理

goroutine是由Go运行时管理的用户层轻量级线程。相较于操作系统线程，goroutine的资源占用和使用代价都要小得多。我们可以创建几十个、几百个甚至成千上万个goroutine，Go的运行时负责对goroutine进行管理。

所谓的管理就是“调度”。简单地说，调度就是决定何时哪个goroutine将获得资源开始执行，哪个goroutine应该停止执行让出资源，哪个goroutine应该被唤醒恢复执行等。

由于一个goroutine占用资源很少，一个Go程序中可以创建成千上万个并发的goroutine。而将这些goroutine按照一定算法放到CPU上执行的程序就称为goroutine调度器（goroutine scheduler）。一个Go程序对于操作系统来说只是一个用户层程序，操作系统眼中只有线程，goroutine的调度全要靠Go自己完成。

## 1.1 调度模型

### 1.1.1 G-M模型

在这个调度器中，每个goroutine对应于运行时中的一个抽象结构—G（goroutine），而被视作“物理CPU”的操作系统线程则被抽象为另一个结构—M（machine）。这个模型实现起来比较简单且能正常工作，但是却存在着诸多问题:

1. 单一全局互斥锁（Sched.Lock）和集中状态存储的存在导致所有goroutine相关操作（如创建、重新调度等）都要上锁。
2. goroutine传递问题：经常在M之间传递“可运行”的goroutine会导致调度延迟增大，带来额外的性能损耗。
3. 每个M都做内存缓存，导致内存占用过高，数据局部性较差。因系统调用（syscall）而形成的频繁的工作线程阻塞和解除阻塞会带来额外的性能损耗

### 1.1.2 G-P-M模型

向G-M模型中增加了一个P，使得goroutine调度器具有很好的伸缩性。

`<img src="images/GPM.png">`

1. P是一个“逻辑处理器”，每个G要想真正运行起来，首先需要被分配一个P，即进入P的本地运行队列（local runq）中，这里暂忽略全局运行队列（global runq）那个环节。
2. 对于G来说，P就是运行它的“CPU”，可以说在G的眼里只有P。但从goroutine调度器的视角来看，真正的“CPU”是M，只有将P和M绑定才能让P的本地运行队列中的G真正运行起来。这样的P与M的关系就好比Linux操作系统调度层面用户线程（user thread）与内核线程（kernel thread）的对应关系：多对多（N:M）。

G：代表goroutine，存储了goroutine的执行栈信息、goroutine状态及goroutine的任务函数等。另外G对象是可以重用的。

P：代表逻辑processor，P的数量决定了系统内最大可并行的G的数量（前提：系统的物理CPU核数>=P的数量）。P中最有用的是其拥有的各种G对象队列、链表、一些缓存和状态。

M：M代表着真正的执行计算资源。在绑定有效的P后，进入一个调度循环；而调度循环的机制大致是从各种队列、P的本地运行队列中获取G，切换到G的执行栈上并执行G的函数，调用goexit做清理工作并回到M。如此反复。M并不保留G状态，这是G可以跨M调度的基础。

**G被抢占调度**

与操作系统按时间片调度线程不同，Go中并没有时间片的概念。如果某个G没有进行系统调用（syscall）、没有进行I/O操作、没有阻塞在一个channel操作上，那么M是如何让G停下来并调度下一个可运行的G的呢？答案是：G是被抢占调度的。

前面说过，除非极端的无限循环或死循环，否则只要G调用函数，Go运行时就有了抢占G的机会。在Go程序启动时，运行时会启动一个名为sysmon的M（一般称为监控线程），该M的特殊之处在于它无须绑定P即可运行（以g0这个G的形式）。该M在整个Go程序的运行过程中至关重要，参见下面代码：

```go
//$GOROOT/src/runtime/proc.go

// The main goroutine.
func main() {
     ...
    systemstack(func() {
        newm(sysmon, nil)
    })
    ...
}

// 运行无须P参与
//
//go:nowritebarrierrec
func sysmon() {
    // 如果一个heap span在垃圾回收后5分钟内没有被使用
    // 就把它归还给操作系统
    scavengelimit := int64(5 * 60 * 1e9)
    ...

    if  ... {
        ...
        // 夺回被阻塞在系统调用中的P
        // 抢占长期运行的G
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }
       ...
    }
}
```

sysmon每20us~10ms启动一次，主要完成如下工作：释

1. 放闲置超过5分钟的span物理内存；
2. 如果超过2分钟没有垃圾回收，强制执行；
3. 将长时间未处理的netpoll结果添加到任务队列；
4. 向长时间运行的G任务发出抢占调度；
5. 收回因syscall长时间阻塞的P。
6. 我们看到sysmon将向长时间运行的G任务发出抢占调度，这由函数retake实施：

```go
// $GOROOT/src/runtime/proc.go

// forcePreemptNS是在一个G被抢占之前给它的时间片
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
    ...
    // 抢占运行时间过长的G
    t := int64(_p_.schedtick)
    if int64(pd.schedtick) != t {
        pd.schedtick = uint32(t)
        pd.schedwhen = now
        continue
    }
    if pd.schedwhen+forcePreemptNS > now {
        continue
    }
    preemptone(_p_)
    ...
}
```

可以看出，如果一个G任务运行超过10ms，sysmon就会认为其运行时间太久而发出抢占式调度的请求。一旦G的抢占标志位被设为true，那么在这个G下一次调用函数或方法时，运行时便可以将G抢占并移出运行状态，放入P的本地运行队列中（如果P的本地运行队列已满，那么将放在全局运行队列中），等待下一次被调度

#### 1.1.2.1 channel阻塞或网络I/O情况下的调度

如果G被阻塞在某个channel操作或网络I/O操作上，那么G会被放置到某个等待队列中，而M会尝试运行P的下一个可运行的G。如果此时P没有可运行的G供M运行，那么M将解绑P，并进入挂起状态。当I/O操作完成或channel操作完成，在等待队列中的G会被唤醒，标记为runnable（可运行），并被放入某个P的队列中，绑定一个M后继续执行。

#### 1.1.2.2 系统调用阻塞情况下的调度

如果G被阻塞在某个系统调用上，那么不仅G会阻塞，执行该G的M也会解绑P（实质是被sysmon抢走了），与G一起进入阻塞状态。

如果此时有空闲的M，则P会与其绑定并继续执行其他G；

如果没有空闲的M，但仍然有其他G要执行，那么就会创建一个新M（线程）。当系统调用返回后，阻塞在该系统调用上的G会尝试获取一个可用的P，如果有可用P，之前运行该G的M将绑定P继续运行G；如果没有可用的P，那么G与M之间的关联将解除，同时G会被标记为runnable，放入全局的运行队列中，等待调度器的再次调度。

### 1.1.3 抢占式调度

G-P-M模型的实现是goroutine调度器的一大进步，但调度器仍然有一个头疼的问题，那就是不支持抢占式调度，这导致一旦某个G中出现死循环的代码逻辑，那么G将永久占用分配给它的P和M，而位于同一个P中的其他G将得不到调度，出现“饿死”的情况。

更为严重的是，当只有一个P（GOMAXPROCS=1）时，整个Go程序中的其他G都将“饿死”。于是Dmitry Vyukov又提出了“Go抢占式调度器设计”（Go Preemptive Scheduler Design）

这个抢占式调度的原理是在每个函数或方法的入口加上一段额外的代码，让运行时有机会检查是否需要执行抢占调度。这种协作式抢占调度的解决方案只是局部解决了“饿死”问题，对于没有函数调用而是纯算法循环计算的G，goroutine调度器依然无法抢占

# 2. GO并发模型

传统的编程语言（如C++、Java、Python等）并非为并发而生的，因此它们面对并发的逻辑多是基于操作系统的线程。其并发的执行单元（线程）之间的通信利用的也是操作系统提供的线程或进程间通信的原语，比如共享内存、信号、管道、消息队列、套接字等。

在这些通信原语中，使用最多、最广泛同时也最高效的是结合了线程同步原语（比如锁以及更为低级的原子操作）的共享内存方式

传统语言的并发模型是基于共享内存的模型:

`<img src="images/传统并发.png">`

Go语言从设计伊始就将解决上述传统并发模型的问题作为目标之一，并在新并发模型设计中借鉴了著名计算机科学家Tony Hoare提出的CSP（Communicating Sequential Process，通信顺序进程）模型

Tony Hoare的CSP模型旨在简化并发程序的编写，让并发程序的编写与编写顺序程序一样简单。

Tony Hoare认为输入/输出应该是基本的编程原语，数据处理逻辑（CSP中的P）仅需调用输入原语获取数据，顺序处理数据，并将结果数据通过输出原语输出即可。

因此，在Tony Hoare眼中，一个符合CSP模型的并发程序应该是一组通过输入/输出原语连接起来的P的集合。

从这个角度来看，CSP理论不仅是一个并发参考模型，也是一种并发程序的程序组织方法。其组合思想与Go的设计哲学不谋而合。

CSP理论中的P（Process，进程）是个抽象概念，它代表任何顺序处理逻辑的封装，它获取输入数据（或从其他P的输出获取），并生产可以被其他P消费的输出数据。

P并不一定与操作系统的进程或线程画等号。在Go中，与Process对应的是goroutine，但Go语言中goroutine的执行逻辑并不一定是顺序的，goroutine也可以创建其他goroutine以并发地完成工作。

为了实现CSP模型中的输入/输出原语，Go引入了goroutine（P）之间的通信原语channel。goroutine可以从channel获取输入数据，再将处理后得到的结果数据通过channel输出。

通过channel将goroutine（P）组合与连接在一起，这使得设计和编写大型并发系统变得更为简单和清晰，我们无须再为那些传统共享内存并发模型中的问题而伤脑筋了

tips:

从程序的整体结构来看，Go始终推荐以CSP模型风格构建并发程序，尤其是在复杂的业务层面。这将提升程序逻辑的清晰度，大大降低并发设计的复杂性，并让程序更具可读性和可维护性；

对于局部情况，比如涉及性能敏感的区域或需要保护的结构体数据，可以使用更为高效的低级同步原语（如sync.Mutex），以保证goroutine对数据的同步访问

# 3. GO的并发模式

在语言层面，Go针对CSP模型提供了三种并发原语。

goroutine：对应CSP模型中的P，封装了数据的处理逻辑，是Go运行时调度的基本执行单元。

channel：对应CSP模型中的输入/输出原语，用于goroutine之间的通信和同步。

select：用于应对多路输入/输出，可以让goroutine同时协调处理多个channel操作。

组合方式:

## 3.1 创建模式

1. 使用go关键字+函数/方法创建goroutine：
2. 但在稍复杂一些的并发程序中，需要考虑通过CSP模型输入/输出原语的承载体channel在goroutine之间建立联系。为了满足这一需求，我们通常使用下面的方式来创建goroutine：

```go
type T struct {...}

func spawn(f func()) chan T {
    c := make(chan T)
    go func() {
        // 使用channel变量c(通过闭包方式)与调用spawn的goroutine通信
        ...
        f()
        ...
    }()
  
    return c
}

func main() {
    c := spawn(func(){})
    // 使用channel变量c与新创建的goroutine通信
}
```

以上方式在内部创建一个goroutine并返回一个channel类型变量的函数，这是Go中最常见的goroutine创建模式。

spawn函数创建的新goroutine与调用spawn函数的goroutine之间通过一个channel建立起了联系：两个goroutine可以通过这个channel进行通信。

spawn函数的实现得益于channel作为Go语言一等公民（first-class citizen）的存在：channel可以像变量一样被初始化、传递和赋值。上面例子中的spawn只返回了一个channel变量，大家可以根据需要自行定义返回的channel个数和用途。

## 3.2 退出模式

在多数情况下，我们无须考虑对goroutine的退出进行控制：goroutine的执行函数返回，即意味着goroutine退出。

但一些常驻的后台服务程序可能会对goroutine有着优雅退出的要求，在这里我们就分类说明一下goroutine的几种退出模式。

### 3.2.1 分离模式

分离模式是使用最为广泛的goroutine退出模式。对于分离的goroutine，创建它的goroutine不需要关心它的退出，这类goroutine在启动后即与其创建者彻底分离，其生命周期与其执行的主函数相关，函数返回即goroutine退出。这类goroutine有两个常见用途。

1. 一次性任务：顾名思义，新创建的goroutine用来执行一个简单的任务，执行后即退出
2. 常驻后台执行一些特定任务，如监视（monitor）、观察（watch）等。其实现通常采用for {...}或for { select{...} }代码段形式，并多以定时器（timer）或事件（event）驱动执行

### 3.2.2 join模式

在线程模型中，父线程可以通过pthread_join来等待子线程结束并获取子线程的结束状态。在Go中，我们有时候也有类似的需求：goroutine的创建者需要等待新goroutine结束

## 3.3 管道模式

### 3.3.1 扇出模式

在某个处理环节，多个功能相同的goroutine从同一个channel读取数据并处理，直到该channel关闭，这种情况被称为“扇出”（fan-out）。使用扇出模式可以在一组goroutine中均衡分配工作量，从而更均衡地利用CPU。

### 3.3.2 扇入模式

在某个处理环节，处理程序面对不止一个输入channel。我们把所有输入channel的数据汇聚到一个统一的输入channel，然后处理程序再从这个channel中读取数据并处理，直到该channel因所有输入channel关闭而关闭。这种情况被称为“扇入”（fan-in

## 3.4 超时与取消模式

编写一个从气象数据服务中心获取气象信息的客户端。该客户端每次会并发向三个气象数据服务中心发起数据查询请求，并以最快返回的那个响应信息作为此次请求的应答返回值

```go
// chapter6/sources/go-concurrency-pattern-10.go 

type result struct {
    value string
}

func first(servers ...*httptest.Server) (result, error) {
    c := make(chan result, len(servers))
    queryFunc := func(server *httptest.Server) {
        url := server.URL
        resp, err := http.Get(url)
        if err != nil {
            log.Printf("http get error: %s\n", err)
            return
        }
        defer resp.Body.Close()
        body, _ := ioutil.ReadAll(resp.Body)
        c <- result{
            value: string(body),
        }
    }
    for _, serv := range servers {
        go queryFunc(serv)
    }
    return <-c, nil
}

func fakeWeatherServer(name string) *httptest.Server {
    return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, 
        r *http.Request) {
        log.Printf("%s receive a http request\n", name)
        time.Sleep(1 * time.Second)
        w.Write([]byte(name + ":ok"))
    }))
}

func main() {
    result, err := first(fakeWeatherServer("open-weather-1"),
        fakeWeatherServer("open-weather-2"),
        fakeWeatherServer("open-weather-3"))
    if err != nil {
        log.Println("invoke first error:", err)
        return
    }
  
    log.Println(result)
}
```

而现实中的网络情况错综复杂，远程服务的状态也不甚明朗，很可能出现服务端长时间没有响应的情况。这时为了保证良好的用户体验，我们需要对客户端的行为进行精细化控制，比如：只等待500ms，超过500ms仍然没有收到任何一个气象数据服务中心的响应，first函数就返回失败，以保证等待时间在人类的忍耐力承受范围之内。

增加了一个定时器，并通过select原语监视该定时器事件和响应channel上的事件。如果响应channel上长时间没有数据返回，则当定时器事件触发时，first函数返回超时错误：

```go
// chapter6/sources/go-concurrency-pattern-11.go
func first(servers ...*httptest.Server) (result, error) {
    c := make(chan result, len(servers))
    queryFunc := func(server *httptest.Server) {
        url := server.URL
        resp, err := http.Get(url)
        if err != nil {
            log.Printf("http get error: %s\n", err)
            return
        }
        defer resp.Body.Close()
        body, _ := ioutil.ReadAll(resp.Body)
        c <- result{
            value: string(body),
        }
    }
    for _, serv := range servers {
        go queryFunc(serv)
    }
  
    select {
    case r := <-c:
        return r, nil
    case <-time.After(500 * time.Millisecond):
        return result{}, errors.New("timeout")
    }
}
```



加上了超时模式的版本依然有一个明显的问题，那就是即便first函数因超时而返回，三个已经创建的goroutine可能依然处在向气象数据服务中心请求或等待应答状态，没有返回，也没有被回收，资源仍然在占用，即使它们的存在已经没有任何意义。一种合理的解决思路是让这三个goroutine支持取消操作。在这种情况下，我们一般使用Go的context包来实现取消模式。context包是谷歌内部关于Go的一个最佳实践，Go在1.7版本将context包引入标准库中。

在这版实现中，我们利用context.WithCancel创建了一个可以取消的context.Context变量，在每个发起查询请求的goroutine中，我们用该变量更新了request中的ctx变量，使其支持被取消。这样在first函数中，无论是成功得到某个查询goroutine的返回结果，还是超时失败返回，通过defer cancel()设定cancel函数在first函数返回前被执行，那些尚未返回的在途（on-flight）查询的goroutine都将收到cancel事件并退出（http包支持利用context.Context的超时和cancel机制

```go
// chapter6/sources/go-concurrency-pattern-12.go
type result struct {
    value string
}

func first(servers ...*httptest.Server) (result, error) {
    c := make(chan result)
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    queryFunc := func(i int, server *httptest.Server) {
        url := server.URL
        req, err := http.NewRequest("GET", url, nil)
        if err != nil {
            log.Printf("query goroutine-%d: http NewRequest error: %s\n", i, err)
            return
        }
        req = req.WithContext(ctx)
      
        log.Printf("query goroutine-%d: send request...\n", i)
        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            log.Printf("query goroutine-%d: get return error: %s\n", i, err)
            return
        }
        log.Printf("query goroutine-%d: get response\n", i)
        defer resp.Body.Close()
        body, _ := ioutil.ReadAll(resp.Body)
      
        c <- result{
            value: string(body),
        }
        return
    }
  
    for i, serv := range servers {
        go queryFunc(i, serv)
    }
  
    select {
    case r := <-c:
        return r, nil
    case <-time.After(500 * time.Millisecond):
        return result{}, errors.New("timeout")
    }
}

func fakeWeatherServer(name string, interval int) *httptest.Server {
    return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, 
        r *http.Request) {
        log.Printf("%s receive a http request\n", name)
        time.Sleep(time.Duration(interval) * time.Millisecond)
        w.Write([]byte(name + ":ok"))
    }))
}

func main() {
    result, err := first(fakeWeatherServer("open-weather-1", 200),
        fakeWeatherServer("open-weather-2", 1000),
        fakeWeatherServer("open-weather-3", 600))
    if err != nil {
        log.Println("invoke first error:", err)
        return
    }
  
    fmt.Println(result)
    time.Sleep(10 * time.Second)
}
```
