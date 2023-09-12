---
title: 从Java类比学习Go之基础语法
author: daviswujiahao
date: 2022-01-01 11:33:00 +0800
categories: [技术]
tags: [编程语言,Golang]
math: true
mermaid: true
---

# 前言

## 什么是Go语言

与 Java 相同，Go 是一门高性能，高并发，语法简单，学习曲线平缓的强类型和静态类型语言，其由 Google 主导开发，其拥有丰富的标准库，完善的工具链，支持快速编译，跨平台且支持垃圾回收（GC）；
与 Java 不同的是，其并不是一门虚拟机语言，不需要通过中间代码表示（例如 JVM Bytecode）和虚拟机（VM）支持代码运行，其以直接将目标代码静态链接并编译到目标平台的形式跨平台。
虽然 Go 和 C/C++ 类似，人们也经常讲 Go 讲述为 “更好的 C/C++”，但 Go 的竞争领域并不是 C/C++ 所适合的领域，相反，Go 更适合 Java 所适合的 Web 工程等领域。理论上，Go 可以提供比 Java 更好的性能和吞吐量。

## 开发环境

要想开发 Go 程序，则需要 Go 开发环境，可以前往 [Go 官网](https://link.zhihu.com/?target=https%3A//go.dev/) 并遵循 [安装文档](https://link.zhihu.com/?target=https%3A//go.dev/doc/install) 安装对应平台的 Go 开发环境。这些开发环境包括 Go 编译器，工具和库。和 Java 不同的是，不存在类似于 JRE（Java Runtime Environment）一样的东西，用户可以直接运行编译后对应平台的可执行文件，无须运行时支持。

接下来，我们当然还需要 IDE 来便捷我们的开发。有两种主流 IDE 可选：VSCode 和 GoLand。前者是由微软开发的开源代码编辑器，后者则是由 Jetbrains 公司开发，基于著名 Java IDE IntelliJ IDEA 构建的功能强大的 IDE。此两种 IDE 的区别是，前者更像手动挡，后者则是自动挡。对于进阶需求，VSCode 为你带来的可自定义性会更强。

## HelloWorld

```Go
package main

import (
    "fmt"
)

func main(){
    fmt.Println("hello world")
}
```

以上是使用 Go 语言输出 Hello World 的代码。可以看出，Go 语言的入口点是 main 函数（注意 Go 语言同时存在函数和方法，前者可以认为是 Java 的静态方法或者 Rust 的关联函数，后者可以认为是非静态方法）；除此之外，fmt.Println 类似于 System.out.println，可将一段数据打印在标准输出流中。

应当注意到，在 Go 语言中，; 不是必要的，当一行中只存在一个语句时，则不必显式的为语句末添加 ;。

你可能注意到，Println 中的 P 是大写的，你可能会主观的认为这是 Go 语言的命名习惯，就像 C# 开发者那样。但实际上，在 Go 语言中，函数 / 方法**首字母大写意味着可被其他包调用**，否则只能在该包被调用，这就类似于 Java 中 public 和 protected 访问修饰符的区别。

# 基本语法

## 变量

### 声明与定义

```Go
// 与 Java 不同，Go 语言的变量是类型后置的，你可以这样创建一个类型为 int 的变量：
var a int = 1
// 允许在同一行声明多个变量：
var b,c int = 1, 2

// Go 支持变量类型自动推断，也就是说，当我们立即为一个变量进行初始化时，其类型是可以省略的
var d = true

// 如果我们未为一个变量初始化，则必须显式指定变量类型，此时，变量会被以初始值自动初始化
var e float64 // got 0

// 可以通过 := 符号以一种简单的方式（也是实际上最常用的方式）声明一个变量：
f := 3.2 // 等价于 var f = 3.2

// 最后，可以使用 const 关键字代替 var 关键字来创建一个常量（不可变变量）：
const h string = "constant"

```

### nil 与零值

只声明未赋值的变量，其值为 nil。类似于 java 中的 “null” 。
没有明确初始值的变量声明会被赋予它们的 零值。
零值是：

- 数值类型为 ​​0​​，
- 布尔类型为 ​​false​​，
- 字符串为 ​​""​​（空字符串）。

## 方法

Go 中方法的定义
使用 func 关键字来定义一个方法，后面跟方法名，然后是参数，返回值（如果有的话，没有返回值则不写）。

```GO
package main
import "fmt"

func main() {
  fmt.Println(add(3, 5)) //8
  var sum = add(3, 5)
}
func add(a int, b int) int{
  return a+b;
}
```
Go 函数与其他编程语言一大不同之处在于支持多返回值，这在处理程序出错的时候非常有用。例如，如果上述 ​​add​​ 函数只支持非负整数相加，传入负数则会报错。

```GO
//返回值只定义了类型 没有定义返回参数
func add(a, b int) (int, error) {
  if a < 0 || b < 0 {
    err := errors.New("只支持非负整数相加")
    return 0, err
  }
  a *= 2
  b *= 3
  return a + b, nil
}

//返回值还定义了参数 这样可以直接return 并且定义的参数可以直接使用 return时只会返回这两个参数
func add1(a, b int) (z int, err error) {
  if a < 0 || b < 0 {
    err := errors.New("只支持非负整数相加")
    return //实际返回0 err 因为z只定义没有赋值 则nil值为0
  }
  a *= 2
  b *= 3
  z = a + b
  return //返回 z err
}

// 函数调用
func main() {
  x, y := -1, 2
  z, err := add(x, y)
  if err != nil {
    fmt.Println(err.Error())
    return
  }
  fmt.Printf("add(%d, %d) = %d\n", x, y, z)
}
```

### 变长参数

```Go
func myfunc(numbers ...int) {
  for _, number := range numbers {
  fmt.Println(number)
  }
}

slice := []int{1, 2, 3, 4, 5}
//使用...将slice打碎传入
myfunc(slice...)

```

### 可见性

在 Go 语言中，无论是变量、函数还是类属性和成员方法，它们的可见性都是以包为维度的，而不是类似传统面向编程那样，类属性和成员方法的可见性封装在所属的类中，然后通过 ​​private​​、​​protected​​ 和 ​​public​​ 这些关键字来修饰其可见性。
Go 语言没有提供这些关键字，不管是变量、函数，还是自定义类的属性和成员方法，它们的可见性都是根据其首字母的大小写来决定的，如果变量名、属性名、函数名或方法名首字母大写，就可以在包外直接访问这些变量、属性、函数和方法，否则只能在包内访问，因此 Go 语言类属性和成员方法的可见性都是包一级的，而不是类一级的。

### 指针

指针，其实就是一个在内存中实际的 16 进制的地址值，引用变量的值通过此地址去内存中取出对应的真实值，格式类似于C/C++

```Go
func main() {
  i := 0
  //使用&来传入地址
  fmt.Println(&i) //0xc00000c054

  var a, b int = 3 ,4
  //传入 0xc00000a089 0xc00000a090
  fmt.Println(add(&a, &b))
}

//使用*来声明一个指针类型的参数与使用指针
func add(a *int, b *int) int {
  //接收到 0xc00000a089 0xc00000a090
  //前往 0xc00000a089位置查找具体数据 并取赋给x
  x := *a
  //前往 0xc00000a090位置查找具体数据 并取赋给y
  y := *b
  return x+y
}

```

## 流程控制

### 选择语句

Go 支持 if，else if，else, switch 进行选择控制。

```Go

// if else
if 7%2 == 0 {
    fmt.Println("7 is even")
} else {
    fmt.Println("7 is odd")
}
if num := 9; num < 0 {
    fmt,Println(num,"is negative")
} else if num < 10 {
    fmt.Println(num, "has 1 digit")
} else {
    fmt.Println(num, "has mutiple digits")
}

```

其他语言中，if（其他类似）后应当紧跟一个括号()，括号内才是表达式，但是在 Go 中，这个括号是可选的，也建议不要使用括号。
要注意的是，if 表达式后面的大括号{}是必需的，即使是对于单行语句块，您也必须添加括号，而不能像其他语言那样直接省略。


```Go

// switch
a := 2
switch a {
    case 0, 1:
        fmt.Println("zero or one")
    case 2:
        fmt.Println("two")
    default:
        fmt.Println("other")
}

// 也可以直接省略 switch 后的变量，来获得一个更加宽松的 switch 语句
t := time.Now()
switch {
    case t.Hour() < 12:
        fmt.Println("It's before noon")
    default:
        fmt.Println("It's after noon")
}
```
与其他语言恰好相反，switch 语句中每个 case 的 break 是隐式存在的，也就是说，每个 case 的逻辑会在执行完毕后立刻退出，而不是跳转到下一个 case。要想跳转到下一个 case，则应该使用 fallthrough 关键字：

```Go
v := 42
switch v {
case 100:
    fmt.Println(100)
    fallthrough
case 42:
    fmt.Println(42)
    fallthrough
case 1:
    fmt.Println(1)
    fallthrough
default:
    fmt.Println("default")
}
// Output:
// 42
// 1
// default

// fallthrough 关键字只能存在于 case 的末尾
switch {
case f():
    if g() {
        fallthrough // 不起作用，
    }
    h()
default:
    error()
}
```

### 循环语句

在 Go 语言中不区分 for 和 while。
```Go
// 通过这样的方式创建一个最普遍的 for 语句：
for j := 7; j < 9; j++ {
    fmt.Println(j)
}

// 或者，将 for 语句中的三段表达式改为一个布尔值表达式，即可得到一个类似于其它语言的 while 语句
i := 1
for i <= 3 {
    fmt.Println(i)
    i = i + 1
}

// 又或者，不为 for 语句填写任何表达式，你将得到一个无限循环，除非使用 break 关键字跳出循环，否则这个循环永远也不会停止，这看起来有些类似于 Java 的 while(true) {} 或是 Rust 的 loop {}：
for {
    fmt.Println("loop")
}
```

当然，我们也可以使用 for range 循环的方式来遍历一个数组，切片，集合乃至映射（Map）。
当我们使用 for range 语句遍历一个数组，切片或是集合的时候，我们将得到该集合元素的索引（idx）和对应值（num）：

```Go
nums := []int{2, 3, 4}
sum := 0
for idx, num := range nums {
    fmt.Println("range to index:", idx)
    sum += num
}
// Will got following output:
// range to index: 0
// range to index: 1
// range to index: 2
// sum: 9
fmt.Println("sum:", sum)

```

当我们遍历一个 Map 时，将得到键（k）和值（v）：

```Go
m := make(map[string]int)
m["hello"] = 0
m["world"] = 1
// If key and value both needed
for k, v := range m {
    // Will got following output:
    // key: hello, value: 0
    // key: world, value: 1
    fmt.Printf("key: %v, value: %v\n", k, v)
}
// Or only need key
for k := range m {
    // Will got following output:
    // key: hello
    // key: world
    fmt.Printf("key: %v", k)
}
```

如果我们不需要循环中的某个值，则可以使用 _ 符号代替变量名来遮蔽该变量（其他语言也有类似的做法，但是在 Go 中，此操作是必须的，因为未被使用的变量或导入会被 Go 编译器认为是一个 error）：

```Go
// When only `v` variable needed
for _, v := range m {
    //... 
}

```
当然，Go语言中，break 和 continue 都是支持的，其用法和其他语言完全相同。

## 集合

### 数组

```Go
// 声明一个指定长度的数组
var a [5]int
a[4] = 100

// 使用 := 进行声明
b := [5]int{1, 2, 3, 4, 5}

// 声明多维数组
var twoD [2][3]int

// 当一个数组未被显式初始化元素值时，将采用元素默认值填充数组。

// 访问一个超出数组长度的索引，编译器将会拒绝为我们编译，并返回一个编译错误
fmt.Println(b[5]) // error: invalid argument: index 5 out of bounds [0:5]


```

### 切片

数组是定长的，因此在实际业务中使用的并不是很多，因此，更多情况下我们会使用切片代替数组。
就像它的名字一样，切片（slice）某个数组或集合的一部分，切片是可变容量的，其工作原理类似于 Java 的 ArrayList，当切片容量不足时，便会自动扩容然后返回一个新的切片给我们。

```Go

// 声明一个切片
s := make([]string, 3)

// 直接指定一个切片的长度和容量
// 切片的 长度 (length) 和 容量 (capacity) 是两个完全不同的东西，前者才是切片实际的长度，后者则是一个阈值，当切片长度达到该阈值时才会对切片进行扩容。
s2 := make([]string, 0, 10)

// 可以使用 append 方法为数组添加新的元素：
s = append(s, "d")
s = append(s, "e", "f") // 并返回更新后的切片。

// 使用 copy 方法将一个切片内的元素复制到另一个切片中：
c := make([]string, len(s))
copy(c, s)

// 但是不同的是，当我们试图越界访问一个切片时，编译器并不会给我们一个错误（因为切片的长度是不确定的），然而，这会得到一个 panic，并使程序直接结束运行：
fmt.Println(s[6]) // panic: runtime error: index out of range [6] with length 6

// 使用以下切片操作从数组和切片中截取元素：
fmt.Println(s[2:5]) // [c d e] 将返回一个新的切片

```

### 键值对

映射（Map）是一个无序 1 对 1 键值对。

```Go
// 使用如下方式声明一个 Map
m := make(map[string]int)

// 可以提前初始化 Map 内的值
m2 := map[string]int{"one" : 1, "two" : 2}

// 使用类似于数组和切片的赋值语法为 Map 赋值
m["one"] = 1
m["two"] = 2

// 使用 len 方法获得一个 Map 内包含键值对的长度
fmt.Println(len(m)) // 2

// 使用和数组和切片类似的方式从切片中获得一个值
fmt.Println(m["one"]) // 1
// 这种写法是非常不好的，因为，当我们试图访问一个不存在的 key，那么 Map 会给我们返回一个初始值, 正确的方式为
r, ok := m["unknown"]
fmt.Println(r, ok) // 0 false

// 删除元素
delete(m, "one")

```