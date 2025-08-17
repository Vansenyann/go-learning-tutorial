# 接口类型

## 1. 接口类型变量

接口类型“动静兼备”的特性决定了它的变量的内部表示绝不像静态类型（如int、float64）变量那样简单。我们可以在$GOROOT/src/runtime/runtime2.go中找到接口类型变量在运行时的表示：

```go
// $GOROOT/src/runtime/runtime2.go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// $GOROOT/src/runtime/runtime2.go

type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32
    _     [4]byte
    fun   [1]uintptr
}

// $GOROOT/src/runtime/type.go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

// $GOROOT/src/runtime/type.go

type _type struct {
    size       uintptr
    ptrdata    uintptr
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```

接口类型变量有两种内部表示—eface和iface，这两种表示分别用于不同接口类型的变量。

1. eface：用于表示没有方法的空接口（empty interface）类型变量，即interface{}类型的变量。
2. iface：用于表示其余拥有方法的接口（interface）类型变量

这两种结构的共同点是都有两个指针字段，并且**第二个指针字段的功用相同，都指向当前赋值给该接口类型变量的动态类型变量的值。**

不同点在于eface所表示的空接口类型并无方法列表，因此其第一个指针字段指向一个_type类型结构，该结构为该接口类型变量的动态类型的信息：

上面itab结构中的第一个字段inter指向的interfacetype结构存储着该接口类型自身的信息。interfacetype类型定义如下，该interfacetype结构由类型信息（typ）、包路径名（pkgpath）和接口方法集合切片（mhdr）组成。

itab结构中的字段_type则存储着该接口类型变量的动态类型的信息，字段fun则是动态类型已实现的接口方法的调用地址数组。

举例:

1. eface空接口类型

```go
type T struct {
    n int
    s string
}

func main() {
    var t = T {
        n: 17,
        s: "hello, interface",
    }
    var ei interface{} = t // Go运行时使用eface结构表示ei
}
```

`<img src="images/eface.png">`

2. iface示例

```go
type T struct {
    n int
    s string
}

func (T) M1() {}
func (T) M2() {}

type NonEmptyInterface interface {
    M1()
    M2()
}

func main() {
    var t = T{
        n: 18,
        s: "hello, interface",
    }
    var i NonEmptyInterface = t
}
```

`<img src="imgaes/iface.png">`

1. 每个接口类型变量在运行时的表示都是由两部分组成的，方法和字段
2. 这两种接口类型可以分别简记为eface(_type,data)和iface(tab, data)。虽然eface和iface的第一个字段有所差别，但tab和_type可统一看作动态类型的类型信息
3. Go语言中每种类型都有唯一的_type信息，无论是内置原生类型，还是自定义类型。Go运行时会为程序内的全部类型建立只读的共享_type信息表，因此拥有相同动态类型的同类接口类型变量的_type/tab信息是相同的。
4. 而接口类型变量的data部分则指向一个动态分配的内存空间，该内存空间存储的是赋值给接口类型变量的动态类型变量的值。未显式初始化的接口类型变量的值为nil，即该变量的_type/tab和data都为nil。即该变量的_type/tab和data都为nil。
5. 这样，我们要判断两个接口类型变量是否相同，只需判断_type/tab是否相同以及data指针所指向的内存空间所存储的数据值是否相同（注意：不是data指针的值

## 装箱

在Go语言中，将任意类型赋值给一个接口类型变量都是装箱操作。有了前面对接口类型变量内部表示的了解，我们知道接口类型的装箱实则就是创建一个eface或iface的过程。

示例:

```go
// chapter5/sources/interface-internal-4.go

type T struct {
    n int
    s string
}

func (T) M1() {}
func (T) M2() {}

type NonEmptyInterface interface {
    M1()
    M2()
}

func main() {
    var t = T{
        n: 17,
        s: "hello, interface",
    }
    var ei interface{}
    ei = t

    var i NonEmptyInterface
    i = t
    fmt.Println(ei)
    fmt.Println(i)
}
```

在将动态类型变量赋值给接口类型变量语句对应的汇编代码中，我们看到了convT2E和convT2I两个runtime包的函数。这两个函数的实现位于$GOROOT/src/runtime/iface.go中

```go
// $GOROOT/src/runtime/iface.go
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}
```

convT2E用于将任意类型转换为一个eface，convT2I用于将任意类型转换为一个iface。两个函数的实现逻辑相似，主要思路就是根据传入的类型信息（convT2E的_type和convT2I的tab._type）分配一块内存空间，并将elem指向的数据复制到这块内存空间中，最后传入的类型信息作为返回值结构中的类型信息，返回值结构中的数据指针（data）指向新分配的那块内存空间。

由此我们也可以看出，经过装箱后，箱内的数据（存放在新分配的内存空间中）与原变量便无瓜葛了，除非是指针类型。

那么convT2E和convT2I函数的类型信息从何而来？这些都依赖Go编译器的工作。编译器知道每个要转换为接口类型变量（toType）的动态类型变量的类型（fromType），会根据这一类型选择适当的convT2X函数（见下面代码中的convFuncName），并在生成代码时使用选出的convT2X函数参与装箱操作：

## 3. 接口使用

1. 使用小接口
2. 避免使用空接口作为函数参数类型

垂直组合（类型组合）：Go语言主要通过类型嵌入机制实现垂直组合，进而实现方法实现的复用、接口定义重用等。

水平组合：通常Go程序以接口类型变量作为程序水平组合的连接点。接口是水平组合的关键，它就好比程序肌体上的关节，给予连接关节的两个部分或多个部分各自自由活动的能力，而整体又实现了某种功能。
