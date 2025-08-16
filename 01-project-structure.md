# GO语言典型项目结构

## 1. 典型项目结构

go语言的一个**典型项目结构**如下图:

<img src="images/project结构.png" height=80% height=80%>

上图就是一个支持构建二进制可执行文件的典型go项目结构,各目录如下:

1. cmd目录:
   1. 存放项目要构建的可执行文件对应的main包的源文件。如果有多个可执行文件需要构建，则将每个可执行文件的main包单独放在一个子目录中，比如图中的app1、app2
   2. 我们会在main包中做一些命令行参数解析、资源初始化、日志设施初始化、数据库连接初始化等工作，之后就会将程序的执行权限交给更高级的执行控制对象
2. pkg目录: 存放项目自身要使用并且同样也是可执行文件对应main包要依赖的库文件。该目录下的包可以被外部项目引用，算是项目导出包的一个聚合
3. go.mod和go.sum: go语言包依赖管理使用的配置文件.Go 1.16版本中，Go module成为默认的依赖包管理和构建机制。因此对于新的Go项目，建议基于Go module进行包依赖管理
4. internal目录: 对于不想暴露给外部引用，仅限项目内部使用的包，最简单的方式就是在顶层加入一个internal目录，将不想暴露到外部的包都放在该目录下.
   1. 举例如下:internal目录下的ilib1、ilib2可以被以GoLibProj目录为根目录的其他目录下的代码（比如lib.go、lib1/lib1.go等）所导入和使用，但是却不可以为GoLibProj目录以外的代码所使用，从而实现选择性地暴露API包
   2. 当然internal也可以放在项目结构中的任一目录层级中，关键是项目结构设计人员明确哪些要暴露到外层代码，哪些仅用于**同级目录或子目录**中。

```plaintext
// 带internal的Go库项目结构

$tree -F ./chapter2/sources/GoLibProj
GoLibProj
├── LICENSE
├── Makefile
├── README.md
├── go.mod
├── internal/
│  ├── ilib1/
│  └── ilib2/
├── lib.go
├── lib1/
│  └── lib1.go
└── lib2/
      └── lib2.go
```

## 2. go代码风格

1. gofmt工具: 格式化代码风格
2. goimports: goimports在gofmt功能的基础上增加了对包导入列表的维护功能，可根据源码的最新变动自动从导入包列表中增删包
3. 以上功能都集成在在主流IDE中, vscode, goland等, 无需深入了解
