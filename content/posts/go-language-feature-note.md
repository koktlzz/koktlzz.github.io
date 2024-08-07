---
title: "《Go 语言设计与实现》读书笔记：语言特性"
date: 2024-08-05T23:09:42+01:00
draft: false
series: ["《Go 语言设计与实现》读书笔记"]
tags: ["Go"]
summary: "函数是 Go 语言的一等公民，这意味着它可以作为参数传递给其他函数、作为其他函数的返回以及分配给变量或存储在数据结构中 ..."
---

> 原书中的代码片段基于 Go 1.15，笔记则根据 Go 1.22 版本的更新进行了相应替换。

## 函数调用

函数是 Go 语言的一等公民，这意味着它可以作为参数传递给其他函数、作为其他函数的返回以及分配给变量或存储在数据结构中。

### 调用惯例

调用惯例（Calling Convention）是调用方与被调用方对参数和返回值传递的约定，它是 [应用二进制接口](https://en.wikipedia.org/wiki/Application_binary_interface)（Application Binary Interface，ABI）的一部分。

![api-abi-isa](https://media.geeksforgeeks.org/wp-content/uploads/Untitled-drawing-15.png)

C 语言的调用惯例详见 [过程](/posts/machine-level-representation-of-programs-note/#过程)，其主要特点为：

- 六个以及六个以下的参数会按从左往右的顺序分别使用 rdi、rsi、rdx、rcx、r8 和 r9 寄存器传递；
- 六个以上的参数会使用栈传递，函数的参数会按从右往左的顺序依次存入栈中；
- 函数的返回值主要是通过 rax 寄存器进行传递的，不能同时返回多个值；
- 如果被调用函数要使用 rbx、rbp 和 %r12–%r15 寄存器（即 [被保存的寄存器](/posts/machine-level-representation-of-programs-note/#被保存的寄存器)），则有责任保存调用之前的数据。

在之前版本的 Go 语言中，被调用函数的参数和返回值均保存在调用者栈中，被调用者根据栈的相对位置读取参数和返回值。这种设计只需要在栈上多分配一些内存就可以返回多个值并且降低了实现的复杂度，但同样也牺牲了函数调用的性能。

Go 语言从 1.17 版本开始将使用寄存器传参，见 [Proposal: Register-based Go calling convention](https://go.googlesource.com/proposal/+/master/design/40724-register-calling.md)。其主要特点如下：

- 九个以及九个以下的参数和返回值会按顺序分别使用  AX、BX、CX、DI、SI、R8、R9、R10 和 R11 寄存器传递；
- 九个以上的参数和返回值则使用栈传递；
- 所有寄存器都是调用者保存（Caller-saved）寄存器，被调用者可以随意修改其值。

### 参数传递

传值和传引用之间的区别：

- 传值：函数调用时会对参数进行复制，被调用方和调用方两者持有不相关的两份数据；
- 传引用：函数调用时会传递参数的指针，被调用方和调用方两者持有相同的数据，任意一方做出的修改都会影响另一方。

Go 语言选择传值的方式，无论是传递基本类型、结构体还是指针，都会赋值传递的参数。在传递数组或者内存占用非常大的结构体时，我们应该尽量使用指针作为参数类型来避免发生数据拷贝进而影响性能。

```go
type MyStruct struct {
    i int
}

func myFunction(a MyStruct, b *MyStruct) {
    a.i = 31
    (*b).i = 41
    fmt.Printf("in my_function - a=(%d, %p) b=(%v, %p)\n", a, &a, b, &b)
}

func main() {
    a := MyStruct{i: 30}
    b := &MyStruct{i: 40}
    fmt.Printf("before calling - a=(%d, %p) b=(%v, %p)\n", a, &a, b, &b)
    myFunction(a, b)
    fmt.Printf("after calling  - a=(%d, %p) b=(%v, %p)\n", a, &a, b, &b)
}

$ go run main.go
before calling - a=({30}, 0xc000018178) b=(&{40}, 0xc00000c028)
in my_function - a=({31}, 0xc000018198) b=(&{41}, 0xc00000c038)
after calling  - a=({30}, 0xc000018178) b=(&{41}, 0xc00000c028)
```

## 接口

### 概述

在计算机科学中，接口是计算机系统中多个组件共享的边界，不同的组件能够在边界上交换信息。如下图所示，接口的本质是引入一个新的中间层，调用方可以通过接口与具体实现分离，解除上下游的耦合，上层的模块不再需要依赖下层的具体模块，只需要依赖一个约定好的接口。

![20240806165330](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20240806165330.png)

Go 语言中的接口是一种内置的类型，它定义了一组方法的签名。

#### 隐式接口

Go 语言中接口的实现都是隐式的，且只会在传递参数、返回参数以及变量赋值时才会对某个类型是否实现接口进行检查。

#### 类型

Go 语言中有两种略微不同的接口，一种是带有一组方法的接口，另一种是不带任何方法的`interface{}`。与 C 语言中的`void *`不同，`interface{}`类型不是任意类型。

```go
func main() {
    type Test struct{}
    v := Test{}
    Print(v)
}

func Print(v interface{}) {
    println(v)
}
```

上述代码中，`Print`函数只接受`interface{}`类型的参数，变量`v`在被该函数调用时会由原来的`Test`类型转换为`interface{}`类型。

#### 指针和接口

当方法的接收者是指针时，只有指针类型的变量才能实现接口，使用结构体初始化变量则无法通过编译：

```go
type Duck interface {
    Quack()
}

type Cat struct{}

func (c *Cat) Quack() {
    fmt.Println("meow")
}

func main() {
    var c Duck = Cat{}
    c.Quack()
}

// cannot use Cat{} (value of type Cat) as Duck value in variable declaration: 
// Cat does not implement Duck (method Quack has pointer receiver)
```

而当方法的接收者是结构体时，指针类型和结构体类型的变量都能实现接口：

```go
type Duck interface {
    Quack()
}

type Cat struct{}

func (c Cat) Quack() {
    fmt.Println("meow")
}

func main() {
    var c Duck = &Cat{}
    c.Quack()
}
```

这是因为编译器会自动生成一个接收者为`*Cat`类型的`Quack()`方法：

```shell
$ go tool compile -l  -p main main.go
$ go tool nm main.o | grep 'T' | grep Cat
    2403 T main.(*Cat).Quack
    1e86 T main.Cat.Quack
```

#### nil 和 non-nil

```go
package main

type TestStruct struct{}

func NilOrNot(v interface{}) bool {
    return v == nil
}

func main() {
    var s *TestStruct
    fmt.Println(s == nil)      // #=> true
    fmt.Println(NilOrNot(s))   // #=> false
}
```

调用`NilOrNot`函数时发生了隐式的类型转换，即`*TestStruct`类型被转换成`interface{}`类型。转换后的变量不仅包含转换前的变量，还包含变量的类型信息`TestStruct`，因此转换后的变量与`nil`不相等。

### 数据结构

Go 语言根据接口类型是否包含一组方法将接口类型分成了两类：

- 使用 [runtime.iface](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/runtime/runtime2.go#L205) 结构体表示包含方法的接口；
- 使用 [runtime.eface](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/runtime/runtime2.go#L210) （Empty Interface）结构体表示不包含任何方法的`interface{}`类型。

```go
type iface struct { 
    // 16 字节
    tab  *itab
    data unsafe.Pointer
}

type eface struct { 
    // 16 字节
    // 只包含指向底层数据和类型的两个指针
    _type *_type
    data  unsafe.Pointer
}
```

#### 类型结构体

[runtime._type](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/runtime/type.go#L18) 是 Go 语言类型的运行时表示，它其实是 [internal/abi.Type](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/internal/abi/type.go#L20) 的别名：

```go
type Type struct {
    Size_       uintptr // 类型占用的内存空间
    PtrBytes    uintptr 
    Hash        uint32  // 用于快速确定类型是否相等
    TFlag       TFlag   
    Align_      uint8   
    FieldAlign_ uint8   
    Kind_       uint8  
    // 判断当前类型的多个对象是否相等
    Equal func(unsafe.Pointer, unsafe.Pointer) bool
    GCData    *byte
    Str       NameOff 
    PtrToThis TypeOff 
}
```

#### itab 结构体

```go
// 每个 itab 结构体 均占 32 字节
type itab struct { 
    inter *interfacetype  // 接口类型
    _type *_type          // 具体类型
    hash  uint32          // 对 _type.hash 的拷贝，用于类型转换
    _     [4]byte
    // 一个用于动态派发的虚函数表，存储了一组函数指针
    // 虽然被声明成大小固定的数组，但在使用时会通过原始指针获取其中的数据
    fun   [1]uintptr      
}
```

### 类型转换

#### 使用指针类型实现接口

```go
type Duck interface {
    Quack()
}

type Cat struct {
    Name string
}

//go:noinline
func (c *Cat) Quack() {
    println(c.Name + " meow")
}

func main() {
    var c Duck = &Cat{Name: "draven"}
    c.Quack()
}
```

`Cat`结构体的初始化和赋值触发的类型转换过程对应的汇编代码经简化后如下：

```nasm
LEAQ    main..autotmp_3+40(SP), DX  ;; DX = SP + 40 + autotmp_3
MOVQ    $6, 8(DX)                   ;; (DX + 8) = 6
LEAQ    go:string."draven"(SB), SI  ;; SI = &"draven"
MOVQ    SI, (DX)                    ;; (DX) = SI
LEAQ    go:itab.*<unlinkable>.Cat,<unlinkable>.Duck(SB), AX
MOVQ    AX, main.c+24(SP)           ;; (SP + 24) = &go:itab.*.Cat,.Duck
MOVQ    DX, main.c+32(SP)           ;; (SP + 32) = &Cat
```

![20240806165711](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20240806165711.png)

SP + 24 ～ SP + 32 共同构成了 [runtime.iface](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/runtime/runtime2.go#L205) 结构体，因此可以作为`Quack()`方法的入参。[runtime.itab](https://github.com/golang/go/blob/4c50f9162cafaccc1ab1bc26b0dea18f124b536d/src/runtime/runtime2.go#L997) 结构体的`fun`字段位于其内部的第 24 字节，而`Duck`接口只有一个方法，因此`itab.fun[0]`存储的就是指向`Quack`方法的指针：

```nasm
MOVQ    24(AX), CX    ;; CX = AX.fun[0] = Cat.Quack
CALL    CX            
```

#### 使用结构体类型实现接口

```go
type Duck interface {
    Quack()
}

type Cat struct {
    Name string
}

//go:noinline
func (c Cat) Quack() {
    println(c.Name + " meow")
}

func main() {
    var c Duck = Cat{Name: "draven"}
    c.Quack()
}
```

`Cat`结构体的初始化和赋值触发的类型转换过程与上一节相似：

```nasm
LEAQ    go:string."draven"(SB), DX   ;; DX = &"draven"
MOVQ    DX, main..autotmp_2+40(SP)   ;; (SP + 40 + tmp) = DX
MOVQ    $6, main..autotmp_2+48(SP)   ;; (SP + 48 + tmp) = 6
LEAQ    go:itab.<unlinkable>.Cat,<unlinkable>.Duck(SB), DX
MOVQ    DX, main.c+24(SP)            ;; (SP + 24) = &go:itab..Cat,.Duck
LEAQ    main..autotmp_2+40(SP), DX   ;; DX = SP + 40 + tmp
MOVQ    DX, main.c+32(SP)            ;; (SP + 32) = DX
```

变量`c`调用接口方法`Quack()`对应的汇编代码经简化后如下。之所以代码中调用的是`Duck.Quack`但生成的汇编是`Cat.Quack`，是因为编译器会将一些需要 [动态派发](/posts/go-language-feature-note/#动态派发) 的方法改写成对目标方法的直接调用，以减少性能开销。

```nasm
MOVQ    main.c+24(SP), AX            ;; AX = &go:itab..Cat,.Duck
LEAQ    go:itab.main.Cat,main.Duck(SB), SI
CMPQ    AX, SI                       ;; 检查 c 的具体类型是否为 Cat
JEQ     111                          ;; 如果是，跳转到方法调用
JMP     139                          ;; 否则跳转到 runtime.panicdottypeI
CALL    main.Cat.Quack(SB)
```

如果我们在初始化变量时使用指针类型`&Cat{Name: "draven"}`，生成的汇编代码便与上一节几乎完全相同。

### 类型断言

基本概念详见：[类型断言：如何检测和转换接口变量的类型](https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/11.3.md)。

#### 非空接口转换为具体类型

```go
func main() {
    var c Duck = &Cat{Name: "draven"}
    switch c.(type) {
    case *Cat:
        cat := c.(*Cat)
        cat.Quack()
    }
}
```

Switch 语句生成的汇编指令会将目标类型的`Hash`字段与接口变量中的`itab.hash`字段进行比较。

#### 空接口转换为具体类型

```go
func main() {
    var c interface{} = &Cat{Name: "draven"}
    switch c.(type) {
    case *Cat:
        cat := c.(*Cat)
        cat.Quack()
    }
}
```

如果禁用编译器优化，上述代码会在类型断言时从`eface._type`中获取变量的具体类型，汇编指令仍然会使用目标类型的`Hash`字段与接口变量的类型进行比较。

### 动态派发

动态派发（Dynamic Dispatch）是在运行时选择具体多态操作（方法或者函数）执行的过程，它是面向对象语言中的常见特性。Go 语言虽然不是严格意义上的面向对象语言，但是接口的引入为它带来了动态派发这一特性。调用接口类型的方法时，如果在编译期间不能确认接口的类型，Go 语言会在运行时决定具体调用该方法的哪个实现。

```go
func main() {
    var c Duck = &Cat{Name: "draven"}
    c.Quack()
    c.(*Cat).Quack()
}
```

示例代码中，`main`函数调用了两次`Quack`方法：第一次以接口类型`Duck`调用，调用时需要经过运行时的动态派发，[前文](/posts/go-language-feature-note/#使用指针类型实现接口) 已分析过它的执行过程；第二次以具体类型`*Cat`调用，因此编译时便可以确定调用的函数`CALL main.(*Cat).Quack(SB)`。

两次方法调用对应的汇编指令差异就是动态派发带来的额外开销，这些额外开销在有低延时、高吞吐量需求的服务中是不能被忽视的。使用结构体实现接口带来的开销会大于使用指针实现，而动态派发在结构体上的表现非常差，这提醒我们应当尽量避免使用结构体类型实现接口。

使用结构体带来的巨大性能差异不仅是接口带来的问题，还主要因为 Go 语言在函数调用时是传值的，动态派发的过程只是放大了参数拷贝带来的影响。

## 反射

[reflect](https://golang.org/pkg/reflect/) 包实现了运行时的反射能力，能够让程序操作不同类型的对象。其中有两对非常重要的函数和类型，它们一一对应：

函数 [reflect.TypeOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L1160) 能获取任意变量的类型信息，返回一个接口类型 [reflect.Type](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L39)；

```go
type Type interface {
    Align() int
    FieldAlign() int
    Method(int) Method
    MethodByName(string) (Method, bool)  // 获取当前类型对应方法的引用
    NumMethod() int
    ...
    Implements(u Type) bool              // 判断当前类型是否实现了某个接口
    ...
}
```

函数 [reflect.ValueOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L3260) 能获取数据的运行时表示，返回一个结构体类型 [reflect.Value](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L39)。后者没有对外暴露的字段，但是提供了获取或者写入数据的方法。

```go
type Value struct {
    // 包含过滤的或者未导出的字段
}

func (v Value) Addr() Value
func (v Value) Bool() bool
func (v Value) Bytes() []byte
...
```

### 三大法则

反射作为一种元编程方式可以减少重复代码，但是过量使用反射会使我们的程序逻辑变得难以理解并且运行缓慢。Go 语言反射的三大法则为：

- 从`interface{}`变量可以反射出反射对象；
- 从反射对象可以获取`interface{}`变量；
- 要修改反射对象，其值必须可以设置。

![20240806165855](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20240806165855.png)

#### 第一法则

函数 [reflect.TypeOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L1160) 和 [reflect.ValueOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L3260) 的入参均为`any`类型（即`interface{}`的别名），所以类似`reflect.ValueOf(1)`这样的调用实际上首先完成了隐式的类型转换。上述两个函数是连接 Go 语言类型和反射类型的桥梁：

![20240806170008](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20240806170008.png)

一旦我们获取到了变量对应的反射对象，就能根据其类型调用不同的方法获取相关信息：

- 结构体：获取字段的数量并通过下标和字段名获取字段`StructField`；
- 哈希表：获取哈希表的`Key`类型；
- 函数或方法：获取入参和返回值的类型；
- …

#### 第二法则

[reflect.Value.Interface](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L1495) 可以将反射对象还原为`interface{}`类型的变量。如果想要将其还原为最原始状态，还需要进行显式的类型转换：

```go
v := reflect.ValueOf(1)
v.Interface().(int)
```

#### 第三法则

如果我们要更新一个`reflect.Value`，那么它“持有”的值一定是可以被更新的：

```go
func main() {
    i := 1
    v := reflect.ValueOf(i)
    v.SetInt(10)
    fmt.Println(i)
}

$ go run reflect.go
panic: reflect: reflect.Value.SetInt using unaddressable value
```

由于 Go 语言函数调用采用值传递，反射对象`v`和原始变量`i`之间没有任何关系，因此直接修改反射对象是无法改变原始变量的。正确的做法是：

```go
func main() {
    i := 1
    v := reflect.ValueOf(&i)
    v.Elem().SetInt(10)
    fmt.Println(i)
}

$ go run reflect.go
10
```

上述代码先获取指针`&i`对应的反射对象`v`，然后通过 [reflect.Value.Elem](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L1230) 方法得到指针指向的变量对应的反射对象，最后调用 [reflect.Value.SetInt](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L2397) 更新变量的值。整个流程的思路与下列代码相同：

```go
func main() {
    i := 1
    v := &i
    *v = 10
}
```

### 类型和值

Go 语言的`interface{}`类型在语言内部是通过 [reflect.emptyInterface](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L206) 结构体表示的：

```go
type emptyInterface struct {
    typ  *rtype           // 变量类型
    word unsafe.Pointer   // 指向内部封装的数据
}
```

函数 [reflect.TypeOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L1160) 会将传入的变量隐式地转换为`reflect.emptyInterface`类型并获取其中存储的类型信息 [reflect.rtype](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L296)：

```go
func TypeOf(i any) Type {
    eface := *(*emptyInterface)(unsafe.Pointer(&i))
    return toType((*abi.Type)(noescape(unsafe.Pointer(eface.typ))))
}

func toType(t *abi.Type) Type {
    if t == nil {
        return nil
    }
    return toRType(t)
}

func toRType(t *abi.Type) *rtype {
    return (*rtype)(unsafe.Pointer(t))
}
```

`reflect.rtype`是一个实现了`reflect.Type`接口的结构体，其 [String](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L548) 方法可以帮助我们获取当前类型的名称：

```go
func (t *rtype) String() string {
    s := t.nameOff(t.t.Str).Name()
    if t.t.TFlag&abi.TFlagExtraStar != 0 {
        return s[1:]
    }
    return s
}
```

函数 [reflect.ValueOf](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L3260) 的实现也非常简单，它调用 [reflect.unpackEface](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L156) 从接口中获取`reflect.Value`结构体：

```go
func ValueOf(i any) Value {
    if i == nil {
        return Value{}
    }
    return unpackEface(i)
}

func unpackEface(i any) Value {
    // 类型转换
    e := (*emptyInterface)(unsafe.Pointer(&i))  
    t := e.typ
    if t == nil {
        return Value{}
    }
    f := flag(t.Kind())                         
    if t.IfaceIndir() {
        f |= flagIndir
    }
    // 将具体类型和指针包装成 reflect.Value 结构体
    return Value{t, e.word, f}
}
```

实际上，当我们要将一个变量转换成反射对象时，其类型和值在编译期间就会被转换成`interface{}`。

### 更新变量

我们可以调用 [reflect.Value.Set](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L2318) 来更新反射变量的值：

```go
func (v Value) Set(x Value) {
    v.mustBeAssignable()          // 检查当前反射对象是否是可以被设置的
    x.mustBeExported()            // 检查新反射对象的字段是否是对外公开的
    var target unsafe.Pointer     // 定义一个指向目标的指针
    if v.kind() == Interface {
        target = v.ptr
    }
    // 调整 x 的类型，使之能够分配给 v，返回值将覆盖 x
    x = x.assignTo("reflect.Set", v.typ, target)  
    if x.flag&flagIndir != 0 {
        if x.ptr == unsafe.Pointer(&zeroVal[0]) {
            // 若 x 指向的是零值则清除 v 指向的内存
            typedmemclr(v.typ(), v.ptr)
        } else {
            // 将 x.ptr 指向的值复制到 v.ptr 指向的值
            typedmemmove(v.typ(), v.ptr, x.ptr)
        }
    } else {
        // 若 x 不是间接引用的，则直接将 x.ptr 赋给 v.ptr
        *(*unsafe.Pointer)(v.ptr) = x.ptr
    }
}
```

其中最为重要的函数是 [reflect.Value.assignTo](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L3323)：

```go
func (v Value) assignTo(context string, dst *rtype, target unsafe.Pointer) Value {
    // 注意这里的 dst 是 Set 函数里的 v.typ，而 v 是 Set 函数里的 x
    ...
    switch {
    // 若当前反射对象的类型可以直接被目标对象替换，则返回目标反射对象
    case directlyAssignable(dst, v.typ):
        ...
        return Value{dst, v.ptr, fl}
    // 若当前反射对象是接口且目标对象实现了接口，则将目标对象简单包装成接口值
    // implements 的实现详见下一节
    case implements(dst, v.typ):
        if v.Kind() == Interface && v.IsNil() {
            return Value{dst, nil, flag(Interface)}
        }
        x := valueInterface(v, false)
        if dst.NumMethod() == 0 {
            *(*any)(target) = x        
        } else {
            ifaceE2I(dst, x, target)
        }
        return Value{dst, target, flagIndir | flag(Interface)}
    }
    panic(context + ": value of type " + stringFor(v.typ()) + " is not assignable to type " + stringFor(dst))
}
```

### 实现协议

[reflect.rtype.Implements](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L1247) 方法可以用于判断某些类型是否遵循特定的接口，示例代码如下：

```go
type CustomError struct{}

func (*CustomError) Error() string {
    return ""
}

func main() {
    // 获取内置接口类型 error 的反射类型比较复杂
    // (*error)(nil) 是一个指向 error 的空指针
    // reflect.TypeOf((*error)(nil)) 获取空指针的类型信息
    // Elem() 获取 *error 指针指向的元素类型，即 error 接口类型本身
    typeOfError := reflect.TypeOf((*error)(nil)).Elem()
    // 获取结构体的反射类型则很简单
    customErrorPtr := reflect.TypeOf(&CustomError{})
    customError := reflect.TypeOf(CustomError{})

    fmt.Println(customErrorPtr.Implements(typeOfError)) // #=> true
    fmt.Println(customError.Implements(typeOfError))    // #=> false
}
```

该函数会检查传入的类型是不是接口，然后在参数符合条件的情况下调用私有方法 [reflect.implements](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/type.go#L1277)：

```go
func implements(T, V *abi.Type) bool {
    t := (*interfaceType)(unsafe.Pointer(T))
    // 如果接口中没有任何方法，则说明 t 是空接口
    // 任何类型都实现了空接口，因此返回 true
    if len(t.methods) == 0 {
        return true
    }
    ...
    v := V.uncommon()
    i := 0
    vmethods := v.methods()
    // 遍历接口和类型的方法
    // 方法均按字母顺序存储，复杂度为 O(N)
    for j := 0; j < int(v.Mcount); j++ {
        tm := &t.Methods[i]
        tmName := t.nameOff(tm.Name)
        vm := vmethods[j]
        vmName := nameOffFor(V, vm.Name)
        if vmName.Name() == tmName.Name() && typeOffFor(V, vm.Mtyp) == t.typeOff(tm.Typ) {
            ...
            if i++; i >= len(t.Methods) {
                return true
            }
        }
    }
    return false
}
```

### 方法调用

示例代码使用反射来执行`Add(0, 1)`函数：

```go
func Add(a, b int) int { return a + b }

func main() {
    // 获取函数 Add 对应的反射对象
    v := reflect.ValueOf(Add)  
    if v.Kind() != reflect.Func {
        return
    }
    t := v.Type()
    // NumIn 获取函数的入参个数
    argv := make([]reflect.Value, t.NumIn())  
    for i := range argv {
        if t.In(i).Kind() != reflect.Int {
            return
        }
        // 将切片中的元素设置为索引（即 0，1）对应的反射对象
        argv[i] = reflect.ValueOf(i)  
    }
    // 调用 Add 反射对象的 Call 方法并传入参数列表
    result := v.Call(argv)            
    // Add 只有一个返回值，因此 result 切片中只有一个元素
    if len(result) != 1 || result[0].Kind() != reflect.Int {
        return
    }
    fmt.Println(result[0].Int())
}
```

使用反射来调用方法非常复杂，原本只需要一行代码就能完成的工作，现在需要十几行代码才能完成，但这也是在静态语言中使用动态特性必须付出的成本。

其中，[reflect.Value.Call](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L377) 是运行时调用方法的入口，它通过两个`mustBe`开头的方法确定了当前反射对象的类型是函数以及可见性，随后调用 [reflect.Value.call](https://github.com/golang/go/blob/8c8adffd5301b5e40a8c39e92030c53c856fb1a6/src/reflect/value.go#L400) 完成方法调用：

```go
func (v Value) Call(in []Value) []Value {
    v.mustBe(Func)
    v.mustBeExported()
    return v.call("Call", in)
}
```

这个私有方法的执行过程会分成以下几部分：

1. 检查输入参数以及类型的合法性；
2. 将传入的参数切片`in`设置到寄存器和栈；
3. 通过函数指针和输入参数调用函数；
4. 从寄存器和栈上获取函数的返回值。

### 应用场景

#### 实现通用函数

反射可以用于实现通用函数，这些函数可以接受任意类型的参数：

```go
func printTypeAndValue(x interface{}) {
    v := reflect.ValueOf(x)
    t := v.Type()
    fmt.Printf("Type: %s, Value: %v\n", t, v.Interface())
}

func main() {
    printTypeAndValue(100)       // Type: int, Value: 100
    printTypeAndValue("Hello")   // Type: string, Value: Hello
    printTypeAndValue(3.14)      // Type: float64, Value: 3.14
}
```

#### 动态调用函数

反射可以根据传入的函数名来调用相应的函数：

```go
func hello() {
    fmt.Println("Hello, world!")
}

func goodbye() {
    fmt.Println("Goodbye!")
}

func callFunc(funcName string) {
    funcs := map[string]interface{}{
        "hello":   hello,
        "goodbye": goodbye,
    }
    f := reflect.ValueOf(funcs[funcName])
    f.Call(nil)
}

func main() {
    funcName := "hello" // 这个值可以是来自用户输入，或者其他动态来源
    callFunc(funcName)
}
```

### 反射的缺点

- 与反射相关的代码，经常是难以阅读的。在软件工程中，代码可读性也是一个非常重要的指标。
- Go 语言作为一门静态语言，编码过程中，编译器能提前发现一些类型错误，但是对于反射代码是无能为力的。所以包含反射相关的代码，很可能会运行很久，才会出错，这时候经常是直接 panic，可能会造成严重的后果。
- 反射对性能影响还是比较大的，比正常代码运行速度慢一到两个数量级。所以，对于一个项目中处于运行效率关键位置的代码，尽量避免使用反射特性。
