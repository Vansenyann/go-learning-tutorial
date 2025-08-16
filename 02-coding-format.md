# go语言编程规范

## 1. 使用Go命名惯例对标识符进行命名

两个原则:

1. 简单且一致
2. 利用上下文辅助命名

常见的命名惯例:

1. 包
   1. 小写形式的单个单词命名
   2. 包命名可以不唯一,导入路径唯一即可. 包名定义在go.mod中, 不是文件夹的名称
   3. 包名应尽量与包导入路径（import path）的最后一个路径分段保持一致
2. 变量, 类型,函数, 方法
   1. 变量: 包级别的变量和局部变量(函数或方法内的变量)
   2. 驼峰命名法
   3. 变量名字不要带有类型信息
   4. 常量和变量命名一致
   5. 接口: 对于接口类型优先以单个单词命名. 对于拥有唯一方法（method）或通过多个拥有唯一方法的接口组合而成的接口，Go语言的惯例是用“方法名+er”命名
3. Go在给标识符命名时还有着考虑上下文环境的惯例，即在不影响可读性的前提下，兼顾一致性原则，尽可能地用短小的名字命名标识符

## 2. 使用一致的变量声明形式

Go语言沿袭了静态编译型语言的传统：使用变量之前需要先进行变量的声

### 2.1 包级变量

1. 包级变量只能使用带有var关键字的变量声明形式
2. 就近原则，即尽可能在靠近第一次使用变量的位置声明该变量。就近原则实际上是变量的作用域最小化的一种实现手段

### 2.2 局部变量

1. 对于延迟初始化的局部变量声明，采用带有var关键字的声明形式
2. 另一种常见的采用带var关键字声明形式的变量是error类型的变量err（将error类型变量实例命名为err也是Go的一个惯用法），尤其是当defer后接的闭包函数需要使用err判断函数/方法退出状态时
3. 短变量声明形式使用
   1. 对于声明且显式初始化的局部变量，建议使用短变量声明形式
   2. 对于不接受默认类型的变量，依然可以使用短变量声明形式，只是在“:=”右侧要进行显式转型：
   3. 尽量在分支控制时应用短变量声明形式

```go
a := 17
f := 3.14
s := "hello, gopher!"
```

### 2.3 常量

有类型常量:用type定义的类型, 不同type之间的变量进行运算时,不支持隐式的类型转换,会报编译错误. 必须进行显式类型转换

无类型常量: 在参与变量法赋值和计算过程中无需显式类型转换,达到简化代码的目的

## 2.4 枚举

go语言没有提供定义枚举常量的语法, 使用常量语法定义枚举常量.

1. Go的const语法提供了“隐式重复前一个非空表达式”的机制

```go
const (
    Apple, Banana = 11, 22
    Strawberry, Grape
    Pear, Watermelon
)
// 两者等价
const (
    Apple, Banana = 11, 22
    Strawberry, Grape  = 11, 22
    Pear, Watermelon  = 11, 22
)

```

2. iota是Go语言的一个预定义标识符，它表示的是const声明块（包括单行声明）中每个常量所处位置在块中的偏移值（从零开始）。同时，每一行中的iota自身也是一个无类型常量，可以像无类型常量那样自动参与不同类型的求值过程，而无须对其进行显式类型转换操作
3. 两者结合

```go
// $GOROOT/src/sync/mutex.go (go 1.12.7)
const (
    mutexLocked = 1 << iota
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6
)
```

1. mutexLocked = 1 << iota：这里是const声明块的第一行，iota的值是该行在const块中的偏移量，因此iota的值为0，我们得到mutexLocked这个常量的值为1 << 0，即1
2. mutexStarving：该常量同mutexWoken，该行等价于mutexStarving = 1 << iota，由于在该行的iota的值为2，因此我们得到mutexStarving这个常量的值为 1 << 2，即4。mutexWaiterShift = iota：这一行的常量初始化表达式与前三行不同，由于该行为第四行，iota的偏移值为3，因此mutexWaiterShift的值就为3。
3. 位于同一行的iota即便出现多次，其值也是一样的, 因为iota指的是const块的行偏移量
4. 如果要略过iota=0, 或者定义非连续枚举值

