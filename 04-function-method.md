# 函数与方法

函数和方法是Go程序逻辑的基本承载单元。本部分聚焦于函数与方法的设计与实现，涵盖init函数的使用、跻身“一等公民”行列的函数有何不同、Go方法的本质等

从程序逻辑结构角度来看，包（package）是Go程序逻辑封装的基本单元，每个包都可以理解为一个“自治”的、封装良好的、对外部暴露有限接口的基本单元。一个Go程序就是由一组包组成的

在Go包这一基本单元中分布着常量、包级变量、函数、类型和类型方法、接口等，我们要保证包内部的这些元素在被使用之前处于合理有效的初始状态，尤其是包级变量。在Go语言中，我们一般通过包的init函数来完成这一工作。

## 1. init函数

Go语言中有两个特殊的函数：一个是main包中的main函数，它是所有Go可执行程序的入口函数；另一个就是包的init函数。

init函数是一个无参数、无返回值的函数：

如果一个包定义了init函数，Go运行时会负责在该包初始化时调用它的init函数。在Go程序中我们不能显式调用init，否则会在编译期间报错

```go
func init() {
    ...
}
```

一个Go包可以拥有多个init函数，每个组成Go包的Go源文件中可以定义多个init函数。在初始化Go包时，Go运行时会按照一定的次序逐一调用该包的init函数。

Go运行时不会并发调用init函数，它会等待一个init函数执行完毕并返回后再执行下一个init函数，且每个init函数在整个Go程序生命周期内仅会被执行一次。因此，init函数极其适合做一些包级数据的初始化及初始状态的检查工作。

般来说，先被传递给Go编译器的源文件中的init函数先被执行，同一个源文件中的多个init函数按声明顺序依次执行。但Go语言的惯例告诉我们：不要依赖init函数的执行次序。

## 2. go程序初始化

Go程序由一组包组合而成，程序的初始化就是这些包的初始化。每个Go包都会有自己的依赖包，每个包还包含有常量、变量、init函数等（其中main包有main函数），这些元素在程序初始化过程中的初始化顺序是什么样的呢？

`<img src="images/包的初始化.png" width=100% height=100%>`

1. main包直接依赖pkg1、pkg4两个包；
2. Go运行时会根据包导入的顺序，先去初始化main包的第一个依赖包pkg1；
3. Go运行时遵循“深度优先”原则查看到pkg1依赖pkg2，于是Go运行时去初始化pkg2；
4. pkg2依赖pkg3，Go运行时去初始化pkg3；
5. pkg3没有依赖包，于是Go运行时在pkg3包中按照常量→变量→init函数的顺序进行初始化；
6. pkg3初始化完毕后，Go运行时会回到pkg2并对pkg2进行初始化，
7. 之后再回到pkg1并对pkg1进行初始化；在调用完pkg1的init函数后，Go运行时完成main包的第一个依赖包pkg1的初始化；
8. Go运行时接下来会初始化main包的第二个依赖包pkg4；
9. pkg4的初始化过程与pkg1类似，也是先初始化其依赖包pkg5，然后再初始化自身；在Go运行时初始化完pkg4后，也就完成了对main包所有依赖包的初始化，
10. 接下来初始化main包自身；在main包中，Go运行时会按照常量→变量→init函数的顺序进行初始化，执行完这些初始化工作后才正式进入程序的入口函数main函数。

## 3. 让自己习惯于函数是“一等公民”

o语言的函数具有如下特点：

1. 以func关键字开头；
2. 支持多返回值；
3. 支持具名返回值；
4. 支持递归调用；
5. 支持同类型的可变参数；
6. 支持defer，实现函数优雅返回。

**一等公民定义:**

> 如果一门编程语言对某种语言元素的创建和使用没有限制，我们可以像对待值（value）一样对待这种语法元素，那么我们就称这种语法元素是这门编程语言的“一等公民”。拥有“一等公民”待遇的语法元素可以存储在变量中，可以作为参数传递给函数，可以在函数内部创建并可以作为返回值从函数返回。在动态类型语言中，语言运行时还支持对“一等公民”类型的检查。

**像使用整型值一样使用函数**

### 3.1 像对整型变量那样对函数进行显式类型转换

函数是“一等公民”，对整型变量进行的操作也可以用在函数上，即函数也可以被显式类型转换，并且这样的类型转换在特定的领域具有奇妙的作用。最为典型的示例就是http.HandlerFunc这个类型，我们来看一下例子：

