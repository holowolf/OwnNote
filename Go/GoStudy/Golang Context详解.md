# Golang's context 详解

### 1. 前言

Context是Golang官方定义的package, 它定义了Context类型, 里面包含了Deadline/Done/Err方法以及绑定到Context上的成员变量值Value, 具体如下:

```go
type Context interface {
    // 返回Context的超时时间(超时返回场景)
    Deadline() (deadline time.Time, ok bool)
    // 在Context超时或取消时(即结束了) 返回一个关闭的channel
    // 即如果当前Context超时或取消时, Done方法会返回一个channel, 然后其他地方就可以通过判断Done方法是否有返回(channel).
    // 如果有说明Context结束
    // 故广播其他相关本Context已经结束, 请做相关处理.
    Done <-chan struct{}
    // 返回Context取消的原因
    Err() error
    // 返回Context相关数据
    Value(key interface{}) interface{}
}
```

#### Context优势

可以字面意思可以理解为上下文, 比较熟悉的有进程/线程上下文, 关于golang中的上下文,一句话概括:
goroutine的相关环境快照, 其中包含函数调用以及设计的相关的变量值.
通过Context可以区分不同的goroutine请求, 因为golangServers中, 每个请求都是在单个goroutine中完成.

proto文件生成的代码, 接口函数第一个统一参数就是ctx context.Context接口.

Context通常被译为上下文, 他是一个比较抽象的概念, 每个Goroutine在执行之前, 都要先知道程序当前的执行状态,
通常将这些执行状态封装在一个Context变量中, 传递给要执行的Goroutine中, 上下文则几乎已经成为传递与请求同生存周期变量的标准方法.
在网络编程下，当收到一个网络请求Request, 处理Request时, 我们可能需要开启不同的Goroutine来获取数据与逻辑处理.
即一个请求的Request, 会在多个Goroutine中处理, 而这些Goroutine可能需要共享Request的一些信息.同时当Request被取消或超时时的时候,
所有从这个Request创建的所有Goroutine页应该被结束.


### 2.为什么使用Context

由于在golang servers中, 每个request都是在单个goroutine中完成, 
并且在单个goroutine(不妨成为A)中会有请求其他服务(启动另一个goroutine(称之为B)去完成的场景),
就会涉及多个Goroutine之间的调用. 如果某一个时刻请求其他服务被取消或超时, 则作为深陷其中的当前goroutineB需要立即退出,
然后系统才可回收B所占用的资源.
即一个request中通常包含多个goroutine, 这些goroutine之间通常会有交互