```go
// $GOROOT/src/syscall/net_js.go，go 1.12.7

const (
    _ = iota
    IPV6_V6ONLY                     // 1
    SOMAXCONN                       // 2
    SO_ERROR                        // 3
)

const (
    _ = iota                        // 0
    Pin1
    Pin2
    Pin3
    _                               // 相当于_ = iota，略过了4这个枚举值
    Pin5                            // 5
)
```

5. Go的枚举常量不限于整型值，也可以定义浮点型的枚举常量

## 2.5 零值初始化

零值可用理念: 变量被赋值零值时仍然可以正常使用, 代码需处理零值时的情况

1. 所有整型类型：0
2. 浮点类型：0.0
3. 布尔类型：false
4. 字符串类型：""
5. 指针、interface、切片（slice）、channel、map、function：nil
6. Go的零值初始是递归的，即数组、结构体等类型的零值初始化就是对其组成元素逐一进行零值初始化

零值可用示例:

```go
// nil可以append
var zeroSlice []int
zeroSlice = append(zeroSlice, 1)
zeroSlice = append(zeroSlice, 2)
zeroSlice = append(zeroSlice, 3)
fmt.Println(zeroSlice) // 输出：[1 2 3]

// nil指针调用
// chapter3/sources/call_method_through_nil_pointer.go

func main() {
    var p *net.TCPAddr
    fmt.Println(p) //输出：<nil>
}
// $GOROOT/src/net/tcpsock.go
func (a *TCPAddr) String() string {
    if a == nil {
        return "<nil>"
    }
    ip := ipEmptyString(a.IP)
    if a.Zone != "" {
        return JoinHostPort(ip+"%"+a.Zone, itoa(a.Port))
    }
    return JoinHostPort(ip, itoa(a.Port))
}

// 无需初始化
var mu sync.Mutex
mu.Lock()
mu.Unlock()

// chapter3/sources/bytes_buffer_write.go
func main() {
    var b bytes.Buffer
    b.Write([]byte("Effective Go"))
    fmt.Println(b.String()) // 输出：Effective Go
}
```

不可用场景:

```go
// 1. 下标索引
var s []int
s[0] = 12         // 报错！
s = append(s, 12) // 正确

// 2. map类型
var m map[string]int
m["go"] = 1 // 报错！

m1 := make(map[string]int)
m1["go"] = 1 // 正确

// 3. 避免值复制
var mu sync.Mutex
mu1 := mu // 错误: 避免值复制
foo(mu) // 错误: 避免值复制
```

## 3. 使用复合字面值作为初值构造器

Go语言中的复合类型包括结构体、数组、切片和map。对于复合类型变量，最常见的值构造方式就是对其内部元素进行逐个赋值

```go
var s myStruct
s.name = "tony"
s.age = 23

var a [5]int
a[0] = 13
a[1] = 14
...
a[4] = 17

sl := make([]int, 5, 5)
sl[0] = 23
sl[1] = 24
...
sl[4] = 27

m := make(map[int]string)
m[1] = "hello"
m[2] = "gopher"
m[3] = "!"
```

但这样的值构造方式让代码显得有些烦琐，尤其是在构造组成较为复杂的复合类型变量的初值时。Go提供的复合字面值（composite literal）语法可以作为复合类型变量的初值构造器。上述代码可以使用复合字面值改写成下面这样：

使用块语句初始化: {}

**复合字面值**由两部分组成：一部分是类型，比如上述示例代码中赋值操作符右侧的myStruct、[5]int(数组)、[]int(切片)和map[int]string；另一部分是由大括号{}包裹的字面值

```go
s := myStruct{"tony", 23}
a := [5]int{13, 14, 15, 16, 17}
sl := []int{23, 24, 25, 26, 27}
m := map[int]string {1:"hello", 2:"gopher", 3:"!"}
```

### 3.1 结构体