```go
// chapter4/sources/function_as_first_class_citizen_2.go

func greeting(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome, Gopher!\n")
}

func main() {
    http.ListenAndServe(":8080", http.HandlerFunc(greeting))
}

// $GOROOT/src/net/http/server.go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}

// $GOROOT/src/net/http/server.go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// $GOROOT/src/net/http/server.go

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP调用f(w, r)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

HandlerFunc其实就是一个基于函数定义的新类型，它的底层类型为func(ResponseWriter, *Request)。

该类型有一个方法ServeHTTP，因而实现了Handler接口。

也就是说，http.HandlerFunc(greeting)这句代码的真正含义是将函数greeting显式转换为HandlerFunc类型，而后者实现了Handler接口，这样转型后的greeting就满足了ListenAndServe函数第二个参数的要求。

另外，之所以http.HandlerFunc(greeting)这条语句可以通过编译器检查，是因为HandlerFunc的底层类型是func(ResponseWriter,*Request)，与greeting的原型是一致的。这和整型变量的转型原理并无二致：

```go
type MyInt int
var x int = 5
y := MyInt(x) // MyInt的底层类型为int，类比 HandlerFunc的底层类型为func(ResponseWriter,  *Request)
```

简化示例:

```go
// chapter4/sources/function_as_first_class_citizen_3.go
// 这是一个接口
type BinaryAdder interface {
    Add(int, int) int
}

// 这是一个类型
type MyAdderFunc func(int, int) int

func (f MyAdderFunc) Add(x, y int) int {
    return f(x, y)
}
// 这是一个func
func MyAdd(x, y int) int {
    return x + y
}

func main() {
    var i BinaryAdder = MyAdderFunc(MyAdd)
    fmt.Println(i.Add(5, 6))
}
```

和Web Server那个例子类似，我们想将MyAdd函数赋值给BinaryAdder接口。直接赋值是不行的，

我们需要一个底层函数类型与MyAdd一致的**自定义类型的显式转换**，这个自定义类型就是MyAdderFunc，

该类型实现了BinaryAdder接口，这样在经过MyAdderFunc的显式类型转换后，MyAdd被赋值给了BinaryAdder的变量i。这样，通过i调用的Add方法实质上就是MyAdd函数。

### 3.2 函数式编程

#### 3.2.1 柯里化函数:

柯里化是把接受多个参数的函数变换成接受一个单一参数（原函数的第一个参数）的函数，并返回接受余下的参数和返回结果的新函数的技术

示例:

```go
// chapter4/sources/function_as_first_class_citizen_4.go
package main

import "fmt"

func times(x, y int) int {
    return x * y
}

func partialTimes(x int) func(int) int {
    return func(y int) int {
        return times(x, y)
    }
}

func main() {
    timesTwo := partialTimes(2)
    timesThree := partialTimes(3)
    timesFour := partialTimes(4)
    fmt.Println(timesTwo(5))
    fmt.Println(timesThree(5))
    fmt.Println(timesFour(5))
}
```

这里的柯里化是指将原来接受两个参数的函数times转换为接受一个参数的函数partialTimes的过程。通过partialTimes函数构造的timesTwo将输入参数扩大为原先的2倍、timesThree将输入参数扩大为原先的3倍

#### **3.2.1 闭包:**

这个例子利用了函数的两点性质：在函数中定义，通过返回值返回；闭包。

闭包前面没有提到，它是Go函数支持的另一个特性。闭包**是在函数内部定义的匿名函数**，**并且允许该匿名函数访问定义它的外部函数的作用域**。本质上，闭包是将函数内部和函数外部连接起来的桥梁

以上述示例来说，partialTimes内部定义的匿名函数就是一个闭包，该匿名函数访问了其外部函数partialTimes的变量x。这样当调用partialTimes(2)时，partialTimes实际上返回一个调用times(2, y)的函数

#### **3.2.2 函子**

什么是函子呢？具体来说，函子需要满足两个条件：函子本身是一个容器类型，以Go语言为例，这个容器可以是切片、map甚至channel；该容器类型需要实现一个方法，该方法接受一个函数类型参数，并在容器的每个元素上应用那个函数，得到一个新函子，原函子容器内部的元素值不受影响。

```go
// chapter4/sources/function_as_first_class_citizen_5.go

type IntSliceFunctor interface {
    Fmap(fn func(int) int) IntSliceFunctor
}

type intSliceFunctorImpl struct {
    ints []int
}

func (isf intSliceFunctorImpl) Fmap(fn func(int) int) IntSliceFunctor {
    newInts := make([]int, len(isf.ints))
    for i, elt := range isf.ints {
        retInt := fn(elt)
        newInts[i] = retInt
    }
    return intSliceFunctorImpl{ints: newInts}
}