![image](https://note.youdao.com/yws/res/4022/08275A93970C4AE991C7161A81F880DD)

那么, 如何有效管理这些goroutine成为一个问题(主要是退出通知和元数据传递的问题), Google的解决方式是Context机制,
相互调用的goroutine之间通过传递context变量保持关联, 这样在不用暴露各goroutine内部实现细节前提下, 有效控制各goroutine运行.

![image](https://note.youdao.com/yws/res/4025/9284233E0F8A4098B2B0DD8367B50BE1)

如此一来, 通过传递Context就可以追踪goroutine调用树, 并在这些调用树之间传传递和元数据.
虽然goroutine之间是平行的, 没有继承关系, 但是Context设计是包含父子关系的, 这杨更好的描述goroutine调用之间的树型关系.

### 3.如何使用Context

生成一个Context方法有两类:

##### 3.1 顶层Context: Background
要创建Context树, 首先要创建根节点

```go
// 返回一个空的Context. 它作为所有由此继承Context的根节点
func Background() Context
```

该Context通常由接收request的第一个goroutine创建, 它不能被取消、没有值、也没有过期时间, 常作为处理request的顶层context存在.

#### 3.2 下层Context: WithCancel/WithDeadline/WithTimeout

有了根节点之后, 接下来就是创建子孙节点, 为了可以很好的控制子孙节点, Context包提供的创建方法均是带有第二个返回值(CancelFunc类型), 
它相当于一个Hook,在子goroutine执行过程中,可以通过触发Hook来达到控制子Goroutine的目的(通常是取消, 即让其停下来), 
再配合Context提供的Done方法, 子goroutine可以检查自身是否被父级节点Cancel:
```go
select {
    case <-ctx.Done();
    // do some clean...
}
```
注: 父节点Context可以主动通过调用cancel方法取消子节点Context,而子节点Context只能被动等待. 同时父节点context自身一旦被取消(如其上级节点Cancel),
其下所有的子节点Context均会自动被取消

有三种创建方法:
```go
// 带cancel返回值的Context, 一旦cancel被调用, 即消失该创建context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 带有效cancel返回值的Context,即必须到达指定时间点调用的cancelFuc方法才被执行
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

// 带超时时间cancel返回值的Context, 类似Deadline, 前者是时间点, 后者为时间间隔.
// 相当于WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

Simple Example

```go
package context

import (
	"context"
	"fmt"
	"github.com/sirupsen/logrus"
	"time"
)

func SimpleContext()  {
	logrus.Info("start ...")
	someHandler()
	logrus.Info("end ...")
}


func someHandler() {
	// 创建继承Background的子节点Context
	ctx, cancel := context.WithCancel(context.Background())
	go work(ctx)

	// 模拟程序运行 sleep 5
	time.Sleep(5 * time.Second)
	cancel()
}


func work(ctx context.Context) {
	var i = 1
	for {
		time.Sleep(1 * time.Second)
		select {
			case <-ctx.Done():
				fmt.Println("done")
				return
			default:
				fmt.Printf("work %d seconds: \n", i)
		}
		i++
	}
}
```

输出结果
```
INFO[0000] start ...                                    
work 1 seconds: 
work 2 seconds: 
work 3 seconds: 
work 4 seconds: 
INFO[0005] end ...     
```
`done`并没有被打印

```go
package context

import (
	"context"
	"fmt"
	"github.com/sirupsen/logrus"
	"time"
)

func TimeoutMain()  {
	logrus.Info("start...")
	timeoutHandler()
	logrus.Info("end...")
}

func timeoutHandler()  {
	// 创建继承Background的子节点Context
	con, cancelFunc := context.WithTimeout(context.Background(), 3*time.Second)
	go timeoutWork(con)
	time.Sleep(time.Second * 10)
	cancelFunc()
}

// 每一秒的work 判断ctx是否done 如果有就退出
func timeoutWork(ctx context.Context) {
	var i = 1
	for {
		time.Sleep(1 * time.Second)
		select {
		case <-ctx.Done():
			logrus.Info("done")
			return
		default:
			logrus.Info(fmt.Sprintf("work %d seconds! \n", i))
		}
		i++
	}
}
```

输出结果

```
INFO[0000] start...
INFO[0001] work 1 seconds!
INFO[0002] work 2 seconds!
INFO[0003] done
INFO[0010] end...
```

### 4.context的设计

#### 4.1 context像病毒一样扩散

一旦代码中某处用到了Context, 传递Context变量(通常作为函数的第一个参数) 会像病毒一样蔓延在各处调用他的地方.
比如在一个request中实现数据库事务或者分布式日志记录, 创建的context, 会作为参数传递到任何有数据库操作或日志记录需求的函数代码处.
即每一个相关函数都必须增加一个context.Context类型的参数. 且作为第一个参数, 这对无关代码完全是侵入式的.


#### 4.2 context不仅仅只是cancel信号
Context机制最核心的功能在goroutine之间传递cancel信号, 但他的实现是不完整的.

Cancel可以细分为主动与被动两种, 通过传递context参数, 让调用goroutine可以主动的cancel被调用goroutine可以主动cancel被调用goroutine.
但是如何得知被调用goroutine什么时候执行完毕, 这部分Context机制是没有实现的, 而现实中的确又有一些这样的场景, eg:一个组装数据的goroutine必须
等待其他goroutine完成才可执行,这里context明显不够用了, 必须借助sync.WaitGroup

```go
func serve(l net.Listener) error {
    var wg sync.WaitGroup
    var conn net.Conn
    var err error
    for {
        conn, err = l.Accept()
        if err != nil {
            break
        }
        wg.Add(1)
        go func(c net.Conn) {
            defer wg.Done()
            handle(c)
        }(conn)
    }
    wg.Wait()
    
}
```

#### 4.3 context.value
context.Value相当于goroutine的TLS(Thread Local Storage), 但它不是静态安全类型, 任何结构体变量必须作为字符串形式存储.
同时,context都会在其中定义变量, 很容易造成命名冲突.


### 5.总结
context包通过构建树型关系的Context, 来达到上一层Goroutine能对传递给下一层Goroutine的控制.对于处理一个Request请求操作,
需要采用context来层层控制Goroutine,以及传递一些变量来共享.

context对象的生命周期一般仅为一个请求的处理周期.即针对一个请求创建一个Context变量(它为Context树结构的根);在请求处理结束后,
撤销此ctx变量,释放资源.

每创建一个Goroutine,要么将原有的Context传递给Goroutine,要么创建一个子Context并传递给Goroutine.

Context能灵活的存储不同类型. 不同数目的值,并且使多个Goroutine安全地读写其中的值.

当通过父Context对象创建子Context对象时, 可同时获得子Context的一个撤销函数,这样父Context对象的创建环境就获得了对子Context将要被传递到的Goroutine的撤销权.

在子Context被传递到的goroutine中,应该对该子Context的Done信道(channel)进行监控,一旦该信道被关闭(即上层运行环境撤销了本goroutine的执行),应主动终止对当前请求的处理,
释放资源并返回.


