1. Go推荐使用field:value的复合字面值形式对struct类型变量进行值构造，这种值构造方式可以降低结构体类型使用者与结构体类型设计者之间的耦合，这也是Go语言的惯用法
2. 与之前普通复合字面值形式不同，field:value形式字面值中的字段可以以任意次序出现，未显式出现在字面值的结构体中的字段将采用其对应类型的零值
3. 通过在复合字面值构造器的类型前面增加&，可以得到对应类型的指针类型变量，如下面例子中的变量p的类型即为Pipe类型指针

```go
// $GOROOT/src/net/http/transport.go
var DefaultTransport RoundTripper = &Transport{
    Proxy: ProxyFromEnvironment,
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        DualStack: true,
    }).DialContext,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}

// $GOROOT/src/io/pipe.go
type pipe struct {
    wrMu sync.Mutex
    wrCh chan []byte
    rdCh chan int

    once sync.Once
    done chan struct{}
    rerr onceError
    werr onceError
}

func Pipe() (*PipeReader, *PipeWriter) {
    p := &pipe{
        wrCh: make(chan []byte),
        rdCh: make(chan int),
        done: make(chan struct{}),
    }
    return &PipeReader{p}, &PipeWriter{p}
}
```

5. 复合字面值作为结构体值构造器的大量使用，使得即便采用类型零值时我们也会使用字面值构造器形式. 而较少使用new这一个Go预定义的函数来创建结构体变量实例
6. 值得注意的是，不允许将从其他包导入的结构体中的**未导出字段**作为复合字面值中的field，这会导致编译错误

### 3.2 数组/切片

与结构体类型不同，数组/切片使用下标（index）作为field:value形式中的field，从而实现数组/切片初始元素值的高级构造形式

```go
numbers := [256]int{'a': 8, 'b': 7, 'c': 4, 'd': 3, 'e': 2, 'y': 1, 'x': 5}

// [10]float{-1, 0, 0, 0, -0.1, -0.1, 0, 0.1, 0, -1}
fnumbers := [...]float{-1, 4: -0.1, -0.1, 7:0.1, 9: -1}

// $GOROOT/src/sort/search_test.go
var data = []int{0: -10, 1: -5, 2: 0, 3: 1, 4: 2, 5: 3, 6: 5, 7: 7,
       8: 11, 9: 100, 10: 100, 11: 100, 12: 1000, 13: 10000}
var sdata = []string{0: "f", 1: "foo", 2: "foobar", 3: "x"}
```

### 3.3 map

和结构体、数组/切片相比，map类型变量使用复合字面值作为初值构造器就显得自然许多，因为map类型具有原生的key:value构造形式：

```go
// $GOROOT/src/time/format.go
var unitMap = map[string]int64{
    "ns": int64(Nanosecond),
    "us": int64(Microsecond),
    "µs": int64(Microsecond), // U+00B5 = 微符号
    "μs": int64(Microsecond), // U+03BC = 希腊字母μ
    "ms": int64(Millisecond),
    ...
}


// $GOROOT/src/net/http/server.go
var stateName = map[ConnState]string{
    StateNew:      "new",
    StateActive:   "active",
    StateIdle:     "idle",
    StateHijacked: "hijacked",
    StateClosed:   "closed",
}
```

对于数组/切片类型而言，当元素为复合类型时，可以省去元素复合字面量中的类型

```go
type Point struct {
    x float64
    y float64
}

sl := []Point{
    {1.2345, 6.2789}, // Point{1.2345, 6.2789}
    {2.2345, 19.2789}, // Point{2.2345, 19.2789}
}

// Go 1.5之前版本

m := map[Point]string{
    Point{29.935523, 52.891566}:   "Persepolis",
    Point{-25.352594, 131.034361}: "Uluru",
    Point{37.422455, -122.084306}: "Googleplex",
}


// Go 1.5及之后版本
m := map[Point]string{
    {29.935523, 52.891566}:   "Persepolis",
    {-25.352594, 131.034361}: "Uluru",
    {37.422455, -122.084306}: "Googleplex",
}

m1 := map[string]Point{
    "Persepolis": {29.935523, 52.891566},
    "Uluru":      {-25.352594, 131.034361},
    "Googleplex": {37.422455, -122.084306},
}
```
