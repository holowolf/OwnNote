## 不安全指针unsafe.Pointer使用

### 前言

`go的指针不支持指针运算和转换`

首先,Go是一门静态语言,所有的变量都必须为标量类型.不同类型不能够进行赋值、计算等跨类型的操作.那么指针也对应的类型,
也在compile的静态类型检查的范围内.同时静态语言是强类型,一旦定义无法更改.

#### 错误示例
```go
package unsafe_pointer

import "github.com/sirupsen/logrus"

func TestUnsafePoint()  {
	num := 555
	numPtr := &num

	float := (*float32)(numPtr)
	logrus.Info(float)
}
```

输出结果, 强制转换*int为*float指针报错
```go
Cannot convert expression of type '*int' to type '*float32' 
```

#### unsafe

对于错误提示可以使用`unsafe`标准库解决.它是一个神奇的包, 在官方诠释中,有如下概念:
* 围绕Go程序内存安全及类型的操作
* 很可能是不会移植的
* 不受Go 1兼容性指南的保护

简单来说就是不推荐使用,但在特殊情况下可以使用.

#### Pointer

`unsafe.Pointer` 它表示任意类型且可寻址的指针值, 可以在不同的指针类型之间进行转换(类型C语言的void *的用途)

其包含四种核心操作
* 任何类型的指针值都可以转换为Pointer
* Pointer可以转换为任何类型的指针值
* uintptr 可以转换为Pointer
* Pointer 可以转换为 uintptr

```go
package unsafe_pointer

import (
	"github.com/sirupsen/logrus"
	"unsafe"
)

func TestUnsafePoint()  {
	num := 555
	numPtr := &num
	float := (*float32)(unsafe.Pointer(numPtr))
	logrus.Info(float)
}
```

输出结果:
```
time="2020-07-03T10:45:49+08:00" level=info msg=0xc0000160e0
```

在上述代码中, 通过`unsafe.Pointer`的特性对该指针变量进行了修改,就可以完成任意类型(*T)的指针转换

需要注意的是,这时无法对变量进行操作或访问.因为不知道该指针指向的东西具体是什么类型.不知道是什么类型,又如何进行解析呢.无法解析也就无法变更.

#### Offsetof

普通的指针变量进行了修改, 那么它是否能做更复杂一点

```go
package unsafe_pointer

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"unsafe"
)

type Num struct {
	i string
	j int64
	k bool
	o float64
}

func UnsafePoint()  {
	num := Num{i: "NewTest", j: 1}
	nPointer := unsafe.Pointer(&num)

	niPointer := (*string)(nPointer)
	*niPointer = "break"

	njPointer := (*int64)(unsafe.Pointer(uintptr(nPointer) + unsafe.Offsetof(num.j)))
	*njPointer = 312

	nkPointer := (*bool)(unsafe.Pointer(uintptr(nPointer) + unsafe.Offsetof(num.k)))
	*nkPointer = true

	noPointer := (*float64)(unsafe.Pointer(uintptr(nPointer) + unsafe.Offsetof(num.o)))
	*noPointer = 333.333
	logrus.Info(fmt.Sprintf("Num i:%s, j:%d k:%v o:%v", num.i, num.j, num.k, num.o))
}
```

输出
```go
time="2020-07-03T11:31:51+08:00" level=info msg="Num i:break, j:312 k:true o:333.333"
```

结构体概念剖析
* 结构体成员变量在内存存储上是一段连续的内存
* 结构体的初始地址就是第一个成员变量的内存地址
* 基于结构体的成员地址去计算偏移量.就能够得出其他成员变量的内存地址

回来看看执行流程：
* 修改`n.i`的值: 由于i为第一个成员变量,因此不需要进行偏移量计算, 直接取出指针后转换为`Pointer`, 再强制转换为字符串类型的指针值即可.
* 修改`n.j`的值: 由于j为第二个成员变量,需要进行偏移量计算,才可以对其内存地址进行修改.在进行了偏移运算后,当前地址已经指向第二个成员变量,接着重复转换赋值即可.

使用以下方法(完成偏移量计算的目标):
1. uintptr: `uintptr`是GO内置的无符号整形用于存放指针,可以用于指针运算
```go
type uintptr uintptr
```

2. unsafe.Offsetof:返回变量的字节大小,也就是文本用到的偏移量大小.需要注意的是入参ArbitraryType表示任意类型, 并非定义的`int`它实际作用是一个占位符
```go
func Offsetof(x ArbitraryType) uintptr
```

在这一部分, 其实就是用了`pointer`的第三、第四点特性.这时候就已经可以对变量进行操作了

#### 错误示例

```go
package unsafe_pointer

import (
	"github.com/sirupsen/logrus"
	"unsafe"
)

func UnsafePointError()  {
	n := Num{i:"aaa", j: 222, k:false, o:2.22}
	nPointer := unsafe.Pointer(&n)
	// 虽然能运行 但不建议 因为uintptr类型不能存储在临时变量中, 因为从gc的角度来看,
	// uintptr类型的临时变量只是一个无符号的整数,并不知道它是一个指针地址
	ptr := uintptr(nPointer)
	njPointer := (*int64)(unsafe.Pointer(ptr + unsafe.Offsetof(n.j)))
	*njPointer = 3333
	logrus.Info(n)
}
```
所以当ptr变量在满足一定条件后可能会被垃圾回收掉,那么接下来的操作就会成为野指针操作.

### 总结

简洁回顾两个知识点.
1. `unsafe.Pointer`可以让你的变量在不同的指针类型转来转去,也可以表示为了寻址的指针类型.
2. `uintptr`常和`unsafe.Pointer`配合,用于做指针运算.
3. 如果没有特殊必要不建议使用`unsafe`标准库, 它并不安全.