func NewIntSliceFunctor(slice []int) IntSliceFunctor {
    return intSliceFunctorImpl{ints: slice}
}

func main() {
    // 原切片
    intSlice := []int{1, 2, 3, 4}
    fmt.Printf("init a functor from int slice: %#v\n", intSlice)
    f := NewIntSliceFunctor(intSlice)
    fmt.Printf("original functor: %+v\n", f)

    mapperFunc1 := func(i int) int {
        return i + 10
    }

    mapped1 := f.Fmap(mapperFunc1)
    fmt.Printf("mapped functor1: %+v\n", mapped1)

    mapperFunc2 := func(i int) int {
        return i * 3
    }
    mapped2 := mapped1.Fmap(mapperFunc2)
    fmt.Printf("mapped functor2: %+v\n", mapped2)
    fmt.Printf("original functor: %+v\n", f) // 原函子没有改变
    fmt.Printf("composite functor: %+v\n", f.Fmap(mapperFunc1).Fmap(mapperFunc2))
}
```

这段代码的具体逻辑如下。定义了一个intSliceFunctorImpl类型，用来作为函子的载体。

我们把函子要实现的方法命名为Fmap，intSliceFunctorImpl类型实现了该方法。该方法也是IntSliceFunctor接口的唯一方法。可以看到在这个代码中，真正的函子其实是IntSliceFunctor，这符合Go的惯用法。

我们定义了创建IntSliceFunctor的函数NewIntSliceFunctor。通过该函数以及一个初始切片，我们可以实例化一个函子。我们在main中定义了两个转换函数，并将这两个函数应用到上述函子实例。得到的新函子的内部容器元素值是原容器的元素值经由转换函数转换后得到的

最后，我们还可以对最初的函子实例连续（组合）应用转换函数，这让我们想到了数学课程中的函数组合。无论如何应用转换函数，原函子中容器内的元素值不受影响。

函子非常适合用来对容器集合元素进行批量同构处理，而且代码也比每次都对容器中的元素进行循环处理要优雅、简洁许多。但要想在Go中发挥函子的最大效能，还需要Go对泛型提供支持，否则我们就需要为每一种容器类型都实现一套对应的Functor机制.这段代码的具体逻辑如下。定义了一个intSliceFunctorImpl类型，用来作为函子的载体。我们把函子要实现的方法命名为Fmap，intSliceFunctorImpl类型实现了该方法。该方法也是IntSliceFunctor接口的唯一方法。可以看到在这个代码中，真正的函子其实是IntSliceFunctor，这符合Go的惯用法。我们定义了创建IntSliceFunctor的函数NewIntSliceFunctor。通过该函数以及一个初始切片，我们可以实例化一个函子。我们在main中定义了两个转换函数，并将这两个函数应用到上述函子实例。得到的新函子的内部容器元素值是原容器的元素值经由转换函数转换后得到的。最后，我们还可以对最初的函子实例连续（组合）应用转换函数，这让我们想到了数学课程中的函数组合。无论如何应用转换函数，原函子中容器内的元素值不受影响。函子非常适合用来对容器集合元素进行批量同构处理，而且代码也比每次都对容器中的元素进行循环处理要优雅、简洁许多。但要想在Go中发挥函子的最大效能，还需要Go对泛型提供支持，否则我们就需要为每一种容器类型都实现一套对应的Functor机制

#### 3.2.3 延续传递式

函数式编程有一种被称为延续传递式（Continuation-passing Style，CPS）的编程风格可以充分运用函数作为“一等公民”的特质。

在CPS风格中，函数是不允许有返回值的。一个函数A应该将其想返回的值显式传给一个continuation函数（一般接受一个参数），而这个continuation函数自身是函数A的一个参数

感觉没啥用

## 4. 使用defer

适用场景: 函数中会申请一些资源并在函数退出前释放或关闭这些资源，比如这里的文件描述符f和互斥锁mu。函数的实现需要确保这些资源在函数退出时被及时正确地释放，无论函数的执行流是按预期顺利进行还是出现错误提前退出

在Go中，只有在函数和方法内部才能使用defer；

defer将它们注册到其所在goroutine用于存放deferred函数的栈数据结构中，这些deferred函数将在执行defer的函数退出前被按后进先出（LIFO）的顺序调度执行

`<img src="images/defer调度.png">`

无论是执行到函数体尾部返回，还是在某个错误处理分支显式调用return返回，抑或出现panic，已经存储到deferred函数栈中的函数都会被调度执行。因此，deferred函数是一个在任何情况下都可以为函数进行收尾工作的好场

defer的常见用法:

1. 拦截panic: defer的运行机制决定了无论函数是执行到函数体末尾正常返回，还是在函数体中的某个错误处理分支显式调用return返回，抑或函数体内部出现panic，已经注册了的deferred函数都会被调度执行。因此，defer的第二个重要用途就是拦截panic，并按需要对panic进行处理，可以尝试从panic中恢复（这也是Go语言中唯一的从panic中恢复的手段），也可以如下面标准库代码中这样触发一个新panic，但为新panic传一个新的error值：
2. deferred函数虽然可以拦截绝大部分的panic，但无法拦截并恢复一些运行时之外的致命问题
3. 修改函数的具名返回值

关键问题:

1. 明确哪些函数可以作为deferred函数
   1. 对于自定义的函数或方法，defer可以给予无条件的支持，但是对于有返回值的自定义函数或方法，返回值会在deferred函数被调度执行的时候被自动丢弃。
   2. 内置函数不能用作defer函数: append、cap、len、make、new等内置函数是不可以直接作为deferred函数的，而close、copy、delete、print、recover等可以
   3. 对于那些不能直接作为deferred函数的内置函数，我们可以使用一个包裹它的匿名函数来间接满足要求

```go
defer func() {
    _ = append(sl, 11)
}()
```

2. 把握好defer关键字后表达式的求值时机: 牢记一点，defer关键字后面的表达式是在将deferred函数注册到deferred函数栈的时候进行求值的。

## 5. 方法的本质

和函数相比，Go语言中的方法在声明形式上仅仅多了一个参数，Go称之为receiver参数。receiver参数是方法与类型之间的纽带。

```go
func (receiver T/*T) MethodName(参数列表) (返回值列表) {
    // 方法体
}
```

Go方法具有如下特点。

1. 方法名的首字母是否大写决定了该方法是不是导出方法。

2. 方法定义要与类型定义放在同一个包内。由此我们可以推出：不能为原生类型（如int、float64、map等）添加方法，只能为自定义类型定义方法. 不能横跨Go包为其他包内的自定义类型定义方法。

3) 每个方法只能有一个receiver参数，不支持多receiver参数列表或变长receiver参数。一个方法只能绑定一个基类型，Go语言不支持同时绑定多个类型的方法
4) receiver参数的基类型本身不能是指针类型或接口类型，下面的示例展示了这点

C++的对象在调用方法时，编译器会自动传入指向对象自身的this指针作为方法的第一个参数。而对于Go来说，receiver其实也是同样道理，我们将receiver作为第一个参数传入方法的参数列表。上面示例中类型T的方法可以等价转换为下面的普通函数

```go
type T struct {
    a int
}

func (t T) Get() int {
    return t.a
}

func (t *T) Set(a int) int {
    t.a = a
    return t.a
}

// 等价
func Get(t T) int {
    return t.a
}

func Set(t *T, a int) int {
    t.a = a
    return t.a
}
```

这种转换后的函数就是方法的原型。只不过在Go语言中，这种等价转换是由Go编译器在编译和生成代码时自动完成的

这就是Go方法的本质：**一个以方法所绑定类型实例为第一个参数的普通函数**

### **5.1 选择正确的receiver类型**

方法和函数的等价变换公式:

```go
func (t T) M1() <=> M1(t T)
func (t *T) M2() <=> M2(t *T)
```

M1方法的receiver参数类型为T，而M2方法的receiver参数类型为*T

1. 当receiver参数的类型为T时，选择值类型的receiver: 选择以T作为receiver参数类型时，T的M1方法等价为M1(t T)。Go函数的参数采用的是值复制传递，也就是说M1函数体中的t是T类型实例的一个副本，这样在M1函数的实现中对参数t做任何修改都只会影响副本，而不会影响到原T类型实例
2. receiver参数的类型为*T时，选择指针类型的receiver: 选择以*T作为receiver参数类型时，T的M2方法等价为M2(t *T)。我们传递给M2函数的t是T类型实例的地址，这样M2函数体中对参数t做的任何修改都会反映到原T类型实例上。

### 5.2 方法集合

如果某个自定义类型T的方法集合是某个接口类型的方法集合的超集，那么就说类型T实现了该接口，并且类型T的变量可以被赋值给该接口类型的变量，即我们说的方法集合决定接口实现。

方法集合是Go语言中一个重要的概念，在为接口类型变量赋值、使用结构体嵌入/接口嵌入、类型别名和方法表达式等时都会用到方法集合，它像胶水一样将自定义类型与接口隐式地黏结在一起

我们完全明确了为receiver选择类型时需要考虑的第三点因素：是否支持将T类型实例赋值给某个接口类型变量。如果需要支持，我们就要实现receiver为T类型的接口类型方法集合中的所有方法。
